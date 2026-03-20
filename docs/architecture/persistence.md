# Danube Persistent Storage Architecture

`danube-persistent-storage` is the storage engine behind reliable topics in Danube. The current design centers on a sealed-segment model with an active local WAL, durable immutable history, and explicit mobility metadata.

At a high level, the runtime is designed around a simple idea:

- keep the **write path local and fast** with a per-topic WAL
- keep **historical data durable and movable** through immutable exported segments
- keep **ownership and recovery decisions explicit** in the metadata store

This page explains the internal architecture of the crate, how the main components collaborate, and how behavior changes between `local`, `shared_fs`, and `object_store` modes.

For operator-facing configuration guidance, see the user-oriented persistence concepts page.

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

## `StorageFactory`

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

## `Wal`

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

## WAL writer

The writer task owns WAL file I/O.

Responsibilities:

- batch writes
- flush buffered bytes periodically or when the batch gets large
- rotate WAL files by size or elapsed time
- write `WalCheckpoint` snapshots

The writer is deliberately separated from `Wal::append()`, so producers do not pay synchronous file I/O latency on the hot path.

## `WalCheckpoint` and `CheckpointStore`

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

## `WalStorage`

`WalStorage` implements the broker-facing `PersistentStorage` trait.

It is the layer that decides how reads should be served:

- from the hot WAL only
- from durable history first, then hand off to the hot WAL

It also contains the special `history_cutover_from_hot` behavior used after sealed topic recovery. In that case, durable history is treated as authoritative for the historical prefix and the hot WAL is only used for the new owner’s local tail.

## WAL readers

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

## Durable history path

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

## Metadata abstractions

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

The export flow is:

1. read the current WAL checkpoint
2. select eligible rotated files, and optionally the active file during sealing
3. trim each file to the last safe frame boundary
4. extract the segment’s offset range
5. upload the safe byte prefix into the durable backend
6. write the `SegmentDescriptor` into metadata
7. advance the `segments/cur` pointer

Duplicate exports are avoided by consulting existing segment descriptors and skipping files whose `start_offset` is already published.

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

The crate supports three runtime modes.

## `local`

`local` keeps the hot WAL on the broker's local filesystem and does not require a separate durable backend.

Characteristics:

- no background export loop
- no separate shared or remote durable-history backend
- local retention can be configured, but it only removes files that are proven safely covered by durable metadata, so it is typically far less useful here than in export-later modes
- best fit for single-broker deployments, development, or simple durable local storage

Behavioral consequence:

- local WAL is the primary source of truth while the topic remains on the same broker
- sealed mobility state preserves offset continuity across restart or reload on that broker, but `local` mode does not provide the same cross-broker durable-history path as `shared_fs` or `object_store`

## `shared_fs`

`shared_fs` uses two filesystem locations:

- a **local cache/staging WAL root** on the current broker
- a **shared durable segment root** visible across brokers

Characteristics:

- background export is enabled
- local retention deleter is enabled
- the broker stages active writes locally and periodically exports sealed files into the shared durable location
- good fit for environments with a shared POSIX-style filesystem

Behavioral consequence:

- local WAL stays fast and broker-local
- durable history becomes cluster-visible through the shared filesystem

## `object_store`

`object_store` also uses local staging, but the durable backend is remote object storage accessed through OpenDAL.

Characteristics:

- background export is enabled
- local retention deleter is enabled
- durable segments are written to S3, GCS, Azure Blob, or another OpenDAL-supported backend
- best fit for cloud-native deployments where brokers should not depend on shared disks

Behavioral consequence:

- local WAL absorbs the active write workload
- exported immutable segments provide elastic durable history independent of broker disks

## Differences between the modes

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

The broker builds `StorageFactoryConfig` from the `storage:` section of `config/danube_broker.yml`.

Key integration facts:

- `local` uses `storage.local_wal_root` as the local WAL root
- `shared_fs` uses `storage.durable.root` as the shared durable segment root and `storage.local_wal_root` as the broker-local staging root when it is set
- if `shared_fs.local_wal_root` is omitted, the broker derives a broker-local cache path next to `meta_store.data_dir` using the suffix `shared-fs-cache`
- `object_store` uses `storage.durable` as the durable backend definition and `storage.local_wal_root` as the broker-local staging root when it is set
- if `object_store.local_wal_root` is omitted, the broker derives a broker-local cache path next to `meta_store.data_dir` using the suffix `object-store-cache`
- `storage.metadata_prefix` controls where persistence metadata is written in the Raft metadata store and defaults to `/danube`
- `storage.local_retention` configures pruning of eligible local staged WAL files in export-later modes
- older YAML aliases such as `root`, `cache_root`, `metadata_root`, `object_store`, and `wal.retention` are still accepted for compatibility, but the current documented shape uses `local_wal_root`, `metadata_prefix`, `durable`, and `local_retention`
- the broker currently uses the default segment export interval from the storage factory unless this is extended in configuration

## Summary

Danube persistence is not a separate “cloud storage layer” bolted onto the broker. It is a coordinated storage runtime made of:

- a fast per-topic WAL
- a durable immutable segment store
- metadata-backed recovery and mobility state
- reader logic that stitches hot and historical storage into one logical stream

`local`, `shared_fs`, and `object_store` all use the same core components. What changes between them is where durable segments live, whether background export runs continuously, and whether local staged WAL can be pruned after durable coverage is established.
