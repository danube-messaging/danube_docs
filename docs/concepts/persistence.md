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
- in `shared_fs` and `object_store`, consumers can still read older history even after local WAL files were rotated away or the topic moved to another broker

## The three storage modes

Danube supports three broker storage modes.

## `local`

`local` keeps the hot WAL on the broker’s local filesystem and does not use a separate shared or remote durable backend.

Use this when:

- you are running a single broker
- you want the simplest persistence setup
- you do not need durable history to be shared across brokers

Operational meaning:

- active writes go to local WAL files
- durable-history export is not continuously running in the background
- continuity relies mainly on broker-local WAL state and sealed mobility state on that broker

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

## Current broker configuration model

Danube broker persistence is configured under the `storage:` section of `config/danube_broker.yml`.

The current documented field names are:

- `storage.mode`
- `storage.local_wal_root`
- `storage.metadata_prefix`
- `storage.local_retention`
- `storage.wal`
- `storage.durable` for `shared_fs` and `object_store`

Danube still accepts older aliases for compatibility, including `root`, `cache_root`, `metadata_root`, `object_store`, and `storage.wal.retention`, but the preferred shape for new configs is the one above.

## Common WAL settings

All storage modes support a `storage.wal` block.

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
```

The most important WAL settings are:

| Field | Default | What it controls |
|---|---|---|
| `file_name` | `wal.log` | The active WAL filename inside each topic’s local WAL directory |
| `cache_capacity` | `1024` messages | How much recent history stays in the in-memory replay cache |
| `file_sync.interval_ms` | `5000` ms | How long the background writer can buffer WAL writes before flushing |
| `file_sync.max_batch_bytes` | `10485760` bytes | How much buffered data can accumulate before the writer flushes early |
| `rotation.max_bytes` | disabled when omitted | The size threshold for rotating the active WAL file |
| `rotation.max_hours` | disabled when omitted | The age threshold for rotating the active WAL file |

Notes for operators:

- `file_sync` is a **flush policy**, not a promise that every message performs its own synchronous fsync before producer progress continues.
- `rotation.max_hours` is checked on the **next write** after the threshold is reached; it is not an always-running idle-file timer.
- The preferred config shape is the flat one shown above. Older configs may still use `storage.wal.advanced.cache_capacity`, `storage.wal.advanced.file_sync`, and `storage.wal.advanced.rotation`; the broker merges both forms.
- `storage.wal.dir` is also supported as an explicit override for the effective local WAL directory, but most users should prefer the mode-level `local_wal_root` and only use `wal.dir` when they need a special override.

## Local staged WAL retention

`storage.local_retention` controls pruning of **local staged WAL files** once durable history safely covers them.

```yaml
storage:
  local_retention:
    time_minutes: 2880
    size_mb: 20480
    check_interval_minutes: 5
```

The retention fields mean:

| Field | Meaning |
|---|---|
| `time_minutes` | Age-based pruning threshold for eligible local staged WAL files |
| `size_mb` | Size-based pruning threshold for eligible local staged WAL files |
| `check_interval_minutes` | How often the retention task checks for files it can delete |

Important scope note:

- this retention applies to local staged WAL files, not to durable segment objects in shared storage or object storage
- it matters most in `shared_fs` and `object_store`, where durable coverage is normally published independently of the local WAL files
- in `local` mode it can still be configured, but it is usually less useful because there is no separate shared or remote durable-history backend

## Metadata prefix

`storage.metadata_prefix` controls where persistence metadata is written in the Raft metadata store.

If you omit it, the broker uses:

```yaml
storage:
  metadata_prefix: "/danube"
```

Use it when you need to isolate multiple logical Danube environments inside the same metadata-store namespace.

## `local` mode configuration

Example:

```yaml
storage:
  mode: local
  local_wal_root: "./danube-data/wal"
  metadata_prefix: "/danube"
  local_retention:
    time_minutes: 2880
    size_mb: 20480
    check_interval_minutes: 5
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
- there is no separate shared or remote durable backend
- the broker does not continuously export segments in the background like export-later modes do

### `local_wal_root` in `local` mode

In `local` mode, `storage.local_wal_root` is the normal WAL root.

If `storage.wal.dir` is also set, it overrides `local_wal_root` for the actual WAL directory.

Example:

```yaml
storage:
  mode: local
  local_wal_root: "./danube-data/wal"
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
  local_wal_root: "/var/lib/danube/shared-fs-cache"
  metadata_prefix: "/danube"
  durable:
    root: "/mnt/danube-shared-segments"
  local_retention:
    time_minutes: 1440
    size_mb: 10240
    check_interval_minutes: 5
  wal:
    file_name: "wal.log"
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
```

What this means:

- local hot WAL files are written under `local_wal_root`
- durable exported segments are written under `durable.root`
- background export publishes sealed local WAL files into the shared durable store
- `local_retention` may prune old local staged WAL files after durable coverage is established

### `durable.root` in `shared_fs` mode

In `shared_fs` mode, `storage.durable.root` is **not** the broker-local WAL directory.

It is the durable shared segment root.

### `local_wal_root` in `shared_fs` mode

`local_wal_root` is the local broker directory used for hot WAL staging.

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
  local_wal_root: "/var/lib/danube/object-store-cache"
  metadata_prefix: "/danube"
  durable:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "<access-key>"
    secret_key: "<secret-key>"
    anonymous: false
    virtual_host_style: false
  local_retention:
    time_minutes: 1440
    size_mb: 10240
    check_interval_minutes: 5
  wal:
    file_name: "wal.log"
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
```

What this means:

- local hot WAL files are written under `local_wal_root`
- durable exported segments are written to the configured S3 bucket/prefix
- background export and local retention both run
- durable historical reads come from the object store when the requested offset is older than the hot local retention window

### `local_wal_root` in `object_store` mode

`local_wal_root` is the broker-local WAL staging directory.

If omitted, the broker derives a default cache path next to `meta_store.data_dir` using the suffix:

```text
object-store-cache
```

### `durable` block in `object_store` mode

In `object_store` mode, the durable backend is configured under:

```yaml
storage:
  durable:
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
  durable:
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
  durable:
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
  durable:
    backend: azblob
    root: "my-container/danube"
    endpoint: "https://<account>.blob.core.windows.net"
    account_name: "<account-name>"
    account_key: "<account-key>"
```

## How reads behave with these modes

The reader model is consistent, but the available history sources differ by mode.

If the requested offset is:

- still within local WAL coverage
  - the broker reads from WAL only
- older than local WAL coverage in `shared_fs` or `object_store`
  - the broker reads from durable history first and then hands off to the hot WAL
- older than local WAL coverage in `local`
  - continuity depends on the broker still having the local WAL files needed for replay

That means consumers can still read old messages after:

- local WAL files were rotated and pruned in `shared_fs` or `object_store`
- a topic moved to another broker in `shared_fs` or `object_store`
- the new broker no longer has the old owner’s local files in `shared_fs` or `object_store`

## How topic moves relate to storage mode

Reliable topic mobility always preserves the same offset rule:

- the old broker seals the topic and records the `last_committed_offset`
- the new broker resumes from `last_committed_offset + 1`

What changes by mode is where the durable historical prefix comes from:

- `local`
  - continuity is mainly broker-local; it does not provide the same shared durable-history path used for cross-broker replay
- `shared_fs`
  - shared durable filesystem segments
- `object_store`
  - remote durable object-store segments

## Common mistakes to avoid

- **Assuming `file_sync.interval_ms` means every message is synced immediately**
  - it controls background flush cadence, not per-message synchronous durability
- **Assuming `local_wal_root` and `durable.root` are interchangeable**
  - they are not; `local_wal_root` is broker-local staging, while `durable.root` is shared or remote durable history
- **Assuming `local_retention` deletes durable history**
  - today it governs local staged WAL cleanup only; it does not delete durable segment objects
- **Hardcoding credentials into version-controlled config**
  - prefer secret management or environment-based injection for production

## Recommended starting points

## Small single broker

```yaml
storage:
  mode: local
  local_wal_root: "./danube-data/wal"
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
  local_wal_root: "/var/lib/danube/shared-fs-cache"
  durable:
    root: "/mnt/danube-shared-segments"
  local_retention:
    time_minutes: 1440
    size_mb: 10240
    check_interval_minutes: 5
  wal:
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
```

## Cloud-native deployment

```yaml
storage:
  mode: object_store
  local_wal_root: "/var/lib/danube/object-store-cache"
  durable:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "<access-key>"
    secret_key: "<secret-key>"
  local_retention:
    time_minutes: 1440
    size_mb: 10240
    check_interval_minutes: 5
  wal:
    cache_capacity: 4096
    file_sync:
      interval_ms: 2000
      max_batch_bytes: 8388608
    rotation:
      max_bytes: 268435456
      max_hours: 6
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
