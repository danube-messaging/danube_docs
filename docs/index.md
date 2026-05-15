# Welcome to Danube Messaging

🌊 [Danube Messaging](https://github.com/danube-messaging/danube) is an open-source messaging platform built in Rust for teams that need reliable pub/sub and streaming without the operational overhead. Built on [Tokio](https://tokio.rs/) and [openraft](https://github.com/databendlabs/openraft), metadata is replicated through embedded Raft consensus, so there are no external dependencies to deploy or manage.

Producers publish to **topics**, consumers receive messages via named **subscriptions**. Choose Non-Reliable (best-effort pub/sub) or Reliable (at-least-once streaming) per topic to match your workload. For design details, see the [Architecture](architecture/architecture.md).

---

## Get Started

The fastest way to try Danube, download the binary from the [releases page](https://github.com/danube-messaging/danube/releases):

```bash
danube-broker --mode standalone --data-dir ./danube-data
```

No config file, no dependencies. Broker on `127.0.0.1:6650`, admin on `127.0.0.1:50051`.

For other deployment options: [Docker Compose](getting_started/Danube_docker_compose.md) · [Kubernetes](getting_started/Danube_kubernetes.md) · [Local multi-broker](getting_started/Danube_local.md)

---

## Deployment Modes

Danube runs as a single binary (`danube-broker`) in three modes. Choose based on your use case:

### 🖥️ Standalone

A single self-contained broker with zero config. Ideal for development, CI, and single-server deployments.

```bash
danube-broker --mode standalone --data-dir ./danube-data
```

### 🌐 Cluster

Multiple brokers forming a Raft consensus group with leader election, automated topic distribution, and load-based rebalancing. The recommended mode for production.

```bash
danube-broker --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6650 --raft-addr 0.0.0.0:7650 \
  --data-dir ./data/raft --seed-nodes "node1:7650,node2:7650,node3:7650"
```

### 🏭 Edge

A lightweight MQTT gateway that ingests IoT device data at the edge and replicates it to the central cluster. Devices publish via standard MQTT (v3.1.1 / v5.0); the edge broker validates payloads, buffers into a local WAL, and continuously replicates to the cloud.

```bash
danube-broker --mode edge --data-dir ./edge-data --edge-config edge.yaml
```

📖 **[Full Broker Modes guide](getting_started/Broker_modes.md)** with configuration reference and examples.

---

## Core Capabilities

📨 **Message Delivery** : [Topics](concepts/topics.md) (partitioned and non-partitioned), [reliable dispatch](concepts/dispatch_strategy.md) (at-least-once with NACK, retry backoff, dead-letter queues), and non-reliable high-throughput dispatch.

🔄 [**Subscriptions**](concepts/subscriptions.md) : Exclusive, Shared, Failover, and Key-Shared (per-key ordering via consistent hashing with optional key filtering).

💾 **Persistence** : Local WAL for fast writes, with optional durable segments on [shared filesystems or object stores](concepts/persistence.md) (S3, GCS, Azure Blob). Tiered replay and metadata-driven recovery across restarts and broker transfers.

📋 **Schema Registry** : Centralized [schema management](concepts/schema_registry_guide.md) with versioning and compatibility enforcement. Supports JSON Schema, Avro, and Protobuf.

🔒 **Security** : TLS/mTLS encryption, JWT and API-key [authentication](concepts/security.md), fine-grained RBAC authorization with default-deny semantics.

🤖 **AI Administration** : [MCP integration](danube_admin/ai_admin_assistant.md) for natural language cluster management via Claude, Cursor, and Windsurf. 40+ tools covering topics, schemas, brokers, diagnostics, and metrics.

---

## Architecture

Explore how Danube works under the hood:

- **[System Overview](architecture/architecture.md)** : component interaction, message flow, and cluster topology
- **[Load Manager & Rebalancing](architecture/load_manager_architecture.md)** : topic assignment strategies and automated rebalancing
- **[Persistence Architecture](architecture/persistence.md)** : WAL, durable segments, tiered reads, and recovery
- **[Schema Registry](architecture/schema_registry_architecture.md)** : versioning, compatibility checking, and governance
- **[Key-Shared Dispatch](architecture/key_shared_architecture.md)** : consistent hashing, per-key ordering, and consumer elasticity

---

## Integrations

**[Danube Connect](integrations/danube_connect_overview.md)** : plug-and-play connector ecosystem

- Source connectors: import data from MQTT, HTTP webhooks, databases, Kafka, etc.
- Sink connectors: export to Delta Lake, ClickHouse, vector databases, APIs, etc.

**Learn more:** [Architecture](integrations/danube_connect_architecture.md) · [Build Source Connector](integrations/source_connector_development.md) · [Build Sink Connector](integrations/sink_connector_development.md)

---

## Client Libraries

- **Rust** : [danube-client](https://crates.io/crates/danube-client) · [examples](https://github.com/danube-messaging/danube/tree/main/danube-client/examples)
- **Go** : [danube-go](https://pkg.go.dev/github.com/danube-messaging/danube-go) · [examples](https://github.com/danube-messaging/danube-go/tree/main/examples)
- **Java** : [danube-java](https://central.sonatype.com/namespace/com.danube-messaging) · [examples](https://github.com/danube-messaging/danube-java/tree/main/examples)
- **Python** : [danube-client](https://pypi.org/project/danube-client/) · [examples](https://github.com/danube-messaging/danube-py/tree/main/examples)

Contributions for other languages (Node.js, C#, etc.) are welcome.

## Tools

- **[danube-cli](danube_cli/getting_started.md)** : command-line producer and consumer for quick testing
- **[danube-admin](danube_admin/getting_started.md)** : cluster administration (CLI, AI/MCP, Web UI)
