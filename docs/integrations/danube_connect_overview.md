# Danube Connect

Connector ecosystem for seamless integration with external systems

---

## What is Danube Connect?

Danube Connect is a **plug-and-play connector framework** that enables Danube to integrate with external systems: databases, message queues, IoT protocols, analytics platforms, and more—without compromising the broker's safety, stability, or performance.

Instead of embedding integrations directly into the Danube broker (monolithic approach), connectors run as **standalone processes** that communicate with Danube via gRPC. This architecture ensures:

- 🛡️ **Isolation** - Connector failures never crash the broker
- 📈 **Scalability** - Scale connectors independently from brokers
- 🔌 **Modularity** - Add or remove integrations without touching core Danube
- 🦀 **Memory Safety** - Pure Rust implementation with zero FFI in the broker

---

## Architecture

```bash
External Systems ↔ Connector Adapter ↔ danube-connect-core ↔ danube-client ↔ Danube Broker
```

**Connectors** are standalone binaries that:

1. Connect to external systems (MQTT, databases, HTTP APIs, etc.)
2. Use **danube-connect-core** for shared runtime concerns
3. Transform data between external formats and generic `SourceRecord` / `SinkRecord` values
4. Run independently with their own lifecycle and resources

**Key principle:** The Danube broker remains "dumb" and pure Rust—it knows nothing about external systems.

---

## Connector Types

### Source Connectors (External → Danube)

Import data **into** Danube from external systems.

**Examples:**

- **MQTT Source** - Bridge IoT devices to Danube topics
- **HTTP Webhook Source** - Ingest webhooks from SaaS platforms
- **PostgreSQL CDC** - Stream database changes to Danube
- **Kafka Source** - Migrate from Kafka to Danube

**Use cases:** IoT data ingestion, event streaming, change data capture, system integration

---

### Sink Connectors (Danube → External)

Export data **from** Danube to external systems.

**Examples:**

- **Qdrant Sink** - Stream vectors to RAG/AI pipelines
- **Delta Lake Sink** - Archive messages to data lakes (S3, Azure, GCS)
- **SurrealDB Sink** - Store events in multi-model databases
- **ClickHouse Sink** - Real-time analytics and feature stores

**Use cases:** Data archival, analytics, machine learning, system integration

---

## Available Connectors

### Already Available

| Connector | Type | Description |
|-----------|------|-------------|
| [MQTT](https://github.com/danube-messaging/danube-connectors/tree/main/source-mqtt) | Source | IoT device integration (MQTT 3.1.1) |
| [HTTP Webhook](https://github.com/danube-messaging/danube-connectors/tree/main/source-webhook) | Source | Universal webhook ingestion |
| [Qdrant](https://github.com/danube-messaging/danube-connectors/tree/main/sink-qdrant) | Sink | Vector embeddings for RAG/AI |
| [SurrealDB](https://github.com/danube-messaging/danube-connectors/tree/main/sink-surrealdb) | Sink | Multi-model database storage |
| [Delta Lake](https://github.com/danube-messaging/danube-connectors/tree/main/sink-deltalake) | Sink | ACID data lake ingestion |

### Coming Soon

- OpenTelemetry Source (traces, metrics, logs)
- PostgreSQL CDC Source
- LanceDB Sink (vector search)
- ClickHouse Sink (analytics)
- GreptimeDB Sink (observability)

---

## Quick Start

### Running a Connector

Deploy connectors using Docker:

```bash
docker run -d \
  -e DANUBE_SERVICE_URL=http://danube-broker:6650 \
  -e CONNECTOR_NAME=mqtt-bridge \
  -e CONNECTOR_CONFIG_PATH=/config/connector.toml \
  -v $(pwd)/config.toml:/config/connector.toml:ro \
  ghcr.io/danube-messaging/danube-source-mqtt:latest
```

**That's it!** The connector handles:

- Connection management to Danube
- Shared message processing runtime
- Retry logic and error handling
- Metrics and health checks
- Graceful shutdown

---

## Why Danube Connect?

### Versus Embedding in Broker

| Embedded Integrations | Danube Connect |
|-----------------------|----------------|
| ❌ Broker crashes if integration fails | ✅ Isolated processes |
| ❌ Tight coupling, hard to maintain | ✅ Clean separation |
| ❌ Bloated broker binary | ✅ Lightweight core |
| ❌ All-or-nothing scaling | ✅ Independent scaling |

### Versus Custom Scripts

| DIY Integration Scripts | Danube Connect |
|------------------------|----------------|
| ❌ Manual retry logic | ✅ Built-in exponential backoff |
| ❌ No observability | ✅ Prometheus metrics + health checks |
| ❌ Ad-hoc error handling | ✅ Standardized error types |
| ❌ Reinvent the wheel | ✅ Reusable SDK framework |

---

## Key Features

### 🔄 Bidirectional Data Flow

Both source (import) and sink (export) connectors supported

### 📋 Schema Registry Integration

**Automatic schema-aware serialization/deserialization**

- Connectors work with typed `serde_json::Value` data, not raw bytes
- Runtime handles all schema operations (fetch, cache, validate, serialize)
- Support for JSON Schema, String, Bytes, Number, and Avro flows
- Schema evolution with version strategies (latest, pinned, minimum)
- **Zero schema boilerplate** in your connector code

### 📦 Modular Architecture

Clean separation between connector framework and implementations

### 🚀 Cloud Native

Docker-first with Kubernetes support, horizontal scaling

### 📊 Observable

Prometheus metrics, structured logging, health endpoints

### ⚡ High Performance

Async I/O, runtime-managed sink batching, streaming source support, connection pooling, and schema caching

### 🦀 Pure Rust

Memory-safe, high-performance, zero-cost abstractions

---

## Learn More

- **[Connector Architecture](danube_connect_architecture.md)** - Deep dive into design and concepts
- **[Build Source Connector](source_connector_development.md)** - Create your own source connector
- **[Build Sink Connector](sink_connector_development.md)** - Create your own sink connector
- **[GitHub Connector Core](https://github.com/danube-messaging/danube-connect-core)** - Connector SDK source code
- **[GitHub Connectors Repo](https://github.com/danube-messaging/danube-connectors)** - Connectors source code and full examples

---

## Community & Support

- **GitHub Issues:** [Report bugs or request connectors](https://github.com/danube-messaging/danube-connectors/issues)
- **Source Code:** [danube-messaging/danube-connect](https://github.com/danube-messaging/danube-connectors)
- **Examples:** [Connector examples and end-to-end flows](https://github.com/danube-messaging/danube-connectors/tree/main)
