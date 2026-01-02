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

**SinkRecord** - Message consumed from Danube

- `payload()` - Raw bytes
- `payload_str()` - UTF-8 string
- `payload_json<T>()` - Deserialize to struct
- `attributes()` - Metadata key-value pairs
- `topic()`, `offset()`, `publish_time()` - Message metadata

**SourceRecord** - Message to publish to Danube

- Created with `SourceRecord::from_json(topic, &data)`
- Builder pattern: `.with_attribute(k, v)`, `.with_key(key)`
- Per-record topic routing and configuration

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
