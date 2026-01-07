# Danube Connect Architecture

**High-level design and core concepts**

---

## Design Philosophy

### Isolation & Safety

The Danube broker remains **pure Rust** with zero knowledge of external systems. Connectors run as **separate processes**, ensuring that a failing connector never crashes the broker.

**Key principle:** Process-level boundaries guarantee fault isolation, a misconfigured PostgreSQL connector cannot impact message delivery for other topics or tenants.

---

### Shared Core Library

The `danube-connect-core` SDK eliminates boilerplate by handling all Danube communication, lifecycle management, retries, and observability.

**Connector developers** implement simple traits:

- `SourceConnector` - Import data into Danube
- `SinkConnector` - Export data from Danube

**Core SDK handles:**

- Danube client connection management
- Message processing loops
- Automatic retry with exponential backoff
- Prometheus metrics and health checks
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

**Key responsibilities:**

- Connect to external system
- Poll for new data or listen for events
- Transform external format to Danube messages
- Route to appropriate Danube topics
- Manage offsets/checkpoints

---

### Sink Connectors (Danube → External)

Consume messages from Danube topics and write to external systems.

**Key responsibilities:**

- Subscribe to Danube topics
- Transform Danube messages to external format
- Write to external system (with batching for efficiency)
- Handle acknowledgments

---

## The danube-connect-core SDK

### Core Traits

Connectors implements the danube-connect-core traits:

#### **SourceConnector**

```rust
#[async_trait]
pub trait SourceConnector {
    // Initialize external system connection
    async fn initialize(&mut self, config: ConnectorConfig) -> Result<()>;
    
    // Define destination Danube topics
    async fn producer_configs(&self) -> Result<Vec<ProducerConfig>>;
    
    // Poll external system for data (non-blocking)
    async fn poll(&mut self) -> Result<Vec<SourceRecord>>;
    
    // Optional: commit offsets after successful publish
    async fn commit(&mut self, offsets: Vec<Offset>) -> Result<()>;
    
    // Optional: cleanup on shutdown
    async fn shutdown(&mut self) -> Result<()>;
}
```

#### **SinkConnector**

```rust
#[async_trait]
pub trait SinkConnector {
    // Initialize external system connection
    async fn initialize(&mut self, config: ConnectorConfig) -> Result<()>;
    
    // Define which Danube topics to consume from
    async fn consumer_configs(&self) -> Result<Vec<ConsumerConfig>>;
    
    // Process a single message
    async fn process(&mut self, record: SinkRecord) -> Result<()>;
    
    // Optional: batch processing (better performance)
    async fn process_batch(&mut self, records: Vec<SinkRecord>) -> Result<()>;
    
    // Optional: cleanup on shutdown
    async fn shutdown(&mut self) -> Result<()>;
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
- `topic()`, `offset()`, `publish_time()`, `producer_name()` - Message metadata
- `message_id()` - Formatted ID for logging/debugging

**Example:**

```rust
async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    // Access as typed data (already deserialized by runtime)
    let payload = record.payload();  // &serde_json::Value
    let user_id = payload["user_id"].as_str().unwrap();
    
    // Or deserialize to struct
    let event: MyEvent = record.as_type()?;
    
    // Check schema info
    if let Some(schema) = record.schema() {
        println!("Schema: {} v{}", schema.subject, schema.version);
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
- `with_attribute(key, value)` - Add metadata attribute
- `with_attributes(map)` - Add multiple attributes
- `with_key(key)` - Set routing key for partitioning
- `payload()` - Get payload reference

**Example:**

```rust
use serde_json::json;

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
```

---

### Runtime Management

The SDK provides `SourceRuntime` and `SinkRuntime` that handle:

1. **Lifecycle** - Initialize connector, create Danube clients, start message loops
2. **Message processing** - Poll/consume messages, call your trait methods
3. **Error handling** - Automatic retry with exponential backoff
4. **Acknowledgments** - Ack successfully processed messages to Danube
5. **Shutdown** - Graceful cleanup on SIGTERM/SIGINT
6. **Observability** - Expose Prometheus metrics and health endpoints

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
| `avro` | Apache Avro | Avro binary | High-performance schemas |
| `protobuf` | Protocol Buffers | Protobuf binary | gRPC, microservices |

**Note:** Avro and Protobuf support coming soon. Currently JSON, String, Bytes, and Number are fully supported.

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
async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    // Payload is already deserialized based on schema
    let payload = record.payload();
    
    // Check what schema was used
    if let Some(schema) = record.schema() {
        info!("Processing message with schema: {} v{}", 
              schema.subject, schema.version);
    }
    
    // Access typed data directly
    let user_id = payload["user_id"].as_str()
        .ok_or_else(|| ConnectorError::invalid_data("Missing user_id", vec![]))?;
    
    // Or deserialize to struct
    #[derive(Deserialize)]
    struct UserEvent {
        user_id: String,
        action: String,
        timestamp: u64,
    }
    
    let event: UserEvent = record.as_type()?;
    
    // Write to external system with typed data
    self.database.insert_user_event(event).await?;
    
    Ok(())
}
```

### Example: Schema-Aware Source

```rust
async fn poll(&mut self) -> ConnectorResult<Vec<SourceRecord>> {
    let external_data = self.client.fetch_events().await?;
    
    let records: Vec<SourceRecord> = external_data
        .into_iter()
        .map(|event| {
            // Create record with typed JSON data
            // Runtime will serialize based on configured schema
            SourceRecord::from_json("/events/users", &event)
                .unwrap()
                .with_attribute("source", "external-api")
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

# Connector-specific settings
[mqtt]
broker_host = "mosquitto"
broker_port = 1883

[[mqtt.topic_mappings]]
mqtt_topic = "sensors/#"
danube_topic = "/iot/sensors"
partitions = 8
```

**Environment variable overrides:**

- Mandatory fields: `DANUBE_SERVICE_URL`, `CONNECTOR_NAME`
- Secrets: `MQTT_PASSWORD`, `API_KEY`, etc.
- Connector-specific overrides as needed

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

- **[Building Connectors](danube_connect_development.md)** - Create your first connector
- **[GitHub Repository](https://github.com/danube-messaging/danube-connect)** - Source code and examples
- **[danube-connect-core API](https://docs.rs/danube-connect-core)** - SDK documentation
- **[MQTT Reference](https://github.com/danube-messaging/danube-connect/tree/main/connectors/source-mqtt)** - Complete example implementation
