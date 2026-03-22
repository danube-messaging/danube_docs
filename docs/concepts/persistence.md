# Persistence & Storage

Danube reliable topics persist messages so consumers can replay history and topics can move between brokers without losing data.

The storage system is built around three ideas:

1. **Recent writes** go to a fast, local per-topic Write-Ahead Log (WAL)
2. **Historical data** is available from durable exported segments (in `shared_fs` and `object_store` modes)
3. **Recovery and topic moves** rely on metadata stored in the embedded Raft metadata store

For implementation details, see [Persistence Architecture](../architecture/persistence.md).

## Storage modes

Danube supports three storage modes, configured under `storage.mode` in the broker config.

### `local`

All data stays on the broker's local disk. No background export, no remote storage.

```yaml
storage:
  mode: local
  local_wal_root: "./danube-data/wal"
```

Best for: **single-node setups, development, and simple deployments**.

### `shared_fs`

Hot writes go to a local WAL; background export copies sealed segments to a shared filesystem visible to all brokers.

```yaml
storage:
  mode: shared_fs
  local_wal_root: "/var/lib/danube/shared-fs-cache"
  durable:
    root: "/mnt/danube-shared-segments"
```

Best for: **on-prem multi-broker clusters with NFS or shared POSIX volumes**.

### `object_store`

Hot writes go to a local WAL; background export pushes sealed segments to cloud object storage (S3, GCS, Azure Blob) via OpenDAL.

```yaml
storage:
  mode: object_store
  local_wal_root: "/var/lib/danube/object-store-cache"
  durable:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
```

Best for: **cloud-native multi-broker deployments**.

### How to choose

| Need | Mode |
|---|---|
| Simplest setup, single broker | `local` |
| Multi-broker with shared disk | `shared_fs` |
| Multi-broker, cloud-native | `object_store` |

## Configuration reference

Storage is configured under the `storage:` block in `config/danube_broker.yml`.

### Common fields (all modes)

| Field | Default | Description |
|---|---|---|
| `mode` | *(required)* | `local`, `shared_fs`, or `object_store` |
| `local_wal_root` | *(required for `local`; auto-derived for others)* | Directory for per-topic WAL files |
| `metadata_prefix` | `"/danube"` | Raft metadata store prefix for storage metadata |

### WAL settings

```yaml
storage:
  wal:
    cache_capacity: 1024       # in-memory replay cache (messages)
    file_sync:
      interval_ms: 5000        # background flush interval
      max_batch_bytes: 10485760 # flush when buffer reaches 10 MiB
    rotation:
      max_bytes: 536870912     # rotate WAL file at 512 MiB
      max_hours: 24            # rotate WAL file after 24 hours
```

**Important:** `file_sync` controls background flush cadence — it is not per-message synchronous durability. Rotation thresholds are checked on the next write after the limit is reached.

### Local retention

Controls pruning of local WAL files after durable segments safely cover them. Most useful in `shared_fs` and `object_store` modes.

```yaml
storage:
  local_retention:
    time_minutes: 2880          # prune files older than 48 hours
    size_mb: 20480              # prune when local WAL exceeds 20 GB
    check_interval_minutes: 5   # check every 5 minutes
```

This only deletes local WAL files — it does **not** delete durable segment objects.

### `shared_fs` durable backend

```yaml
storage:
  durable:
    root: "/mnt/danube-shared-segments"
```

`durable.root` is the shared segment directory — not the broker-local WAL directory.

If `local_wal_root` is omitted, the broker auto-derives a local cache path from `meta_store.data_dir`.

### `object_store` durable backend

Supported backends: **`s3`**, **`gcs`**, **`azblob`**.

**S3:**

```yaml
storage:
  durable:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "<access-key>"
    secret_key: "<secret-key>"
```

**GCS:**

```yaml
storage:
  durable:
    backend: gcs
    root: "gcs://my-bucket/danube"
    project: "my-gcp-project"
    credentials_path: "/path/to/credentials.json"
```

**Azure Blob:**

```yaml
storage:
  durable:
    backend: azblob
    root: "my-container/danube"
    endpoint: "https://<account>.blob.core.windows.net"
    account_name: "<account-name>"
    account_key: "<account-key>"
```

> **Security:** never hardcode cloud credentials in version-controlled config files. Use secret management or environment variables in production.

## How reads work

When a consumer reads from a topic, the broker decides where to serve the data from:

- **Offset is within local WAL** → read directly from WAL files and cache
- **Offset is older than local WAL** (in `shared_fs` or `object_store`) → read from durable segments first, then hand off to the WAL for recent data
- **Offset is older than local WAL** (in `local` mode) → depends on whether the local WAL files still exist

This means in `shared_fs` and `object_store`, consumers can read old messages even after local WAL files have been pruned or a topic has moved to another broker.

## Topic moves

When a topic moves between brokers, offset continuity is preserved:

1. The old broker **seals** the topic and records the last committed offset
2. The new broker **resumes** from `last_committed_offset + 1`

In `shared_fs` and `object_store`, historical reads on the new broker are served from durable segments. In `local` mode, only the offset sequence is preserved — there is no shared durable history backend.

## Legacy config aliases

Danube still accepts older field names for backward compatibility: `root`, `cache_root`, `metadata_root`, `object_store`, and `wal.retention`. New configs should use `local_wal_root`, `metadata_prefix`, `durable`, and `local_retention`.
