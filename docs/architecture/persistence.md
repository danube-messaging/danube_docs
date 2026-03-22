# Danube Persistent Storage Architecture

`danube-persistent-storage` is the storage engine behind reliable topics in Danube. The current design centers on a sealed-segment model with an active local WAL, durable immutable history, and explicit mobility metadata.

At a high level, the runtime is designed around a simple idea:

- keep the **write path local and fast** with a per-topic WAL
- keep **historical data durable and movable** through immutable exported segments
- keep **ownership and recovery decisions explicit** in the metadata store

This page explains the internal architecture of the crate, how the main components collaborate, and how behavior changes between `local`, `shared_fs`, and `object_store` modes.

For operator-facing configuration guidance, see [Persistence & Storage](../concepts/persistence.md).

## Design goals

- **Low-latency hot path**
  - Appends should not wait on remote storage.
- **Ordered replay with continuity**
  - Readers must see a continuous stream across WAL files, cache, durable segments, and broker moves.
- **Topic mobility**
  - A topic can move to another broker without resetting offsets.
- **Mode flexibility**
  - The same storage engine can back local-only deployments, shared-filesystem deployments, or cloud object-store deployments.

## High-level model

At runtime, each reliable topic is backed by three layers of state:

1. **Hot local state**
   - a per-topic WAL and in-memory cache used for appends and recent reads
2. **Durable historical state**
   - immutable exported segments stored either on local filesystem, shared filesystem, or object store
3. **Metadata state**
   - segment descriptors, current durable frontier, and mobility markers stored in the Raft metadata store

The system is intentionally split this way:

- the WAL optimizes for active ownership and recent replay
- the durable segment store optimizes for recovery, history, and ownership transfer
- the metadata store provides the authoritative map between offsets and durable objects

## Core components

### `StorageFactory`

`StorageFactory` is the orchestration layer. It is the entry point used by the broker to obtain per-topic storage.

Responsibilities:

- normalize topic names into internal storage paths
- create or recover the topic WAL
- wire `WalStorage` with durable history when configured
- start per-topic background tasks when the mode requires them
- coordinate sealing, topic cleanup, and recovery after ownership transfer

Important internal behavior:

- `for_topic()` creates the runtime view of a topic
- `get_or_create_wal()` determines the correct starting offset for the local WAL
- `seal()` exports remaining local history, clears local state, and writes a mobility marker

### `Wal`

`Wal` is the hot-path append log for a topic.

Responsibilities:

- assign monotonically increasing topic offsets
- serialize messages into framed WAL records
- keep a bounded in-memory replay cache
- enqueue disk I/O to the background writer
- broadcast live appends to tailing readers

Each frame is stored as:

```text
[u64 offset][u32 len][u32 crc][payload bytes]
```

The CRC is used by file and durable-history readers to detect corruption.

### WAL writer

The writer task owns WAL file I/O.

Responsibilities:

- batch writes
- flush buffered bytes periodically or when the batch gets large
- rotate WAL files by size or elapsed time
- write `WalCheckpoint` snapshots

The writer is deliberately separated from `Wal::append()`, so producers do not pay synchronous file I/O latency on the hot path.

### `WalCheckpoint` and `CheckpointStore`

`WalCheckpoint` is the local summary of WAL topology.

It tracks:

- the oldest locally retained offset (`start_offset`)
- the latest written offset
- the active file
- rotated files and their first offsets

This checkpoint is used by:

- file replay
- segment export
- local retention deletion
- recovery on restart

`CheckpointStore` keeps the latest checkpoint in memory and atomically persists it to disk using temp-file plus rename.

### `WalStorage`

`WalStorage` implements the broker-facing `PersistentStorage` trait.

It is the layer that decides how reads should be served:

- from the hot WAL only
- from durable history first, then hand off to the hot WAL

It also contains the special `history_cutover_from_hot` behavior used after sealed topic recovery. In that case, durable history is treated as authoritative for the historical prefix and the hot WAL is only used for the new owner’s local tail.

### WAL readers

There are two important reader paths inside the WAL:

- **`StatefulReader`**
  - drives `Files -> Cache -> Live` replay for local WAL history
- **streaming WAL file reader**
  - parses WAL files incrementally and stops at safe frame boundaries

The key invariant is continuity:

- readers always track the next expected offset
- duplicates are ignored
- gaps are repaired from cache when possible
- unrecoverable gaps are surfaced as errors

### Durable history path

Historical durable reads are handled by:

- `DurableHistoryReader`
- `DurableStore`
- `OpendalDurableStore`

The reader:

- loads segment descriptors from metadata
- selects overlapping durable segments for the requested offset range
- uses sparse offset indexes to seek near the requested start offset
- parses WAL frames sequentially from durable objects
- hands control back to the hot WAL at the hot/durable boundary

This means durable history uses the same frame format as the local WAL. Exported segments are not a different logical message format; they are immutable slices of WAL history.

### Metadata abstractions

Persistent-storage metadata lives in the broker’s Raft metadata store and is wrapped by these components:

- **`StorageMetadata`**
  - low-level key/value wrapper
- **`SegmentCatalog`**
  - high-level access to durable segment descriptors
- **`MobilityState`**
  - sealed-state marker used during topic movement

Important metadata objects:

- **`SegmentDescriptor`**
  - identifies one durable segment and its offset range
- **`segments/cur` pointer**
  - points to the latest durable segment by padded start offset key
- **`StorageStateSealed`**
  - records the last committed local offset and marks that a broker sealed the topic for takeover

## Durable segment export

Export turns sealed local WAL byte ranges into immutable durable objects.

The export flow:

1. Read the current WAL checkpoint
2. Select eligible rotated files (and optionally the active file during sealing)
3. Trim each file to the last safe frame boundary
4. Extract the segment's offset range from intact frames
5. Upload the safe byte prefix into the durable backend
6. Write the `SegmentDescriptor` into metadata
7. Advance the `segments/cur` pointer

Duplicate exports are avoided by consulting existing segment descriptors and skipping files whose `start_offset` is already published. A sparse offset index is built during export to accelerate historical reads.

## Local retention deleter

In export-later modes, the local WAL is only a staging area. Once durable history safely covers older WAL files, the deleter can prune them.

The deleter:

- loads the local WAL checkpoint
- checks which rotated WAL files are fully covered by durable history
- applies time and size retention policies
- deletes eligible files
- rewrites the checkpoint so local replay starts at the new retained boundary

The active WAL file is never deleted by the retention pass.

## Write path

The normal append path looks like this:

1. broker writes a message to the topic storage
2. `Wal::append()` first reserves capacity on the background writer channel
3. the WAL atomically assigns the next topic offset
4. a `Write` command for that offset is enqueued to the background writer
5. only after enqueue is guaranteed, the message is published into the in-memory cache and live reader stream

Key property:

- the offset is assigned before durable export and before reader delivery
- reader-visible publication does not happen if the append cannot be enqueued to the writer
- this offset becomes the stable identity of the message for the rest of its lifetime

## Read path

When a consumer asks to read from an offset, `WalStorage` decides whether the request can be satisfied locally.

### WAL-only path

If the requested offset is within the hot local retention window, the reader is created directly from the WAL:

- local files if needed
- cache
- live broadcast stream

### Durable-history-plus-hot path

If the requested offset is older than the hot retention boundary:

1. `DurableHistoryReader` streams the historical prefix from durable segments
2. `WalStorage` chains that stream to `Wal::tail_reader()` starting at `hot_start_offset`

This gives the caller a single ordered stream across historical and live state.

## Recovery and topic mobility

Topic recovery is centered on one question: **what should the next local offset be on this broker?**

`StorageFactory::resolve_recovery_start()` answers that in this order:

1. **sealed mobility state**
   - if the topic was explicitly sealed on another broker, resume from `last_committed_offset + 1`
2. **durable segment catalog**
   - if there is no usable local WAL continuity in `shared_fs` or `object_store`, resume from the current durable segment’s `end_offset + 1`
3. **local WAL continuity**
   - if the local WAL checkpoint still references real files, let local WAL recovery continue from there
4. **empty topic**
   - otherwise start from offset `0`

The important distinction is between:

- **local WAL continuity**
  - useful when the same broker still has its staged files
- **sealed continuity**
  - useful when ownership moved and the new broker must continue the offset sequence without those files

## Storage modes

All three modes use the same core components. What changes is where durable segments live and which background tasks run.

- **`local`** — WAL-only on the broker's local disk. No background export, no separate durable backend. The local WAL is the sole source of truth.
- **`shared_fs`** — local WAL for hot writes, background export to a shared filesystem root visible to all brokers.
- **`object_store`** — local WAL for hot writes, background export to remote object storage (S3, GCS, Azure Blob) via OpenDAL.

| Aspect | `local` | `shared_fs` | `object_store` |
|---|---|---|---|
| Hot write path | Local WAL | Local WAL | Local WAL |
| Durable backend | No separate durable backend | Shared filesystem root | Remote object store |
| Background segment export | No | Yes | Yes |
| Local WAL retention | Optional, but most useful only when durable coverage exists | Yes | Yes |
| Requires broker-local staging directory | No extra staging beyond local WAL root | Yes | Yes |
| Best fit | Single broker / simple local durability | Multi-broker with shared volume | Cloud-native multi-broker deployments |

## Important invariants

- **Offsets are monotonic per topic**
  - once assigned, they are never rewritten
- **`current_offset()` means next offset to assign**
  - not the last written offset
- **`last_committed_offset()` means highest local WAL offset already accepted**
- **durable history and hot WAL must meet at a clean boundary**
  - readers should not see gaps or overlaps at the handoff
- **sealed state is authoritative for takeover**
  - when present, it determines where the next owner resumes

## Broker integration

The broker builds `StorageFactoryConfig` from the `storage:` section of `config/danube_broker.yml` (see `build_storage_factory_config()` in `main.rs`). `StorageFactory` receives the config together with an `Arc<dyn MetadataStore>` and manages all per-topic storage from there.

Key integration points:

- `StorageFactory::for_topic()` is called by `TopicManager::ensure_local()` when a topic is assigned to this broker
- `StorageFactory::seal()` is called during topic unload to persist mobility state and export remaining WAL data
- `StorageFactory::delete_storage_metadata()` is used for full topic cleanup
- Background segment export runs at a default interval of 300 seconds (configurable)
- If `local_wal_root` is omitted in `shared_fs` or `object_store` modes, the broker auto-derives a local cache path next to `meta_store.data_dir`

For YAML configuration details, see [Persistence & Storage](../concepts/persistence.md).

## Summary

Danube persistence is not a separate “cloud storage layer” bolted onto the broker. It is a coordinated storage runtime made of:

- a fast per-topic WAL
- a durable immutable segment store
- metadata-backed recovery and mobility state
- reader logic that stitches hot and historical storage into one logical stream

`local`, `shared_fs`, and `object_store` all use the same core components. What changes between them is where durable segments live, whether background export runs continuously, and whether local staged WAL can be pruned after durable coverage is established.
