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
    prometheus: 9040  # Metrics exporter
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

**broker.ports.prometheus**

- **Type**: Integer (optional)
- **Default**: `9040`
- **Impact**: Prometheus metrics scraping endpoint
- **When to change**: Set to `null` to disable metrics exporter

---

### Metadata Store (ETCD)

```yaml
meta_store:
  host: "127.0.0.1"
  port: 2379
```

**meta_store.host**

- **Type**: String
- **Default**: `127.0.0.1`
- **Impact**: ETCD connection endpoint for cluster metadata
- **When to change**: Point to your ETCD cluster (single node or load balancer)

**meta_store.port**

- **Type**: Integer
- **Default**: `2379`
- **Impact**: ETCD port
- **When to change**: Rarely (ETCD standard port is 2379)

---

### Bootstrap Configuration

```yaml
bootstrap_namespaces:
  - "default"

auto_create_topics: true
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
  assignment_strategy: "balanced"
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
- **Default**: `balanced`
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
- **Impact**: How often brokers publish resource metrics to ETCD
- **When to change**:
  - `5-10`: Testing, rapid response to changes (higher ETCD traffic)
  - `30-60`: Production, balanced overhead
  - `>60`: Low-traffic clusters, reduce ETCD load

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

## WAL and Cloud Storage Configuration

Cloud-native persistence: local WAL (fast writes) + background cloud uploads (durability). See [Persistence Architecture](../architecture/persistence.md) for details.

```yaml
wal_cloud:
  wal: { ... }
  uploader: { ... }
  cloud: { ... }
  metadata: { ... }
```

### Local Write-Ahead Log (WAL)

```yaml
wal:
  dir: "./danube-data/wal"
  file_name: "wal.log"
  cache_capacity: 1024
  file_sync:
    interval_ms: 5000
    max_batch_bytes: 10485760
  rotation:
    max_bytes: 536870912
    # max_hours: 24
  retention:
    time_minutes: 2880
    size_mb: 20480
    check_interval_minutes: 5
```

**wal.dir**

- **Type**: String (path) or `null`
- **Default**: `./danube-data/wal`
- **Impact**: Root directory for WAL files (per-topic subdirectories created automatically)
- **When to change**:
  - Point to fast SSD/NVMe for best performance
  - Set to `null` for in-memory mode (testing only, no durability)

**wal.file_name**

- **Type**: String
- **Default**: `wal.log`
- **Impact**: Base filename (rotation creates `wal.0.log`, `wal.1.log`, etc.)
- **When to change**: Rarely needed

**wal.cache_capacity**

- **Type**: Integer (number of messages)
- **Default**: `1024`
- **Impact**: In-memory replay cache for consumer reads
- **When to change**:
  - Higher (`4096+`): Better consumer hit rates, more memory usage
  - Lower (`512`): Less memory, more disk reads for catch-up consumers

### File Sync (Durability)

**wal.file_sync.interval_ms**

- **Type**: Integer (milliseconds)
- **Default**: `5000` (5 seconds)
- **Impact**: How often buffered writes are fsynced to disk
- **Trade-offs**:

| Value | Checkpoint Freshness | Fsync Pressure | Data Loss Window |
|-------|---------------------|----------------|------------------|
| 1000 (1s) | Very fresh | High | 1 second |
| 5000 (5s) | Fresh | **Balanced** | 5 seconds |
| 10000 (10s) | Stale | Low | 10 seconds |

**When to change**:

- Low-latency: `1000` (1s)
- Production: `5000` (5s) - recommended
- High-throughput: `10000` (10s)

**wal.file_sync.max_batch_bytes**

- **Type**: Integer (bytes)
- **Default**: `10485760` (10 MiB)
- **Impact**: Force flush when buffer reaches this size
- **When to change**:
  - Increase for high-volume topics (better throughput)
  - Decrease for lower latency (more frequent fsyncs)

### File Rotation

**wal.rotation.max_bytes**

- **Type**: Integer (bytes)
- **Default**: `536870912` (512 MiB)
- **Impact**: Create new WAL file when current file exceeds this size
- **When to change**:
  - `512 MiB`: Balanced (recommended)
  - `1 GiB`: Fewer files, larger cloud objects
  - `256 MiB`: More files, smaller cloud objects

**wal.rotation.max_hours**

- **Type**: Integer (hours, optional)
- **Default**: Disabled (commented out)
- **Impact**: Rotate even if size threshold not reached
- **When to enable**: Good for low-traffic topics to prevent infinitely old files
- **Recommended**: `24` hours for production

### WAL Retention (Local Cleanup)

**wal.retention.time_minutes**

- **Type**: Integer (minutes)
- **Default**: `2880` (48 hours)
- **Impact**: Delete local WAL files older than this (after cloud upload)
- **When to change**:
  - Production: `2880` (48h) for safety margin
  - Space-constrained: `1440` (24h) minimum
  - **Must be larger than uploader interval**

**wal.retention.size_mb**

- **Type**: Integer (megabytes per topic)
- **Default**: `20480` (20 GiB)
- **Impact**: Delete oldest files when total exceeds this
- **When to change**:
  - High-volume: `20480` (20 GiB) or higher
  - Space-constrained: `10240` (10 GiB) minimum

**wal.retention.check_interval_minutes**

- **Type**: Integer (minutes)
- **Default**: `5`
- **Impact**: How often retention policy runs
- **When to change**: Rarely (5 minutes is responsive without overhead)

---

### Cloud Uploader

```yaml
uploader:
  enabled: true
  interval_seconds: 30
  root_prefix: "/danube-data"
  max_object_mb: 256
```

**uploader.enabled**

- **Type**: Boolean
- **Default**: `true`
- **Impact**: Enable/disable background cloud uploads
- **When to disable**: Local-only testing, ephemeral workloads
- **When to enable**: Production (always)

**uploader.interval_seconds**

- **Type**: Integer (seconds)
- **Default**: `300` (5 minutes)
- **Impact**: Upload cycle frequency

| Value | RPO (Recovery Point) | Cloud Overhead | Use Case |
|-------|---------------------|----------------|----------|
| 30s | 30 seconds | High | Testing, fast feedback |
| 60s | 1 minute | Medium-High | High-durability production |
| 300s | 5 minutes | **Balanced** | General production |
| 600s | 10 minutes | Low | Cost-sensitive |

**uploader.root_prefix**

- **Type**: String
- **Default**: `/danube-data`
- **Impact**: ETCD metadata prefix for cloud object descriptors
- **When to change**: Only for multiple independent Danube clusters sharing ETCD

**uploader.max_object_mb**

- **Type**: Integer (megabytes, optional)
- **Default**: `256`
- **Impact**: Maximum size per cloud object
- **When to change**:
  - `256 MB`: Optimal for S3/GCS multipart uploads (recommended)
  - Larger: Fewer objects, higher per-object latency
  - Smaller: More objects, better parallelism

---

### Cloud Storage Backend

```yaml
cloud:
  backend: "fs"
  root: "./danube-data/cloud-storage"
  # Backend-specific options below
```

**cloud.backend**

- **Type**: String enum
- **Options**: `fs` | `s3` | `gcs` | `azblob` | `memory`
- **Default**: `fs`
- **Impact**: Cloud storage provider

| Backend | Use Case | Multi-Broker Support |
|---------|----------|---------------------|
| `memory` | Testing only (no durability) | No |
| `fs` | Local development, shared NFS/EFS | Yes (with shared storage) |
| `s3` | Production (AWS, MinIO, etc.) | Yes |
| `gcs` | Production (Google Cloud) | Yes |
| `azblob` | Production (Azure) | Yes |

**cloud.root**

- **Type**: String (backend-specific format)
- **Impact**: Storage location
- **Formats**:
  - `fs`: `./path/to/directory`
  - `s3`: `s3://bucket-name/prefix`
  - `gcs`: `gcs://bucket-name/prefix`
  - `azblob`: `container-name/prefix`
  - `memory`: `namespace-prefix`
- **Multi-broker requirement**: Must be shared/accessible across all brokers

### Backend-Specific Options

**Amazon S3:**

```yaml
cloud:
  backend: "s3"
  root: "s3://my-bucket/danube-data"
  region: "us-east-1"
  endpoint: "https://s3.us-east-1.amazonaws.com"  # Optional
  access_key: "${AWS_ACCESS_KEY_ID}"              # Or use IAM roles
  secret_key: "${AWS_SECRET_ACCESS_KEY}"
  anonymous: false
```

**Google Cloud Storage:**

```yaml
cloud:
  backend: "gcs"
  root: "gcs://my-bucket/danube-data"
  project: "my-gcp-project"
  credentials_path: "/path/to/service-account.json"
  # Or credentials_json: "{ ... }"  # Inline JSON string
```

**Azure Blob Storage:**

```yaml
cloud:
  backend: "azblob"
  root: "my-container/danube-data"
  endpoint: "https://myaccount.blob.core.windows.net"
  account_name: "myaccount"
  account_key: "${AZURE_STORAGE_KEY}"
```

**Local Filesystem:**

```yaml
cloud:
  backend: "fs"
  root: "./danube-data/cloud-storage"
  # For multi-broker: use shared storage like NFS
  # root: "/mnt/shared-nfs/danube-cloud"
```

---

### Metadata Store

```yaml
metadata:
  etcd_endpoint: "127.0.0.1:2379"
  in_memory: false
```

**metadata.etcd_endpoint**

- **Type**: String (host:port)
- **Default**: `127.0.0.1:2379`
- **Impact**: ETCD endpoint for storing cloud object descriptors and indexes
- **When to change**: Should match `meta_store.host:port` for consistency
- **Production**: Use clustered ETCD for high availability

**metadata.in_memory**

- **Type**: Boolean
- **Default**: `false`
- **Impact**: Use in-memory metadata storage (no persistence)
- **When to enable**: Testing only (ephemeral)
- **Production**: Always `false`

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
