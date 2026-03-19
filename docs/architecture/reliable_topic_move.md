# Reliable Topic Moves

This document explains what happens inside Danube when a reliable topic moves from one broker to another.

The key requirement is simple:

- **the topic offset space must remain continuous across the move**

That means:

- producers must continue writing at the next correct offset
- consumers must keep reading from the same global history
- subscription cursors must remain valid even though topic ownership changed

## The pieces involved in a move

A reliable topic move touches four kinds of state.

## 1. Local WAL state

Each broker keeps a per-topic WAL with:

- a hot in-memory cache
- local WAL files
- a local checkpoint describing WAL topology

This state is fast, but it is tied to the broker currently owning the topic.

## 2. Durable segment history

Historical data is stored as immutable exported segments. These segments are written either to:

- the local filesystem in `local` mode
- a shared filesystem in `shared_fs` mode
- an object store in `object_store` mode

Each segment contains raw WAL frames and is described in the metadata store by a `SegmentDescriptor`.

## 3. Metadata state

The Raft metadata store holds the persistence metadata that survives broker moves:

- durable segment descriptors under `storage/topics/<topic>/segments/...`
- the current durable frontier pointer `segments/cur`
- the sealed mobility marker under `storage/topics/<topic>/state`

## 4. Subscription cursors

Consumer progress is stored independently from broker-local WAL state. That is what lets a consumer reconnect after a move and continue from the same logical offset.

## Why moves work

Danube does not try to move an in-memory topic object from one broker to another. Instead, it reconstructs the topic on the new broker from durable metadata and recovery rules.

The move works because Danube preserves two boundaries:

- **the last committed offset on the old owner**
- **the durable history frontier available to readers**

## Normal reliable topic operation before a move

Before a move, the topic behaves like any other reliable topic:

1. a producer appends a message
2. the WAL assigns the next offset
3. the message is inserted into cache and queued for the WAL writer
4. the writer flushes and rotates local files over time
5. in `shared_fs` and `object_store`, background export periodically publishes sealed WAL files as durable segments
6. readers consume from the WAL when the requested offset is still within local retention, or from durable history plus WAL when it is not

At this stage, the currently owning broker is authoritative for new writes.

## What the old broker does during unload

When the topic is unloaded from Broker A, the broker seals the topic.

Conceptually, the sequence is:

1. stop accepting new use of the topic storage instance
2. capture the topic’s `last_committed_offset`
3. shut down the WAL cleanly
4. stop any background segment exporter and deleter tasks for that topic
5. export any remaining local WAL history that still needs to become durable
6. clear the broker-local WAL directory
7. write a sealed mobility marker into metadata

The important output of this phase is:

```text
StorageStateSealed {
  sealed: true,
  last_committed_offset: <N>,
  broker_id: <old broker>,
  timestamp: <seal time>
}
```

This says: **the old owner accepted offsets through `N`; the next owner must resume at `N + 1`**.

## What gets exported before the handoff

The old owner does not publish arbitrary byte ranges. It exports safe, immutable WAL history.

During export:

1. the current WAL checkpoint is read
2. eligible WAL files are selected
3. each file is trimmed to the last safe frame boundary
4. the segment offset range is extracted
5. the safe prefix is uploaded as a durable segment
6. a `SegmentDescriptor` is written to metadata
7. the `segments/cur` pointer is updated

This is the durable history that later readers and new owners rely on.

## Metadata layout relevant to moves

The persistence metadata for a topic looks like this conceptually:

```text
/storage/topics/default/my-topic/segments/00000000000000000000
/storage/topics/default/my-topic/segments/00000000000000000512
/storage/topics/default/my-topic/segments/cur
/storage/topics/default/my-topic/state
```

Where:

- `segments/<padded_start_offset>` stores a `SegmentDescriptor`
- `segments/cur` stores the padded start offset of the latest durable segment
- `state` stores `StorageStateSealed`

Older documentation sometimes described `objects/...` metadata or a separate uploader checkpoint. The current implementation uses segment descriptors and a sealed mobility marker instead.

## How the new broker recovers the topic

When Broker B becomes the new owner, `StorageFactory::for_topic()` creates a new per-topic storage instance.

The critical step is recovery-start resolution.

`StorageFactory::resolve_recovery_start()` decides the initial WAL offset using this precedence:

1. **sealed mobility state**
   - if present, resume from `last_committed_offset + 1`
2. **durable segment catalog**
   - if there is no usable local WAL continuity but durable history exists, resume from the current durable segment’s `end_offset + 1`
3. **local WAL continuity**
   - if this broker still has a valid local WAL checkpoint and referenced files, continue from that local state
4. **empty topic**
   - otherwise start from offset `0`

In a real broker-to-broker move, the first case is the important one.

## Why the sealed state matters

Without the sealed state, a new broker could start a fresh WAL at the wrong offset.

With the sealed state:

- old owner ends at offset `N`
- new owner starts `next_offset` at `N + 1`

That preserves a single global offset sequence for the topic.

## What `WalStorage` does after recovery

When the topic was resumed from a sealed state, `StorageFactory` enables `with_hot_cutover()` on `WalStorage`.

This changes one important read decision:

- durable history is treated as authoritative for the historical prefix that existed before takeover
- the new owner’s hot WAL is treated as authoritative only for the newly rebuilt local tail

This prevents the new owner from pretending it still has complete local historical coverage when it only has newly created local state.

## Producer behavior after the move

After ownership changes, producers reconnect to the new broker and continue writing.

Suppose Broker A sealed at offset `21`.

Broker B will recover the topic with:

```text
next_offset = 22
```

The next messages will therefore receive offsets:

```text
22, 23, 24, ...
```

From the producer’s point of view, nothing special happened beyond reconnecting to the broker now hosting the topic.

## Consumer behavior after the move

Consumers do not resume from broker-local state. They resume from their subscription cursor.

Suppose a consumer previously acknowledged through offset `13`.

After reconnecting, it requests the next unread offset:

```text
14
```

Now `WalStorage::create_reader()` decides how to serve that request.

## Case 1: the request is still inside the hot WAL window

If the requested offset is within the new broker’s local hot window, the reader is created directly from the WAL.

## Case 2: the request is older than the hot WAL window

If the requested offset predates the new broker’s local hot window, `WalStorage` creates a hybrid reader:

1. `DurableHistoryReader` streams the historical prefix from durable segments
2. the stream is chained to the hot WAL at `hot_start_offset`

That means the consumer sees one logical stream such as:

```text
14..21   from durable history
22..N    from the new broker's hot WAL
```

No broker-specific cursor translation is needed because the offset space remained continuous.

## Mode-specific move behavior

The move protocol is the same in all storage modes, but the source of durable history differs.

## `local`

In `local` mode:

- there is no continuous background export loop
- durable segments are especially important during sealing
- the old broker exports the remaining durable history as part of the handoff

This mode is the least “always-exported” configuration, so sealing is the main moment when durable history becomes authoritative for takeover.

## `shared_fs`

In `shared_fs` mode:

- the broker stages active WAL locally
- background export continuously publishes sealed segments to a shared filesystem
- the seal step mainly ensures the final tail is exported and local staging is cleared

By the time a move happens, much of the topic history may already be present in the shared durable segment store.

## `object_store`

In `object_store` mode:

- the broker also stages active WAL locally
- background export continuously publishes segments to remote object storage
- sealing exports any remaining local tail and writes the mobility marker

This is the most cloud-native move path because the durable history is already broker-independent.

## Timeline example

Assume the following state before a move:

- Broker A has accepted offsets `0..21`
- a consumer cursor is at `13`
- durable history already covers `0..21`

Then the move looks like this:

1. Broker A seals the topic
   - records `last_committed_offset = 21`
   - exports remaining local history if needed
   - clears local WAL files
2. Broker B loads the topic
   - reads sealed mobility state
   - creates a WAL with `initial_offset = 22`
3. producers reconnect
   - new messages become `22, 23, 24, ...`
4. consumers reconnect
   - resume from cursor `14`
   - read `14..21` from durable history and `22..` from the new broker’s WAL if necessary

The continuity guarantee is preserved at both boundaries:

- producer write boundary: `21 -> 22`
- consumer read boundary: durable history -> hot WAL

## What this document validates about the current implementation

The current `danube-persistent-storage` implementation relies on:

- `StorageStateSealed` as the takeover marker
- `SegmentDescriptor` plus `segments/cur` as the durable-history catalog
- `StorageFactory::resolve_recovery_start()` to choose the next correct offset
- `WalStorage` hot-cutover logic to keep historical and hot reads aligned after takeover

The move protocol no longer depends on an “uploader checkpoint” model or `objects/...` metadata layout. The authoritative continuity mechanisms are the sealed mobility state and the durable segment catalog.

## Summary

Reliable topic moves in Danube are implemented as a controlled handoff between:

- the old broker’s final local WAL boundary
- the durable segment history stored outside the broker
- the new broker’s recovered WAL starting offset

As long as the old owner seals with the correct `last_committed_offset` and the new owner resumes from `last_committed_offset + 1`, producers and consumers keep seeing one continuous topic history even though ownership changed underneath them.
