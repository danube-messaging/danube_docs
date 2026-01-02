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
External Systems ↔ Connectors ↔ danube-connect-core ↔ danube-client ↔ Danube Broker
```

**Connectors** are standalone binaries that:

1. Connect to external systems (MQTT, databases, HTTP APIs, etc.)
2. Use **danube-connect-core** SDK for Danube communication
3. Transform data between external formats and Danube messages
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

### Production Ready

| Connector | Type | Description |
|-----------|------|-------------|
| [MQTT](https://github.com/danube-messaging/danube-connect/tree/main/connectors/source-mqtt) | Source | IoT device integration (MQTT 3.1.1) |
| [HTTP Webhook](https://github.com/danube-messaging/danube-connect/tree/main/connectors/source-webhook) | Source | Universal webhook ingestion |
| [Qdrant](https://github.com/danube-messaging/danube-connect/tree/main/connectors/sink-qdrant) | Sink | Vector embeddings for RAG/AI |
| [SurrealDB](https://github.com/danube-messaging/danube-connect/tree/main/connectors/sink-surrealdb) | Sink | Multi-model database storage |
| [Delta Lake](https://github.com/danube-messaging/danube-connect/tree/main/connectors/sink-deltalake) | Sink | ACID data lake ingestion |

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
  -v $(pwd)/config.toml:/config.toml \
  danube-connect/source-mqtt:latest
```

**That's it!** The connector handles:

- Connection management to Danube
- Message transformation and routing
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

### 📦 Modular Architecture

Clean separation between connector framework and implementations

### 🚀 Cloud Native

Docker-first with Kubernetes support, horizontal scaling

### 📊 Observable

Prometheus metrics, structured logging, health endpoints

### ⚡ High Performance

Async I/O, batching, connection pooling, parallel processing

### 🦀 Pure Rust

Memory-safe, high-performance, zero-cost abstractions

---

## Learn More

- **[Connector Architecture](danube_connect_architecture.md)** - Deep dive into design and concepts
- **[Building Connectors](danube_connect_development.md)** - Create your own connector
- **[GitHub Repository](https://github.com/danube-messaging/danube-connect)** - Source code and examples

---

## Community & Support

- **GitHub Issues:** [Report bugs or request connectors](https://github.com/danube-messaging/danube-connect/issues)
- **Source Code:** [danube-messaging/danube-connect](https://github.com/danube-messaging/danube-connect)
- **Examples:** [Complete connector examples](https://github.com/danube-messaging/danube-connect/tree/main/examples)
