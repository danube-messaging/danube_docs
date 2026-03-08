# Danube Connect Architecture

**High-level design and core concepts**

---

## Design Philosophy

### Isolation & Safety

The Danube broker remains **pure Rust** with zero knowledge of external systems. Connectors run as **separate processes**, ensuring that a failing connector never crashes the broker.

**Key principle:** Process-level boundaries guarantee fault isolation, a misconfigured PostgreSQL connector cannot impact message delivery for other topics or tenants.

---

### Shared Core Library

The `danube-connect-core` SDK eliminates boilerplate by handling the generic runtime concerns that every connector shares.

**Connector developers** implement simple traits:

- `SourceConnector` - Import data into Danube
- `SinkConnector` - Export data from Danube

**Core SDK handles:**

- Danube client connection management
- configuration loading helpers and shared root config
- sink batching, batch flush timing, and acknowledgments
- source polling loops or streaming envelope drain loops
- schema-aware serialization and deserialization
- automatic retry with exponential backoff
- Prometheus metrics export and health checks
- Signal handling and graceful shutdown

---

## Architecture Layers

```bash
External Systems (MQTT, databases, APIs)
          ↓
Your Connector Logic (implement trait)
          ↓
danube-connect-core (SDK framework)
          ↓
danube-client (gRPC communication)
          ↓
Danube Broker Cluster
```

**Separation of concerns:**

- **Connector responsibility:** External system integration and data transformation
- **SDK responsibility:** Danube communication, lifecycle, retries, metrics

---

## Connector Types

### Source Connectors (External → Danube)

Poll or subscribe to external systems and publish messages to Danube topics.

Source connectors can now run in **two modes**:

- `Polling` - runtime calls `poll()` on an interval
- `Streaming` - connector pushes `SourceEnvelope` values to the runtime with `SourceSender`

**Key responsibilities:**

- Connect to external system
- Read or receive external events
- Transform external format into `SourceRecord` or `SourceEnvelope`
- Route to appropriate Danube topics
- Optionally manage offsets/checkpoints through `Offset`

---

### Sink Connectors (Danube → External)

Consume messages from Danube topics and write to external systems.

**Key responsibilities:**

- Subscribe to Danube topics
- Transform Danube messages to external format
- Perform connector-specific validation and routing
- Execute bulk writes to the external system from `process_batch()`

---

## The danube-connect-core SDK

### Core Traits

Connectors implement the danube-connect-core traits:

#### **SourceConnector**

```rust
#[async_trait]
pub trait SourceConnector {
    async fn initialize(&mut self, config: ConnectorConfig) -> Result<()>;

    async fn producer_configs(&self) -> Result<Vec<ProducerConfig>>;

    fn mode(&self) -> SourceConnectorMode {
        SourceConnectorMode::Polling
    }

    async fn start_streaming(&mut self, sender: SourceSender) -> Result<()>;

    async fn poll(&mut self) -> Result<Vec<SourceEnvelope>>;

    async fn commit(&mut self, offsets: Vec<Offset>) -> Result<()>;

    async fn shutdown(&mut self) -> Result<()>;

    async fn health_check(&self) -> Result<()>;
}
```

#### **SinkConnector**

```rust
#[async_trait]
pub trait SinkConnector {
    async fn initialize(&mut self, config: ConnectorConfig) -> Result<()>;

    async fn consumer_configs(&self) -> Result<Vec<ConsumerConfig>>;

    async fn process_batch(&mut self, records: Vec<SinkRecord>) -> Result<()>;

    async fn shutdown(&mut self) -> Result<()>;

    async fn health_check(&self) -> Result<()>;
}
```

---

### Message Types

**SinkRecord** - Message consumed from Danube (Danube → External System)

**The runtime handles schema-aware deserialization automatically.** Your connector receives typed data as `serde_json::Value`, already deserialized based on the message's schema.

**API:**

- `payload()` - Returns `&serde_json::Value` (typed data, not raw bytes)
- `as_type<T>()` - Deserialize payload to specific Rust type
- `schema()` - Get schema information (subject, version, type) if message has schema
- `attributes()` - Metadata key-value pairs from producer
- `topic()`, `publish_time()`, `producer_name()` - Message metadata
- `routing_context()` / `context()` - Generic routing and record context helpers

**Example:**

```rust
async fn process_batch(&mut self, records: Vec<SinkRecord>) -> ConnectorResult<()> {
    for record in records {
        // Access as typed data (already deserialized by runtime)
        let payload = record.payload();
        let user_id = payload["user_id"].as_str().unwrap();

        // Or deserialize to struct
        let event: MyEvent = record.as_type()?;

        // Access generic routing context
        let context = record.context();
        println!("topic={} producer={:?}", context.topic(), context.producer_name());

        if let Some(schema) = record.schema() {
            println!("Schema: {} v{}", schema.subject, schema.version);
        }

        let _ = (user_id, event);
    }

    Ok(())
}
```

---

**SourceRecord** - Message to publish to Danube (External System → Danube)

**You provide typed data as `serde_json::Value`.** The runtime handles schema-aware serialization before sending to Danube.

**API:**

- `new(topic, payload)` - Create from `serde_json::Value`
- `from_json(topic, data)` - Create from any JSON-serializable type
- `from_string(topic, text)` - Create from string
- `from_number(topic, value)` - Create from numeric payloads
- `from_bytes(topic, bytes)` - Create from binary payloads
- `with_attribute(key, value)` - Add metadata attribute
- `with_attributes(map)` - Add multiple attributes
- `with_key(key)` - Set routing key for partitioning
- `payload()` / `context()` - Access payload and generic routing context

**Example:**

```rust
use serde_json::json;
use danube_connect_core::{Offset, SourceEnvelope};

// From JSON value
let record = SourceRecord::new("/events/users", json!({
    "user_id": "123",
    "action": "login"
})).with_attribute("source", "api");

// From struct (automatically serialized to JSON)
#[derive(Serialize)]
struct Event { user_id: String, action: String }

let event = Event { user_id: "123".into(), action: "login".into() };
let record = SourceRecord::from_json("/events/users", &event)?
    .with_attribute("source", "api")
    .with_key(&event.user_id);  // For partitioning

// Attach an offset/checkpoint when needed
let envelope = SourceEnvelope::with_offset(record, Offset::new("api-page", 42));
```

---

### Runtime Management

The SDK provides `SourceRuntime` and `SinkRuntime` that handle:

1. **Lifecycle** - Initialize connector, create Danube clients, start message loops
2. **Message processing** - Polling or streaming for sources, runtime-managed batching for sinks
3. **Error handling** - Automatic retry with exponential backoff
4. **Schema handling** - Serialize source payloads and deserialize sink payloads
5. **Acknowledgments** - Ack successfully processed sink messages and commit source offsets
6. **Shutdown** - Graceful cleanup on SIGTERM/SIGINT
7. **Observability** - Expose Prometheus metrics and health endpoints

---

### Multi-Topic Support

A single connector can handle **multiple topics** with different configurations.

**Source connector example:**

- One MQTT connector routes `sensors/#` → `/iot/sensors` (8 partitions, reliable)
- Same connector routes `debug/#` → `/iot/debug` (1 partition, non-reliable)

**Sink connector example:**

- One ClickHouse connector consumes from `/analytics/events` and `/analytics/metrics`
- Writes both to different tables in the same database

---

## Schema Registry Integration

**Critical feature:** The runtime automatically handles schema-aware serialization and deserialization, eliminating the need for connectors to manage schema logic.

### How It Works

**For Sink Connectors (Consuming):**

1. **Message arrives** with schema ID
2. **Runtime fetches schema** from registry (cached for performance)
3. **Runtime deserializes** payload based on schema type (JSON, String, Bytes, Avro, Protobuf)
4. **Your connector receives** `SinkRecord` with typed `serde_json::Value` payload
5. **You process** the typed data without worrying about raw bytes or schemas

**For Source Connectors (Publishing):**

1. **You create** `SourceRecord` with `serde_json::Value` payload
2. **Runtime serializes** payload based on configured schema type
3. **Runtime registers/validates** schema with registry
4. **Message is sent** to Danube with schema ID attached
5. **Consumers** automatically deserialize using the schema

### Schema Configuration

Define schemas per topic in your connector configuration:

```toml
# Core Danube settings
danube_service_url = "http://danube-broker:6650"
connector_name = "my-connector"

# Schema mappings for topics
[[schemas]]
topic = "/events/users"
subject = "user-events"
schema_type = "json_schema"
schema_file = "schemas/user-event.json"
version_strategy = "latest"  # or "pinned" or "minimum"

[[schemas]]
topic = "/iot/sensors"
subject = "sensor-data"
schema_type = "json_schema"
schema_file = "schemas/sensor.json"
version_strategy = { pinned = 2 }  # Pin to version 2

[[schemas]]
topic = "/raw/telemetry"
subject = "telemetry"
schema_type = "bytes"  # Binary data
```

### Supported Schema Types

| Type | Description | Serialization | Use Case |
|------|-------------|---------------|----------|
| `json_schema` | JSON Schema | JSON | Structured events, APIs |
| `string` | UTF-8 text | String bytes | Logs, text messages |
| `bytes` | Raw binary | Base64 in JSON | Binary data, images |
| `number` | Numeric data | JSON number | Metrics, counters |
| `avro` | Apache Avro | JSON payload with Avro validation | High-performance schemas |
| `protobuf` | Protocol Buffers | Planned | gRPC, microservices |

**Note:** JSON Schema, String, Bytes, Number, and Avro flows are supported in the current runtime model. Protobuf serialization/deserialization is not implemented yet.

### Version Strategies

Control schema evolution with version strategies:

```toml
# Latest version (default)
version_strategy = "latest"

# Pin to specific version
version_strategy = { pinned = 3 }

# Minimum version (use >= version)
version_strategy = { minimum = 2 }
```

### Connector Benefits

**What you DON'T need to do:**

- ❌ Fetch schemas from registry
- ❌ Deserialize/serialize payloads manually
- ❌ Handle schema versions
- ❌ Manage schema caching
- ❌ Deal with raw bytes

**What you DO:**

- ✅ Work with typed `serde_json::Value` data
- ✅ Focus on business logic
- ✅ Let the runtime handle all schema operations

### Example: Schema-Aware Sink

```rust
async fn process_batch(&mut self, records: Vec<SinkRecord>) -> ConnectorResult<()> {
    for record in records {
        let payload = record.payload();

        if let Some(schema) = record.schema() {
            info!("Processing message with schema: {} v{}",
                  schema.subject, schema.version);
        }

        let user_id = payload["user_id"].as_str()
            .ok_or_else(|| ConnectorError::invalid_data("Missing user_id", vec![]))?;

        #[derive(Deserialize)]
        struct UserEvent {
            user_id: String,
            action: String,
            timestamp: u64,
        }

        let event: UserEvent = record.as_type()?;
        self.database.insert_user_event(event).await?;
        let _ = user_id;
    }

    Ok(())
}
```

### Example: Schema-Aware Source

```rust
async fn poll(&mut self) -> ConnectorResult<Vec<SourceEnvelope>> {
    let external_data = self.client.fetch_events().await?;

    let records: Vec<SourceEnvelope> = external_data
        .into_iter()
        .map(|event| {
            SourceRecord::from_json("/events/users", &event)
                .unwrap()
                .with_attribute("source", "external-api")
                .into()
        })
        .collect();

    Ok(records)
}
```

**The runtime automatically:**

- Serializes each record's payload based on `/events/users` topic's configured schema
- Registers schema with registry if needed
- Attaches schema ID to messages
- Handles schema validation errors

---

## Configuration Pattern

Connectors use a **single configuration file** combining core Danube settings with connector-specific settings:

```toml
# Core Danube settings (provided by danube-connect-core)
danube_service_url = "http://danube-broker:6650"
connector_name = "mqtt-bridge"

[processing]
batch_size = 500
batch_timeout_ms = 1000
poll_interval_ms = 100
metrics_port = 9090

# Connector-specific settings
[mqtt]
broker_host = "mosquitto"
broker_port = 1883

[[mqtt.routes]]
from = "sensors/#"
to = "/iot/sensors"
partitions = 8
```

**Environment variable overrides:**

- Mandatory fields: `DANUBE_SERVICE_URL`, `CONNECTOR_NAME`
- Secrets and connection endpoints: `MQTT_PASSWORD`, `QDRANT_URL`, API keys, etc.
- Structural config such as `routes`, `from`, `to`, subscriptions, and processing settings should stay in TOML

**See:** [Configuration Guide](https://github.com/danube-messaging/danube-connect/blob/main/info/unified_configuration_guide.md) for complete details

---

## Deployment Architecture

Connectors run as **standalone Docker containers** alongside your Danube cluster:

```bash
┌─────────────────────────────┐
│  External Systems           │
│  (MQTT, DBs, APIs)          │
└──────────┬──────────────────┘
           ↓
┌──────────────────────────────┐
│  Connector Layer             │
│  (Docker Containers)         │
│  ┌────────┐  ┌────────┐     │
│  │ MQTT   │  │ Delta  │ ... │
│  │ Source │  │ Sink   │     │
│  └────┬───┘  └───┬────┘     │
└───────┼──────────┼───────────┘
        ↓          ↓
┌───────────────────────────────┐
│  Danube Broker Cluster        │
└───────────────────────────────┘
```

**Scaling:**

- Connectors scale independently from brokers
- Run multiple instances for high availability
- No impact on broker resources

---

## Error Handling

The SDK categorizes errors into three types:

1. **Retryable** - Temporary failures (network issues, rate limits) → automatic retry with exponential backoff
2. **Invalid Data** - Malformed messages that can never be processed → log and skip
3. **Fatal** - Unrecoverable errors → graceful shutdown, let orchestrator restart

**Your connector:** Return the appropriate error type, the SDK handles the rest

---

## Observability

Every connector automatically exposes:

**Prometheus metrics** (`http://connector:9090/metrics`):

- `connector_messages_received` / `connector_messages_sent`
- `connector_errors_total`
- `connector_processing_duration_seconds`

**Health checks** (`http://connector:9090/health`):

- `healthy`, `degraded`, or `unhealthy`

**Structured logging:**

- JSON-formatted logs with trace IDs
- Configurable log levels via `LOG_LEVEL` env var

---

## Next Steps

- **[Build Source Connector](source_connector_development.md)** - How to build a source connector
- **[Build Sink Connector](sink_connector_development.md)** - How to build a sink connector
- **[GitHub Repository](https://github.com/danube-messaging/danube-connectors)** - Source code and examples
- **[danube-connect-core API](https://docs.rs/danube-connect-core)** - SDK documentation
