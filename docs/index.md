# Welcome to Danube Messaging

[Danube](https://github.com/danube-messaging/danube) is a lightweight, cloud‑native messaging platform built in Rust. It delivers sub‑second dispatch with cloud economics by combining a Write‑Ahead Log (WAL) with object storage, so you get low‑latency pub/sub and durable streaming—on one broker.

Danube enables one or many **producers** publish to **topics**, and multiple **consumers** receive messages via named **subscriptions**. Choose Non‑Reliable (best‑effort pub/sub) or Reliable (at‑least‑once streaming) per topic to match your workload.

For design details, see the [Architecture](architecture/architecture.md).

## Try Danube in minutes

**[Docker Compose Quickstart](getting_started/Danube_docker_compose.md)**: Use the provided Docker Compose setup with MinIO and ETCD.

## Danube capabilities

### 🏗️ **Cluster & Broker Characteristics**

- **Stateless brokers**: Metadata in ETCD and data in WAL/Object Storage
- **Horizontal scaling**: Add brokers in seconds; partitions rebalance automatically
- **Leader election & HA**: Automatic failover and coordination via ETCD
- **Rolling upgrades**: Restart or replace brokers with minimal disruption
- **Multi-tenancy**: Isolated namespaces with policy controls
- **Security-ready**: TLS/mTLS support in Admin and data paths

**Cloud-Native by Design** - Danube's architecture separates compute from storage, enabling:

### 🌩️ **Write-Ahead Log + Cloud Persistence**

- **Sub-millisecond producer acknowledgments** via local WAL
- **Asynchronous background uploads** to S3/GCS/Azure object storage
- **Automatic failover** with shared cloud state
- **Infinite retention** without local disk constraints

### ⚡ **Performance & Scalability**

- **Hot path optimization**: Messages served from in-memory WAL cache
- **Stream per subscription**: WAL + cloud storage from selected offset 
- **Multi-cloud support**: AWS S3, Google Cloud Storage, Azure Blob, MinIO

## Core features

**[Topics](architecture/topics.md)**

  - Non‑partitioned: served by a single broker.
  - Partitioned: split across brokers for scale and HA.

**[Dispatch strategies](architecture/dispatch_strategy.md)**

  - Non‑Reliable: in‑memory, best‑effort delivery, lowest latency.
  - Reliable: WAL + Cloud persistence with acknowledgments and replay.

**[Persistence (WAL + Cloud)](architecture/persistence.md)**

  - Write‑Ahead Log on local disk for fast durable writes.
  - Background uploads to object storage for durability and replay at cloud cost.
  - Seamless handoff from historical replay to live tail.

**[Subscriptions](architecture/subscriptions.md)**

  - Exclusive, Shared, Failover patterns for queueing and fan‑out.

**[Message schemas](architecture/messages.md)**
  
  - Bytes, String, Int64, JSON.

**Concept guides**

  - [Messaging Modes (Pub/Sub vs Streaming)](architecture/messaging_modes_pubsub_vs_streaming.md)
  - [Messaging Patterns (Queuing vs Pub/Sub)](architecture/messaging_patterns_queuing_vs_pubsub.md)

## Crates in the workspace

Repository: https://github.com/danube-messaging/danube

- [danube-broker](https://github.com/danube-messaging/danube/tree/main/danube-broker) – The broker service (topics, producers, consumers, subscriptions).
- [danube-core](https://github.com/danube-messaging/danube/tree/main/danube-core) – Core types, protocol, and shared logic.
- [danube-metadata-store](https://github.com/danube-messaging/danube/tree/main/danube-metadata-store) – Metadata storage and cluster coordination.
- [danube-persistent-storage](https://github.com/danube-messaging/danube/tree/main/danube-persistent-storage) – WAL and cloud persistence backends.

CLIs and client libraries:

- [danube-client](https://github.com/danube-messaging/danube/tree/main/danube-client) – Async Rust client library.
- [danube-cli](https://github.com/danube-messaging/danube/tree/main/danube-cli) – Publish/consume client CLI.
- [danube-admin-cli](https://github.com/danube-messaging/danube/tree/main/danube-admin-cli) – Admin CLI for cluster management.

## Client libraries

- [danube-client (Rust)](https://crates.io/crates/danube-client)
- [danube-go (Go)](https://pkg.go.dev/github.com/danube-messaging/danube-go)

Contributions for other languages (Python, Java, etc.) are welcome.
