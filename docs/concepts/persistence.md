# Persistence & Storage

Danube reliable topics are backed by `danube-persistent-storage`.

From a user point of view, the storage system is built around three ideas:

- recent writes go to a **local per-topic WAL**
- historical data can be read from **durable exported segments**
- topic recovery and moves rely on **metadata** stored in the Raft metadata store

This page explains what that means operationally, how the broker `storage:` configuration works, and how to choose between `local`, `shared_fs`, and `object_store`.

If you want the implementation details, read the [Persistence Architecture](../architecture/persistence.md) page.

## What persistence means in Danube

For reliable topics, Danube does not write every message directly to remote storage.

Instead, the broker uses:

- a **hot path**
  - a local Write-Ahead Log (WAL) for fast writes and recent reads
- a **durable history path**
  - immutable exported segments for recovery, long-range replay, and topic moves
- a **metadata path**
  - segment descriptors and sealed topic state in the metadata store

This gives Danube two important properties:

- producers write with low latency because the active write path is local
- consumers can still read older history even after local WAL files were rotated away or the topic moved to another broker

## The three storage modes

Danube supports three broker storage modes.

## `local`

`local` keeps both the hot WAL and the durable segment backend on the broker’s local filesystem.

Use this when:

- you are running a single broker
- you want the simplest persistence setup
- you do not need a separate shared durable backend

Operational meaning:

- active writes go to local WAL files
- durable-history export is not continuously running in the background
- durable segment publication matters mainly for sealed handoff and durable-history replay

Good fit:

- development
- single-node production
- simple local persistent storage setups

## `shared_fs`

`shared_fs` uses:

- a **broker-local WAL/cache directory** for hot writes
- a **shared filesystem root** for durable segments

Use this when:

- brokers have access to a shared filesystem
- you want background export of historical data without using object storage

Operational meaning:

- the broker still writes locally first
- background export publishes immutable segments to the shared filesystem
- local staged WAL files can be pruned after durable history safely covers them

Good fit:

- on-prem clusters with shared storage
- Kubernetes or VM environments with shared POSIX-like volumes

## `object_store`

`object_store` uses:

- a **broker-local WAL/cache directory** for hot writes
- a **remote object store backend** for durable segments

Use this when:

- you want cloud-native storage
- brokers should not depend on shared disks
- you want durable history on S3, GCS, or Azure Blob via OpenDAL

Operational meaning:

- the broker stages active writes locally
- background export pushes sealed segment files into object storage
- local staged WAL files can be pruned after durable coverage is confirmed

Good fit:

- multi-broker cloud deployments
- clusters that need elastic, broker-independent historical storage

## How to choose a mode

| Need | Recommended mode |
|---|---|
| Simple single-broker persistence | `local` |
| Shared storage available across brokers | `shared_fs` |
| Cloud-native durable history | `object_store` |

A practical rule:

- choose **`local`** for simplicity
- choose **`shared_fs`** if you already operate shared storage reliably
- choose **`object_store`** if you want the most portable multi-broker durable storage model

## Broker configuration overview

Danube broker persistence is configured under the `storage:` section of `config/danube_broker.yml`.

At the top level, the broker accepts one of these shapes:

- `mode: local`
- `mode: shared_fs`
- `mode: object_store`

The actual YAML is a tagged structure, so the available fields depend on the selected mode.

## Common WAL settings

All storage modes support a `storage.wal` section.

```yaml
storage:
  wal:
    file_name: "wal.log"
    cache_capacity: 1024
    file_sync:
      interval_ms: 5000
      max_batch_bytes: 10485760
    rotation:
      max_bytes: 536870912
      max_hours: 24
    retention:
      time_minutes: 2880
      size_mb: 20480
      check_interval_minutes: 5
```

Here is what each field means.

## `storage.wal.file_name`

Default active WAL filename.

What it affects:

- the name of the active local WAL file inside the topic WAL directory

What it does **not** affect:

- rotated WAL files, which use generated names like `wal.<seq>.log`

## `storage.wal.cache_capacity`

The number of recent messages kept in the in-memory replay cache.

What it affects:

- how much recent history can be replayed from memory
- how often readers can satisfy recent reads without going back to files

Trade-off:

- larger value = better hot replay window, more memory
- smaller value = lower memory, more fallback to file/durable reads

## `storage.wal.file_sync.interval_ms`

How long the background writer can wait before flushing its buffered WAL writes.

What it affects:

- write-buffer flush cadence
- checkpoint freshness
- I/O pressure on the local WAL disk

Important note:

- despite the YAML name `file_sync`, this is effectively a **flush interval**, not a guarantee that every message performs its own synchronous fsync before producer progress continues

## `storage.wal.file_sync.max_batch_bytes`

The maximum number of buffered bytes before the writer flushes immediately.

What it affects:

- write latency under load
- memory used by the writer buffer
- how large a burst can accumulate before a flush happens

## `storage.wal.rotation.max_bytes`

Rotate the WAL after at least this many bytes have been written to the active file.

What it affects:

- local WAL file size
- granularity of exported durable segments
- retention/deletion behavior in export-later modes

## `storage.wal.rotation.max_hours`

Rotate the WAL after the file has been open this long.

What it affects:

- how long one active file is kept before rotation in low-write topics

Important note:

- this threshold is checked **when a new write arrives**
- it is not an always-running background idle-file rotation timer

## `storage.wal.retention.*`

Retention settings for **local staged WAL files**.

What they affect:

- pruning of older local WAL files after they are safely covered by durable history

What they do **not** currently do:

- they do not delete durable segment objects from shared storage or object storage

Important mode note:

- these settings matter in **`shared_fs`** and **`object_store`**
- in **`local`** mode, local staged WAL retention is not currently managed by the background deleter path

## `storage.wal.retention.time_minutes`

Age-based pruning threshold for eligible local staged WAL files.

## `storage.wal.retention.size_mb`

Size-based pruning threshold for eligible local staged WAL files.

## `storage.wal.retention.check_interval_minutes`

How often the local WAL retention task checks for files it can delete.

## `metadata_root`

Optional metadata prefix used under the Raft metadata store.

If omitted, the broker defaults it to:

```yaml
metadata_root: "/danube"
```

What it affects:

- where storage metadata such as segment descriptors and sealed-state markers are written

Use this when:

- you want to isolate multiple logical Danube environments inside the same metadata store namespace

## `local` mode configuration

Example:

```yaml
storage:
  mode: local
  root: "./danube-data/wal"
  metadata_root: "/danube"
  wal:
    file_name: "wal.log"
    cache_capacity: 1024
    file_sync:
      interval_ms: 5000
      max_batch_bytes: 10485760
    rotation:
      max_bytes: 536870912
```

What this means:

- each topic gets a local WAL under `./danube-data/wal`
- the same local filesystem root is also reused as the durable segment backend
- the broker does not continuously export segments in the background like export-later modes do

### `root` in `local` mode

In `local` mode, `storage.root` is the default WAL root.

If `storage.wal.dir` is also set, it overrides `root` for the actual WAL directory.

Example:

```yaml
storage:
  mode: local
  root: "./danube-data/wal"
  wal:
    dir: "./custom-wal-root"
```

Result:

- the broker uses `./custom-wal-root` for local topic WAL files

## `shared_fs` mode configuration

Example:

```yaml
storage:
  mode: shared_fs
  root: "/mnt/danube-shared-segments"
  cache_root: "/var/lib/danube/shared-fs-cache"
  metadata_root: "/danube"
  wal:
    file_name: "wal.log"
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
    retention:
      time_minutes: 1440
      size_mb: 10240
      check_interval_minutes: 5
```

What this means:

- local hot WAL files are written under `cache_root`
- durable exported segments are written under the shared filesystem `root`
- background export publishes sealed local WAL files into the shared durable store
- retention may prune old local staged WAL files after durable coverage is established

### `root` in `shared_fs` mode

In `shared_fs` mode, `storage.root` is **not** the broker-local WAL directory.

It is the durable shared segment root.

### `cache_root` in `shared_fs` mode

`cache_root` is the local broker directory used for hot WAL staging.

If omitted, the broker derives a default cache path next to `meta_store.data_dir` using the suffix:

```text
shared-fs-cache
```

That makes it easy to start with a minimal config while still keeping broker-local WAL staging separate from the shared durable root.

## `object_store` mode configuration

Example for S3-compatible storage:

```yaml
storage:
  mode: object_store
  cache_root: "/var/lib/danube/object-store-cache"
  metadata_root: "/danube"
  wal:
    file_name: "wal.log"
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
    retention:
      time_minutes: 1440
      size_mb: 10240
      check_interval_minutes: 5
  object_store:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "<access-key>"
    secret_key: "<secret-key>"
    anonymous: false
    virtual_host_style: false
```

What this means:

- local hot WAL files are written under `cache_root`
- durable exported segments are written to the configured S3 bucket/prefix
- background export and local retention both run
- durable historical reads come from the object store when the requested offset is older than the hot local retention window

### `cache_root` in `object_store` mode

`cache_root` is the broker-local WAL staging directory.

If omitted, the broker derives a default cache path next to `meta_store.data_dir` using the suffix:

```text
object-store-cache
```

### `object_store` block

In `object_store` mode, the durable backend is configured under:

```yaml
storage:
  object_store:
    backend: ...
```

Currently supported backends in broker configuration are:

- `s3`
- `gcs`
- `azblob`

## Object store examples

## S3

```yaml
storage:
  mode: object_store
  object_store:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "<access-key>"
    secret_key: "<secret-key>"
    profile: null
    role_arn: null
    session_token: null
    anonymous: false
    virtual_host_style: false
```

## GCS

```yaml
storage:
  mode: object_store
  object_store:
    backend: gcs
    root: "gcs://my-bucket/danube"
    project: "my-gcp-project"
    credentials_json: null
    credentials_path: "/path/to/credentials.json"
```

## Azure Blob

```yaml
storage:
  mode: object_store
  object_store:
    backend: azblob
    root: "my-container/danube"
    endpoint: "https://<account>.blob.core.windows.net"
    account_name: "<account-name>"
    account_key: "<account-key>"
```

## How reads behave with these modes

The mode changes where durable history comes from, but the reader model is consistent.

If the requested offset is:

- still within local WAL coverage
  - the broker reads from WAL only
- older than local WAL coverage
  - the broker reads from durable history first and then hands off to the hot WAL

That means consumers can still read old messages after:

- local WAL files were rotated and pruned
- a topic moved to another broker
- the new broker no longer has the old owner’s local files

## How topic moves relate to storage mode

All reliable topic moves use the same continuity rule:

- the old broker seals the topic and records the `last_committed_offset`
- the new broker resumes from `last_committed_offset + 1`

What changes by mode is where the durable historical prefix comes from:

- `local`
  - local durable segments on the broker filesystem
- `shared_fs`
  - shared durable filesystem segments
- `object_store`
  - remote durable object-store segments

## Common mistakes to avoid

- **Assuming `file_sync.interval_ms` means every message is synced immediately**
  - it controls background flush cadence, not per-message synchronous durability
- **Assuming `storage.root` means the same thing in every mode**
  - it does not; in `shared_fs` it is the durable shared root, not the local cache path
- **Assuming `retention` deletes durable history**
  - today it governs local staged WAL cleanup in export-later modes
- **Hardcoding credentials into version-controlled config**
  - prefer secret management or environment-based injection for production

## Recommended starting points

## Small single broker

```yaml
storage:
  mode: local
  root: "./danube-data/wal"
  wal:
    cache_capacity: 1024
    file_sync:
      interval_ms: 5000
      max_batch_bytes: 10485760
    rotation:
      max_bytes: 536870912
```

## Multi-broker with shared storage

```yaml
storage:
  mode: shared_fs
  root: "/mnt/danube-shared-segments"
  cache_root: "/var/lib/danube/shared-fs-cache"
  wal:
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
    retention:
      time_minutes: 1440
      size_mb: 10240
      check_interval_minutes: 5
```

## Cloud-native deployment

```yaml
storage:
  mode: object_store
  cache_root: "/var/lib/danube/object-store-cache"
  wal:
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
    retention:
      time_minutes: 1440
      size_mb: 10240
      check_interval_minutes: 5
  object_store:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "<access-key>"
    secret_key: "<secret-key>"
```

## Summary

For users and operators, the main thing to remember is:

- Danube always writes new reliable-topic messages to a **local WAL first**
- `shared_fs` and `object_store` add a **background durable-history export layer**
- old reads and topic moves rely on **durable segments plus metadata**, not just on whatever WAL files happen to be left on the current broker

If you are deciding between modes:

- choose `local` for simplicity
- choose `shared_fs` if you have reliable shared filesystem infrastructure
- choose `object_store` for the most cloud-native durable-history setup
