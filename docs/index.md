# Welcome to Danube Messaging

🌊 [Danube Messaging](https://github.com/danube-messaging/danube) is a lightweight, cloud‑native messaging platform built in Rust. It delivers sub‑second dispatch by combining a local Write‑Ahead Log (WAL), durable segment storage, and metadata-driven recovery, so you get low-latency pub/sub and reliable streaming across local disks, shared filesystems, or object stores.

Danube enables one or many **producers** publish to **topics**, and multiple **consumers** receive messages via named **subscriptions**. Choose Non‑Reliable (best‑effort pub/sub) or Reliable (at‑least‑once streaming) per topic to match your workload.

For design details, see the [Architecture](architecture/architecture.md).

## Try Danube in minutes

**[Docker Compose Quickstart](getting_started/Danube_docker_compose.md)**: Use the provided Docker Compose setup to deploy a 3-broker cluster with MinIO and Prometheus.

## Architecture

### 🏗️ **Cluster & Broker Characteristics**

- **Embedded Raft consensus**: Metadata replicated across brokers via openraft — no ETCD, no ZooKeeper, no external dependencies
- **Horizontal scaling**: Add brokers in seconds with zero-downtime expansion
- **Intelligent load balancing**: Automatic topic placement and rebalancing across brokers
- **Broker resilience**: Automatic leader election, failover, and topic reconciliation on restart
- **Security**: TLS/mTLS, multi-method authentication, and fine-grained RBAC authorization

### 🌩️ **Write-Ahead Log + Durable Storage**

- **Flexible storage modes**: `local`, `shared_fs`, and `object_store`
- **Cloud-ready durable history**: AWS S3, Google Cloud Storage, Azure Blob, or shared filesystems depending on mode
- **Hot path optimization**: Messages served from in-memory WAL cache and local WAL files
- **Tiered historical replay**: Durable segments for older offsets with seamless handoff to the WAL tail
- **Metadata-driven recovery and topic moves**: Continuous offsets across restarts and broker transfers

### 🎯 **Intelligent Load Management**

- **Automated rebalancing**: Detects cluster imbalances and redistributes topics automatically
- **Smart topic assignment**: Places new topics on least-loaded brokers using configurable strategies
- **Resource monitoring**: Tracks CPU, memory, throughput, and backlog per broker in real-time
- **Configurable policies**: Conservative, balanced, or aggressive rebalancing based on workload
- **Graceful topic migration**: Moves topics between brokers

## Core Capabilities

### 📨 **Message Delivery**

- **[Topics](concepts/topics.md)**: Partitioned and non-partitioned with automatic load balancing
- **[Reliable Dispatch](concepts/dispatch_strategy.md)**: At-least-once delivery with configurable failure policies (NACK, retry backoff, dead-letter queues)
- **Non-Reliable Dispatch**: High-throughput, low-latency for real-time scenarios

### 🔄 **Subscription Models**

- **[Exclusive](concepts/subscriptions.md)**: Single consumer per subscription
- **Shared**: Load-balanced message distribution across consumers
- **Failover**: Automatic consumer failover with ordered delivery
- **Key-Shared**: Key-based message routing with per-key ordering and optional key filtering

### 📋 **Schema Registry**

- **Centralized schema management**: Single source of truth for message schemas across all topics
- **Schema versioning**: Automatic version tracking with compatibility enforcement
- **Multiple formats**: Bytes, String, Number, JSON Schema, Avro, Protobuf
- **Validation & governance**: Prevent invalid messages and ensure data quality

### 🤖 **AI-Powered Administration**

Danube features **the AI-native messaging platform administration** through the Model Context Protocol (MCP):

- **Natural language cluster management**: Manage your cluster by talking to AI assistants (Claude, Cursor, Windsurf)
- **32 intelligent tools**: Full cluster operations accessible via AI - topics, schemas, brokers, diagnostics, metrics
- **Automated troubleshooting**: AI-guided workflows for consumer lag analysis, health checks, and performance optimization
- **Multiple interfaces**: CLI commands, Web UI, or AI conversation - your choice

**Example**: Ask Claude *"What's the cluster balance?"* or *"Create a partitioned topic for analytics"* and watch it happen.

## Architecture Deep Dives

Explore how Danube works under the hood:

**[System Overview](architecture/architecture.md)** - Complete architecture diagram and component interaction

**[Load Manager & Rebalancing](architecture/load_manager_architecture.md)** - Smart topic assignment and automatic rebalancing

**[Persistence Architecture](architecture/persistence.md)** - WAL, durable segments, and recovery metadata

- Local WAL for fast appends and recent reads
- Durable segment history on local disk, shared filesystem, or object store depending on mode
- Seamless handoff from historical replay to the live WAL tail

**[Schema Registry](architecture/schema_registry_architecture.md)** - Centralized schema management

- Schema versioning and compatibility checking
- Support for JSON Schema, Avro, and Protobuf
- Data validation and governance

**[Key-Shared Dispatch](architecture/key_shared_architecture.md)** - Key-affinity routing architecture

- Consistent hashing with virtual-node ring for stable key-to-consumer assignment
- Per-key ordering with multi-message in-flight window and contiguous cursor tracking
- Glob-based key filtering, per-consumer backpressure, and inactive consumer eviction

---

## Integrations

**[Danube Connect](integrations/danube_connect_overview.md)** - Plug-and-play connector ecosystem

- Source connectors: Import data from MQTT, HTTP webhooks, databases, Kafka , etc.
- Sink connectors: Export to Delta Lake, ClickHouse, vector databases, APIs, etc.
- Pure Rust framework with automatic retries, metrics, and health checks

**Learn more:** [Architecture](integrations/danube_connect_architecture.md) | [Build Source Connector](integrations/source_connector_development.md) | [Build Sink Connector](integrations/sink_connector_development.md)

---

## Crates in the workspace

Repository: <https://github.com/danube-messaging/danube>

- [danube-broker](https://github.com/danube-messaging/danube/tree/main/danube-broker) – The broker service (topics, producers, consumers, subscriptions).
- [danube-core](https://github.com/danube-messaging/danube/tree/main/danube-core) – Core types, protocol, and shared logic.
- [danube-raft](https://github.com/danube-messaging/danube/tree/main/danube-raft) – Embedded Raft consensus and metadata replication.
- [danube-persistent-storage](https://github.com/danube-messaging/danube/tree/main/danube-persistent-storage) – WAL and durable storage engine for reliable topics.

CLIs and client libraries:

- [danube-client](https://github.com/danube-messaging/danube/tree/main/danube-client) – Async Rust client library.
- [danube-cli](https://github.com/danube-messaging/danube/tree/main/danube-cli) – Publish/consume client CLI.
- [danube-admin](https://github.com/danube-messaging/danube/tree/main/danube-admin) – Unified admin tool (CLI + AI/MCP + Web UI)

## Client libraries

- [danube-client (Rust)](https://crates.io/crates/danube-client)
- [danube-go (Go)](https://pkg.go.dev/github.com/danube-messaging/danube-go)
- [danube-java (Java)](https://central.sonatype.com/namespace/com.danube-messaging)
- [danube-client (Python)](https://pypi.org/project/danube-client/)

Contributions for other languages (NodeJs, C#, etc.) are welcome.
