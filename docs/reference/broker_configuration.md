# Broker Configuration Reference

Complete reference for `danube_broker.yml` configuration options. This guide explains each parameter, its impact, and when to adjust it.

---

## Basic Configuration

### Cluster Identity

```yaml
cluster_name: "MY_CLUSTER"
```

**cluster_name**

- **Type**: String
- **Default**: None (required)
- **Impact**: Identifies your Danube cluster in metrics, logs, and multi-cluster setups
- **When to change**: Set a meaningful name for production environments (e.g., `production-us-east`, `staging-cluster`)

---

### Broker Services

```yaml
broker:
  host: "0.0.0.0"
  ports:
    client: 6650      # Producer/consumer connections
    admin: 50051      # Admin API (gRPC)
    raft: 7650        # Raft inter-node gRPC transport
    prometheus: 9040  # Metrics exporter
  # Optional: advertised addresses for proxy/k8s mode
  # advertised_listeners:
  #   broker_url: "broker-0.danube-broker-headless:6650"
  #   connect_url: "danube-proxy.example.com:6650"
```

**broker.host**

- **Type**: String (IP address or hostname)
- **Default**: `0.0.0.0` (all interfaces)
- **Impact**: Controls which network interfaces the broker binds to
- **When to change**:
  - Use `0.0.0.0` for containers or multi-interface servers
  - Use specific IP for security/isolation
  - Use `127.0.0.1` for local-only testing

**broker.ports.client**

- **Type**: Integer
- **Default**: `6650`
- **Impact**: Port where producers and consumers connect
- **When to change**: Resolve port conflicts or follow network policies

**broker.ports.admin**

- **Type**: Integer
- **Default**: `50051`
- **Impact**: gRPC port for admin operations (topic creation, subscriptions, etc.)
- **When to change**: Avoid conflicts with other gRPC services

**broker.ports.raft**

- **Type**: Integer
- **Default**: `7650`
- **Impact**: Raft inter-node gRPC transport port for consensus and metadata replication
- **When to change**: When running multiple brokers on the same host (use unique ports per broker)

**broker.ports.prometheus**

- **Type**: Integer (optional)
- **Default**: `9040`
- **Impact**: Prometheus metrics scraping endpoint
- **When to change**: Set to `null` to disable metrics exporter

### Advertised Listeners

**broker.advertised_listeners.broker_url**

- **Type**: String (optional)
- **Impact**: Address reachable inside the cluster (broker identity, inter-broker communication)
- **When to set**: Kubernetes (headless service DNS) or proxy deployments where the bind address differs from the routable address
- **If omitted**: Defaults to `host:ports.client`

**broker.advertised_listeners.connect_url**

- **Type**: String (optional)
- **Impact**: Address reachable by external clients via gRPC proxy
- **When to set**: When clients connect through a load balancer or ingress
- **If omitted**: Defaults to `broker_url`

---

### Metadata Store (Embedded Raft)

Danube uses embedded Raft consensus for metadata replication — no external dependency (no ETCD, no ZooKeeper). The `node_id` is auto-generated on first boot and persisted in `{data_dir}/node_id`.

```yaml
meta_store:
  data_dir: "./danube-data/raft"
  # seed_nodes:
  #   - "0.0.0.0:7650"
  #   - "0.0.0.0:7651"
  #   - "0.0.0.0:7652"
```

**meta_store.data_dir**

- **Type**: String (path)
- **Default**: `./danube-data/raft`
- **Impact**: Directory for Raft log, snapshots, and node identity
- **When to change**: Point to durable storage; ensure each broker has its own directory

**meta_store.seed_nodes**

- **Type**: Array of strings (optional)
- **Default**: Empty / omitted (single-node cluster, auto-init)
- **Impact**: List of Raft transport addresses (`host:raft_port`) for initial cluster formation
- **When to set**:
  - **Single-node**: Omit entirely — the broker auto-initializes a single-node Raft cluster
  - **Multi-node**: List ALL initial peers (including this broker's own `host:raft_port`)
- **Example** (3-node cluster):

```yaml
seed_nodes:
  - "broker1:7650"
  - "broker2:7651"
  - "broker3:7652"
```

---

### Bootstrap Configuration

```yaml
bootstrap_namespaces:
  - "default"

auto_create_topics: true

# admin_tls: false
```

**bootstrap_namespaces**

- **Type**: Array of strings
- **Default**: `["default"]`
- **Impact**: Namespaces created automatically on broker startup
- **When to change**: Pre-create namespaces for multi-tenant environments (e.g., `["default", "team-a", "team-b"]`)

**auto_create_topics**

- **Type**: Boolean
- **Default**: `true`
- **Impact**: Whether producers can create topics automatically when publishing
- **When to change**:
  - `true`: Development, rapid prototyping
  - `false`: Production with controlled topic creation via Admin API

**admin_tls**

- **Type**: Boolean (optional)
- **Default**: `false`
- **Impact**: Enable TLS on the admin gRPC API. When `true`, reuses the same `cert_file` / `key_file` from `auth.tls`
- **When to enable**: Remote cluster management over untrusted networks

---

## Security Configuration

```yaml
auth:
  mode: tls  # Options: none | tls | tlswithjwt
  tls:
    cert_file: "./cert/server-cert.pem"
    key_file: "./cert/server-key.pem"
    ca_file: "./cert/ca-cert.pem"
    verify_client: false
  jwt:
    secret_key: "your-secret-key"
    issuer: "danube-auth"
    expiration_time: 3600
```

### Authentication Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `none` | No authentication | Local development, trusted networks |
| `tls` | Mutual TLS (mTLS) | Secure production, service-to-service |
| `tlswithjwt` | TLS + JWT tokens | Multi-tenant, user-level auth |

**auth.mode**

- **Type**: String enum
- **Default**: `none`
- **Impact**: Security layer for client connections
- **When to change**: Always use `tls` or `tlswithjwt` in production

### TLS Settings

**auth.tls.cert_file**

- **Type**: File path
- **Impact**: Server certificate (public key)
- **When to change**: Use valid certificates from your PKI/CA

**auth.tls.key_file**

- **Type**: File path
- **Impact**: Server private key
- **When to change**: Keep secure, rotate periodically

**auth.tls.ca_file**

- **Type**: File path
- **Impact**: Certificate authority for validating client certs
- **When to change**: Match your organization's CA

**auth.tls.verify_client**

- **Type**: Boolean
- **Default**: `false`
- **Impact**: Require client certificates (mutual TLS)
- **When to change**:
  - `true`: Maximum security, requires client certs
  - `false`: Server-only TLS, simpler client setup

### JWT Settings

**auth.jwt.secret_key**

- **Type**: String
- **Impact**: Shared secret for signing/validating JWT tokens
- **When to change**: Use strong random key, rotate regularly, store in secrets manager

**auth.jwt.issuer**

- **Type**: String
- **Default**: `danube-auth`
- **Impact**: JWT issuer claim for validation
- **When to change**: Match your auth service's issuer

**auth.jwt.expiration_time**

- **Type**: Integer (seconds)
- **Default**: `3600` (1 hour)
- **Impact**: Token validity duration
- **When to change**:
  - Shorter: More secure, requires frequent renewal
  - Longer: Less overhead, wider security window

---

## Load Manager Configuration

The Load Manager handles topic assignment and automated rebalancing. See [Load Manager Architecture](../architecture/load_manager_architecture.md) for concepts.

```yaml
load_manager:
  assignment_strategy: "fair"
  load_report_interval_seconds: 30
  rebalancing:
    enabled: false
    aggressiveness: "balanced"
    check_interval_seconds: 300
    max_moves_per_hour: 10
    cooldown_seconds: 60
    min_brokers_for_rebalance: 2
    min_topic_age_seconds: 300
    blacklist_topics: []
```

### Topic Assignment Strategy

Controls how NEW topics are assigned to brokers.

| Strategy | Algorithm | CPU Overhead | Best For |
|----------|-----------|--------------|----------|
| `fair` | Topic count only | Lowest | Development, testing |
| `balanced` | Multi-factor (topic load + CPU + memory) | Medium | **Production (recommended)** |
| `weighted_load` | Adaptive bottleneck detection | Highest | Variable workloads |

**assignment_strategy**

- **Type**: String enum
- **Default**: `fair`
- **Impact**: How new topics are placed across brokers
- **When to change**:
  - `fair`: Simple setups, predictable placement
  - `balanced`: General production (recommended)
  - `weighted_load`: Clusters with highly variable workloads

**Formula for `balanced` strategy:**

```bash
score = (weighted_topic_load × 0.3) + (CPU% × 0.35) + (Memory% × 0.35)
```

### Load Report Interval

**load_report_interval_seconds**

- **Type**: Integer (seconds)
- **Default**: `30`
- **Impact**: How often brokers publish resource metrics to the metadata store
- **When to change**:
  - `5-10`: Testing, rapid response to changes (higher metadata traffic)
  - `30-60`: Production, balanced overhead
  - `>60`: Low-traffic clusters, reduce metadata store load

---

### Automated Rebalancing

Proactively moves topics between brokers to maintain cluster balance.

**rebalancing.enabled**

- **Type**: Boolean
- **Default**: `false` (disabled for safety)
- **Impact**: Enables automatic topic movement
- **When to enable**: After cluster is stable, you understand baseline behavior
- **Caution**: Start with `conservative` aggressiveness

### Aggressiveness Levels

Controls how aggressively the system optimizes cluster balance.

| Level | CV Threshold | Check Interval | Max Moves/Hour | Cooldown | Use Case |
|-------|--------------|----------------|----------------|----------|----------|
| `conservative` | >40% | 600s (10m) | 5 | 120s | Stable production, risk-averse |
| `balanced` | >30% | 300s (5m) | 10 | 60s | **General production** |
| `aggressive` | >20% | 180s (3m) | 20 | 30s | Dynamic workloads, testing |

**CV (Coefficient of Variation)** measures cluster imbalance:

- **<20%**: Excellent balance
- **20-30%**: Good balance
- **30-40%**: Moderate imbalance
- **>40%**: Significant imbalance

**rebalancing.aggressiveness**

- **Type**: String enum
- **Default**: `balanced`
- **Impact**: Sets default values for check interval, thresholds, and move limits
- **When to change**:
  - `conservative`: Production, minimize disruption
  - `balanced`: Most production clusters
  - `aggressive`: Testing, rapid optimization

### Rebalancing Knobs

**rebalancing.check_interval_seconds**

- **Type**: Integer
- **Default**: Varies by aggressiveness
- **Impact**: How often to evaluate cluster balance
- **When to change**: Override aggressiveness defaults for fine-tuning

**rebalancing.max_moves_per_hour**

- **Type**: Integer
- **Default**: Varies by aggressiveness
- **Impact**: Rate limit for topic moves (prevents storms)
- **When to change**: Increase for large rebalancing operations, decrease for stability

**rebalancing.cooldown_seconds**

- **Type**: Integer
- **Default**: Varies by aggressiveness
- **Impact**: Minimum time between consecutive topic moves
- **When to change**: Increase to slow down rebalancing, decrease for faster convergence

**rebalancing.min_brokers_for_rebalance**

- **Type**: Integer
- **Default**: `2`
- **Impact**: Rebalancing skipped if cluster has fewer brokers
- **When to change**: Rarely (single-broker clusters don't need rebalancing)

**rebalancing.min_topic_age_seconds**

- **Type**: Integer
- **Default**: `300` (5 minutes)
- **Impact**: Don't move topics younger than this
- **When to change**: Increase to protect recently created/moved topics

### Topic Blacklist

**rebalancing.blacklist_topics**

- **Type**: Array of strings (patterns)
- **Default**: `[]` (empty)
- **Impact**: Topics matching these patterns will NEVER be rebalanced
- **Patterns**:
  - Exact: `/admin/critical-topic`
  - Namespace wildcard: `/system/*`
- **When to use**: Protect critical topics, pin topics to specific brokers

**Example:**

```yaml
blacklist_topics:
  - "/system/*"              # All system topics
  - "/admin/critical-topic"  # Specific critical topic
  - "/production/high-priority/*"
```

---

## Storage Configuration

Reliable-topic persistence is configured under `storage:`. Danube uses a local WAL for new writes and can pair it with one of three durable-history modes. See [Persistence Architecture](../architecture/persistence.md) and [Persistence & Storage](../concepts/persistence.md) for behavioral details.

### Storage modes at a glance

| Mode | Durable history location | Typical use case |
|------|---------------------------|------------------|
| `local` | Local filesystem on the broker | Single broker, simplest setup |
| `shared_fs` | Shared filesystem | Multi-broker clusters with shared storage |
| `object_store` | Remote object store via OpenDAL | Cloud-native multi-broker clusters |

### Current top-level fields

The preferred storage configuration model uses these fields:

- `storage.mode`
- `storage.local_wal_root`
- `storage.metadata_prefix`
- `storage.local_retention`
- `storage.wal`
- `storage.durable` in `shared_fs` and `object_store`

For backward compatibility, the broker still accepts older aliases such as `root`, `cache_root`, `metadata_root`, `object_store`, and `storage.wal.retention`, but new configs should use the preferred names above.

### `storage` in `local` mode

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
    rotation:
      max_bytes: 536870912
      # max_hours: 24
```

**storage.local_wal_root**

- **Type**: String (path)
- **Impact**: Local root for topic WAL files in `local` mode
- **When to change**:
  - Point to durable local storage
  - Use fast SSD/NVMe for better write and replay performance

**storage.metadata_prefix**

- **Type**: String (optional)
- **Default**: `/danube`
- **Impact**: Prefix used for persistence metadata in the Raft metadata store
- **When to change**: Only when multiple logical Danube environments share the same metadata namespace

### `storage` in `shared_fs` mode

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
    rotation:
      max_bytes: 268435456
      max_hours: 6
```

**storage.local_wal_root**

- **Type**: String (path, optional)
- **Default**: Derived next to `meta_store.data_dir` with the suffix `shared-fs-cache`
- **Impact**: Broker-local WAL staging directory for hot writes and recent reads
- **When to change**: Set explicitly when you want the local cache on a specific disk

**storage.durable.root**

- **Type**: String (path)
- **Impact**: Durable shared segment root in `shared_fs` mode
- **When to change**: Point to storage accessible by every broker that may own reliable topics

### `storage` in `object_store` mode

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
    access_key: "${AWS_ACCESS_KEY_ID}"
    secret_key: "${AWS_SECRET_ACCESS_KEY}"
  local_retention:
    time_minutes: 1440
    size_mb: 10240
    check_interval_minutes: 5
  wal:
    rotation:
      max_bytes: 268435456
      max_hours: 6
```

**storage.local_wal_root**

- **Type**: String (path, optional)
- **Default**: Derived next to `meta_store.data_dir` with the suffix `object-store-cache`
- **Impact**: Broker-local WAL staging directory for hot writes and recent reads
- **When to change**: Set explicitly when you want the cache on a specific disk

**storage.durable**

- **Type**: Tagged object
- **Impact**: Durable backend definition for `object_store` mode
- **Supported backends**: `s3`, `gcs`, `azblob`

### WAL settings

The `storage.wal` block is supported by all storage modes. Most deployments only need `rotation`. Advanced tuning knobs live under `wal.advanced`.

```yaml
wal:
  rotation:
    max_bytes: 536870912     # 512 MB — rotate WAL file at this size
    # max_hours: 24           # optional time-based rotation
  # advanced:
  #   cache_capacity: 1024   # in-memory replay cache (number of messages)
  #   file_sync:
  #     interval_ms: 5000    # background flush interval (ms)
  #     max_batch_bytes: 10485760  # force flush at this buffer size
```

Legacy note: `cache_capacity`, `file_sync`, and `rotation` are also accepted directly under `wal:` (outside `advanced`), but new configs should prefer the structure above.

**storage.wal.dir**

- **Type**: String (path, optional)
- **Impact**: Explicit override for the effective local WAL directory
- **When to change**:
  - Use when the WAL should live somewhere other than the mode-derived local root
  - Keep it on fast local storage

If `storage.wal.dir` is set, it overrides the effective local WAL path derived from `storage.local_wal_root`.

**storage.wal.file_name**

- **Type**: String (optional)
- **Default**: `wal.log`
- **Impact**: Base filename for the active WAL file
- **When to change**: Rarely needed

### Advanced WAL tuning

These settings live under `storage.wal.advanced` (or directly under `storage.wal` for backward compatibility).

**storage.wal.advanced.cache_capacity**

- **Type**: Integer (number of messages)
- **Default**: `1024`
- **Impact**: In-memory replay cache for recent reads
- **When to change**:
  - Higher (`4096+`): Better hot replay window, more memory usage
  - Lower (`512`): Less memory, more fallback to files or durable history

### File flush behavior

**storage.wal.advanced.file_sync.interval_ms**

- **Type**: Integer (milliseconds)
- **Default**: `5000`
- **Impact**: Flush cadence for buffered WAL writes
- **When to change**:
  - Lower values: fresher on-disk WAL state, more disk pressure
  - Higher values: better throughput, less frequent flushes

This setting controls the writer's background flush interval — it is not per-message synchronous fsync.

**storage.wal.advanced.file_sync.max_batch_bytes**

- **Type**: Integer (bytes)
- **Default**: `10485760` (10 MiB)
- **Impact**: Forces an earlier flush when the write buffer reaches this size
- **When to change**:
  - Increase for high-volume workloads that benefit from larger batches
  - Decrease for lower-latency flush behavior

### WAL rotation

**storage.wal.rotation.max_bytes**

- **Type**: Integer (bytes)
- **Default**: Disabled when omitted
- **Impact**: Rotates the active WAL file after it reaches this size
- **When to change**:
  - Larger values: fewer, larger WAL files and durable segments
  - Smaller values: more frequent rotation and smaller segments

**storage.wal.rotation.max_hours**

- **Type**: Integer (hours, optional)
- **Default**: Disabled
- **Impact**: Rotates the active WAL file once it has been open this long
- **When to enable**: Low-traffic topics where size-based rotation alone would keep files open too long

This threshold is checked when new writes arrive; it is not an always-running idle rotation timer.

`rotation` is also accepted under `storage.wal.advanced.rotation` for backward compatibility.

### Local staged WAL retention

`storage.local_retention` controls pruning of eligible local staged WAL files once durable history safely covers them.

```yaml
local_retention:
  time_minutes: 2880
  size_mb: 20480
  check_interval_minutes: 5
```

**storage.local_retention.time_minutes**

- **Type**: Integer (minutes, optional)
- **Impact**: Age threshold for pruning eligible local staged WAL files
- **When to change**:
  - Increase to keep more local recovery buffer
  - Decrease when local disk space is limited

**storage.local_retention.size_mb**

- **Type**: Integer (megabytes, optional)
- **Impact**: Size threshold for pruning eligible local staged WAL files
- **When to change**:
  - Increase for high-volume reliable topics
  - Decrease for tighter local disk budgets

**storage.local_retention.check_interval_minutes**

- **Type**: Integer
- **Default**: `5`
- **Impact**: How often the local WAL retention pass runs
- **When to change**: Rarely needed

Retention applies to local staged WAL files once durable history safely covers them. It does not currently delete durable segment objects from shared storage or object storage.

This retention behavior is most relevant in `shared_fs` and `object_store` mode, where durable coverage is normally published independently of the local WAL files. In `local` mode it can still be configured, but it is usually less useful because there is no separate shared or remote durable-history backend.

For compatibility, the broker also accepts the older alias `storage.wal.retention`.

### Object store backends

The `storage.durable` block is used in `object_store` mode.

**storage.durable.backend**

- **Type**: String enum
- **Options**: `s3` | `gcs` | `azblob`
- **Impact**: Selects the durable object-store backend

**storage.durable.root**

- **Type**: String
- **Impact**: Backend-specific storage root or prefix
- **Formats**:
  - `s3`: `s3://bucket-name/prefix`
  - `gcs`: `gcs://bucket-name/prefix`
  - `azblob`: `container-name/prefix`

### S3

```yaml
storage:
  mode: object_store
  durable:
    backend: s3
    root: "s3://my-bucket/danube"
    region: "us-east-1"
    endpoint: "https://s3.us-east-1.amazonaws.com"
    access_key: "${AWS_ACCESS_KEY_ID}"
    secret_key: "${AWS_SECRET_ACCESS_KEY}"
    profile: null
    role_arn: null
    session_token: null
    anonymous: false
    virtual_host_style: false
```

**storage.durable.region**

- **Type**: String (optional)
- **Impact**: S3 region selection

**storage.durable.endpoint**

- **Type**: String (optional)
- **Impact**: Custom S3-compatible endpoint such as MinIO

**storage.durable.access_key** / **storage.durable.secret_key**

- **Type**: String (optional)
- **Impact**: Static credentials for the S3 backend
- **When to set**: Use only when you are not relying on instance roles, profiles, or external secret injection

**storage.durable.profile**

- **Type**: String (optional)
- **Impact**: Named credentials profile

**storage.durable.role_arn**

- **Type**: String (optional)
- **Impact**: IAM role assumption target

**storage.durable.session_token**

- **Type**: String (optional)
- **Impact**: Temporary session credentials

**storage.durable.anonymous**

- **Type**: Boolean (optional)
- **Impact**: Anonymous access mode

**storage.durable.virtual_host_style**

- **Type**: Boolean (optional)
- **Impact**: Enables virtual-host-style bucket addressing

### GCS

```yaml
storage:
  mode: object_store
  durable:
    backend: gcs
    root: "gcs://my-bucket/danube"
    project: "my-gcp-project"
    credentials_json: null
    credentials_path: "/path/to/service-account.json"
```

**storage.durable.project**

- **Type**: String (optional)
- **Impact**: GCP project identifier

**storage.durable.credentials_json**

- **Type**: String (optional)
- **Impact**: Inline service account JSON

**storage.durable.credentials_path**

- **Type**: String (optional)
- **Impact**: Path to a service account credentials file

### Azure Blob

```yaml
storage:
  mode: object_store
  durable:
    backend: azblob
    root: "my-container/danube"
    endpoint: "https://myaccount.blob.core.windows.net"
    account_name: "myaccount"
    account_key: "${AZURE_STORAGE_KEY}"
```

**storage.durable.endpoint**

- **Type**: String (optional)
- **Impact**: Azure Blob service endpoint

**storage.durable.account_name**

- **Type**: String (optional)
- **Impact**: Azure storage account name

**storage.durable.account_key**

- **Type**: String (optional)
- **Impact**: Azure storage account key

For production, avoid hardcoding credentials directly in version-controlled YAML. Prefer environment-variable substitution, secret mounts, or provider-native identity mechanisms.

---

## Broker Policies

Default resource limits for topics. Can be overridden at namespace or topic level.

```yaml
policies:
  max_producers_per_topic: 0
  max_subscriptions_per_topic: 0
  max_consumers_per_topic: 0
  max_consumers_per_subscription: 0
  max_publish_rate: 0
  max_subscription_dispatch_rate: 0
  max_message_size: 10485760  # 10 MB
```

**Default value `0` means unlimited** for all rate/count limits.

### Connection Limits

**max_producers_per_topic**

- **Type**: Integer
- **Default**: `0` (unlimited)
- **Impact**: Maximum concurrent producers per topic
- **When to set**: Prevent resource exhaustion from producer storms

**max_subscriptions_per_topic**

- **Type**: Integer
- **Default**: `0` (unlimited)
- **Impact**: Maximum subscriptions per topic
- **When to set**: Control fan-out complexity

**max_consumers_per_topic**

- **Type**: Integer
- **Default**: `0` (unlimited)
- **Impact**: Maximum concurrent consumers across all subscriptions
- **When to set**: Limit total consumer connections

**max_consumers_per_subscription**

- **Type**: Integer
- **Default**: `0` (unlimited)
- **Impact**: Maximum consumers sharing a single subscription
- **When to set**: Control load-sharing fan-out

### Rate Limits

**max_publish_rate**

- **Type**: Integer (messages/second or bytes/second)
- **Default**: `0` (unlimited)
- **Impact**: Throttle producer publish rate
- **When to set**: Prevent producer overwhelming broker

**max_subscription_dispatch_rate**

- **Type**: Integer (messages/second)
- **Default**: `0` (unlimited)
- **Impact**: Throttle consumer dispatch rate
- **When to set**: Protect slow consumers

### Message Size

**max_message_size**

- **Type**: Integer (bytes)
- **Default**: `10485760` (10 MB)
- **Impact**: Maximum single message size
- **When to change**:
  - Increase: Large payloads (video, logs)
  - Decrease: Prevent memory issues from huge messages
  - **Note**: Very large messages impact performance
