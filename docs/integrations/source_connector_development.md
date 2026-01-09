# Source Connector Development Guide

**Build source connectors to import data into Danube from any external system**

---

## Overview

A **source connector** imports data from external systems into Danube topics. The connector polls or listens to the external system, transforms data into `SourceRecord`s, and the runtime handles publishing to Danube with schema validation and delivery guarantees.

**Examples:** MQTT→Danube, PostgreSQL CDC→Danube, HTTP webhooks→Danube, Kafka→Danube

---

## Core Concepts

### Division of Responsibilities

| **You Handle** | **Runtime Handles** |
|----------------|---------------------|
| Connect to external system | Connect to Danube broker |
| Poll/listen for data | Publish to Danube topics |
| Transform to `SourceRecord` | Schema serialization & validation |
| Topic routing logic | Message delivery & retries |
| Error handling per system | Lifecycle & health monitoring |
| Offset tracking (optional) | Metrics & observability |

**Key insight:** You work with typed `serde_json::Value` data. The runtime handles schema-based serialization and raw byte formats.

---

## SourceConnector Trait

```rust
#[async_trait]
pub trait SourceConnector: Send + Sync {
    async fn initialize(&mut self, config: ConnectorConfig) -> ConnectorResult<()>;
    async fn producer_configs(&self) -> ConnectorResult<Vec<ProducerConfig>>;
    async fn poll(&mut self) -> ConnectorResult<Vec<SourceRecord>>;
    
    // Optional methods with defaults
    async fn commit(&mut self, offsets: Vec<Offset>) -> ConnectorResult<()> { Ok(()) }
    async fn shutdown(&mut self) -> ConnectorResult<()> { Ok(()) }
    async fn health_check(&self) -> ConnectorResult<()> { Ok(()) }
}
```

**Required:** `initialize`, `producer_configs`, `poll`  
**Optional:** `commit`, `shutdown`, `health_check`

---

## Configuration Architecture

### Unified Configuration Pattern

Use a **single configuration struct** that combines core Danube settings with connector-specific settings:

```rust
#[derive(Deserialize)]
pub struct MySourceConfig {
    #[serde(flatten)]
    pub core: ConnectorConfig,  // Danube settings + schemas
    
    pub my_source: MySettings,  // Connector-specific
}
```

**Critical:** `ConnectorConfig` is **flattened** to the root level. This means:
- ✅ Core fields appear at TOML root: `danube_service_url`, `connector_name`
- ✅ `[[schemas]]` sections at root level (not nested)
- ✅ Access schemas via `config.core.schemas`
- ❌ **Never** duplicate schemas field in your config struct

### TOML Structure

```toml
# At root level (flattened from ConnectorConfig)
danube_service_url = "http://danube-broker:6650"
connector_name = "my-source"

[[schemas]]  # ← From ConnectorConfig (flattened)
topic = "/data/events"
subject = "events-v1"
schema_type = "json_schema"
schema_file = "schemas/events.json"
auto_register = true
version_strategy = "latest"

# Connector-specific section
[my_source]
external_host = "system.example.com"
poll_interval_ms = 100

[[my_source.topic_mappings]]
source_pattern = "input/*"
danube_topic = "/data/events"
partitions = 4
```

### Configuration Loading

**Mechanism:**
1. Load TOML file from `CONNECTOR_CONFIG_PATH` environment variable
2. Apply environment variable overrides (secrets only)
3. Validate configuration
4. Pass to connector and runtime

**Environment overrides:**
- Core: `DANUBE_SERVICE_URL`, `CONNECTOR_NAME`
- Secrets: System-specific credentials (e.g., `API_KEY`, `PASSWORD`)

**Not overridable:** Topic mappings, schemas, processing settings (must be in TOML)

---

## Implementation Steps

### 1. Initialize Connection

**`async fn initialize(&mut self, config: ConnectorConfig)`**

**Purpose:** Establish connection to external system and prepare for data collection.

**What to do:**
- Create client/connection to external system
- Configure authentication (API keys, certificates, tokens)
- Set up connection pools if needed
- Subscribe to data sources (topics, streams, tables)
- Validate connectivity with test query/ping
- Store client in struct for later use in `poll()`

**Error handling:**
- Return `ConnectorError::Initialization` for connection failures
- Log detailed error context for debugging
- Fail fast - don't retry in `initialize()`

**Examples:**
- **MQTT:** Connect to broker, authenticate, subscribe to topic patterns
- **Database:** Create connection pool, validate table access
- **HTTP:** Set up client with auth headers, test endpoint
- **Kafka:** Configure consumer group, set deserializers

---

### 2. Define Producer Configs

**`async fn producer_configs(&self) -> Vec<ProducerConfig>`**

**Purpose:** Tell the runtime which Danube topics you'll publish to and their configurations.

**What to do:**
- Map your topic routing logic to Danube topics
- For each destination topic, create a `ProducerConfig`
- Attach schema configuration if defined for that topic
- Specify partitions and reliability settings

**Schema mapping:**
```rust
// Find schema for this Danube topic from config.core.schemas
let schema_config = self.schemas.iter()
    .find(|s| s.topic == danube_topic)
    .map(|s| convert_to_schema_config(s));

ProducerConfig {
    topic: danube_topic,
    partitions: 4,
    reliable_dispatch: true,
    schema_config,  // Runtime uses this for serialization
}
```

**Called once** at startup after `initialize()`. Runtime creates producers based on these configs.

---

### 3. Poll for Data

**`async fn poll(&mut self) -> Vec<SourceRecord>`**

**Purpose:** Fetch new data from external system and transform to `SourceRecord`s.

**Called:** Repeatedly at configured interval (default: 100ms)

**What to do:**

1. **Fetch data** from external system (query, consume, poll buffer)
2. **Check for data** - if none, return empty `Vec` (non-blocking)
3. **Route messages** - determine destination Danube topic for each message
4. **Transform** - convert to `SourceRecord` with typed data
5. **Return** batch of records

**Transformation mechanism:**
```rust
// From JSON-serializable struct
SourceRecord::from_json(topic, &struct_data)?

// From JSON value
SourceRecord::new(topic, serde_json::Value)

// From string
SourceRecord::from_string(topic, &text)

// From number
SourceRecord::from_number(topic, value)
```

**Runtime handles:**
- Schema validation (if configured)
- Serialization based on schema type
- Publishing to Danube
- Retries and delivery guarantees

**Polling patterns:**
- **Pull:** Query with cursor/offset, track last position
- **Push:** Drain in-memory buffer filled by subscriptions/callbacks
- **Stream:** Read batch from continuous stream

**Return empty `Vec`** when:
- No data available (normal, not an error)
- External system returns empty result
- Rate limited (after logging warning)

---

### 4. Error Handling

**Error types guide runtime behavior:**

| Error Type | When to Use | Runtime Action |
|------------|-------------|----------------|
| `Retryable` | Temporary failures (network, rate limit) | Retry with backoff |
| `Fatal` | Permanent failures (auth, bad config) | Shutdown connector |
| `Ok(vec![])` | No data or skippable errors | Continue polling |

**Mechanism:**
- **Temporary errors:** Network blips, rate limits, timeouts → `Retryable`
- **Invalid data:** Log warning, skip message → `Ok(vec![])` with empty result
- **Fatal errors:** Auth failure, config error → `Fatal` (stops connector)

**The runtime:**
- Retries `Retryable` errors with exponential backoff
- Stops connector on `Fatal` errors
- Continues normally on `Ok(vec![])`

---

### 5. Optional: Offset Management

**`async fn commit(&mut self, offsets: Vec<Offset>)`**

**Purpose:** Track processing position for exactly-once semantics and resumability.

**When to implement:**
- Exactly-once processing required
- Need to resume from specific position after restart
- External system supports offset/cursor tracking

**When to skip:**
- At-least-once is acceptable (most cases)
- Stateless data sources (webhooks, streams)
- Idempotent downstream consumers

**Mechanism:**
1. Runtime calls `commit()` after successfully publishing batch to Danube
2. You save offsets to external system or local storage
3. On restart, load offsets and resume from last committed position

---

### 6. Optional: Graceful Shutdown

**`async fn shutdown(&mut self)`**

**Purpose:** Clean up resources before connector stops.

**What to do:**
- Unsubscribe from data sources
- Flush any buffered data
- Close connections to external system
- Save final state/offsets

**Runtime guarantees:**
- `shutdown()` called on SIGTERM/SIGINT
- All pending `SourceRecord`s published before shutdown
- No new `poll()` calls after shutdown starts

---

## Schema Registry Integration

### Why Use Schemas?

**Without schemas:**
- Manual serialization/deserialization
- No validation or type safety
- Schema drift between producers/consumers
- Difficult evolution

**With schemas:**
- ✅ Runtime handles serialization based on schema type
- ✅ Automatic validation at message ingestion
- ✅ Type safety and data contracts
- ✅ Managed schema evolution
- ✅ You work with `serde_json::Value`, not bytes

### Schema Configuration

Schemas are defined in `ConnectorConfig` and accessed via `config.core.schemas`:

```toml
[[schemas]]
topic = "/iot/sensors"              # Danube topic
subject = "sensor-data-v1"          # Registry subject
schema_type = "json_schema"         # Type
schema_file = "schemas/sensor.json" # File path
auto_register = true                # Auto-register
version_strategy = "latest"         # Version
```

### Schema Types

| Type | Use Case | Schema File |
|------|----------|-------------|
| `json_schema` | Structured data, events | Required |
| `string` | Logs, plain text | Not needed |
| `bytes` | Binary data, images | Not needed |
| `number` | Metrics, counters | Not needed |

### Version Strategies

- **`"latest"`** - Always use newest version (development)
- **`{ pinned = 2 }`** - Lock to version 2 (production stability)
- **`{ minimum = 1 }`** - Use version ≥ 1 (backward compatibility)

### How It Works

**You create records:**
```rust
let record = SourceRecord::from_json("/iot/sensors", &sensor_data)?;
```

**Runtime automatically:**
1. Validates payload against JSON schema (if `json_schema` type)
2. Serializes based on `schema_type`
3. Registers schema with registry (if `auto_register = true`)
4. Attaches schema ID to message
5. Publishes to Danube

**Consumers receive:**
- Typed data already deserialized
- Schema metadata (subject, version)
- No manual schema operations needed

---

## Main Entry Point

```rust
#[tokio::main]
async fn main() -> ConnectorResult<()> {
    // 1. Initialize logging
    tracing_subscriber::fmt::init();
    
    // 2. Load configuration
    let config = MySourceConfig::load()?;
    
    // 3. Create connector with settings and schemas
    let connector = MySourceConnector::new(
        config.my_source,
        config.core.schemas.clone(),  // ← From ConnectorConfig
    );
    
    // 4. Create and run runtime (handles everything else)
    let runtime = SourceRuntime::new(connector, config.core).await?;
    runtime.run().await
}
```

**Runtime handles:**
- Danube connection
- Producer creation with schema configs
- Polling loop at configured interval
- Publishing records with retries
- Metrics and health monitoring
- Graceful shutdown on signals

---

## Best Practices

### Configuration
- ✅ Use flattened `ConnectorConfig` - schemas via `config.core.schemas`
- ✅ Never duplicate schemas field in your config
- ✅ Keep secrets in environment variables, not TOML
- ✅ Validate configuration at startup

### Schema Management
- ✅ Use `json_schema` for structured data validation
- ✅ Set `auto_register = true` for development
- ✅ Pin versions in production (`{ pinned = N }`)
- ✅ Define schemas for all structured topics
- ✅ Access via `config.core.schemas`, not separate field

### Data Polling
- ✅ Return empty `Vec` when no data (non-blocking)
- ✅ Use appropriate batch sizes (100-1000 records)
- ✅ Handle rate limits gracefully
- ✅ Log warnings for skipped messages
- ✅ Add attributes for routing metadata

### Error Handling
- ✅ `Retryable` for temporary failures
- ✅ `Fatal` for permanent failures
- ✅ `Ok(vec![])` for no data or skippable errors
- ✅ Log detailed context for debugging

### Performance
- ✅ Batch data retrieval when possible
- ✅ Avoid blocking operations in `poll()`
- ✅ Use connection pooling
- ✅ Configure appropriate poll intervals
- ✅ Monitor external system rate limits

---

## Common Patterns

### Pull-Based Polling
Query external system with cursor/offset tracking. Suitable for databases, REST APIs.

### Push-Based Subscription
External system pushes data to in-memory buffer. Drain buffer in `poll()`. Suitable for MQTT, webhooks.

### Stream-Based
Continuous stream reader. Read batches in `poll()`. Suitable for Kafka, message queues.

### Topic Routing
Map external identifiers to Danube topics using pattern matching, lookup tables, or configuration mappings.

---

## Testing

### Local Development
1. Start Danube: `docker-compose up -d`
2. Set config: `export CONNECTOR_CONFIG_PATH=./config/connector.toml`
3. Run: `cargo run`
4. Verify: `danube-cli consume --topic /your/topic`

### Unit Tests
- Test message transformation logic
- Test pattern matching/routing
- Mock external system for isolated testing
- Validate configuration parsing

---

## Summary

**You implement:**
- `SourceConnector` trait (3 required, 3 optional methods)
- Configuration with flattened `ConnectorConfig`
- External system integration
- Message transformation to `SourceRecord`
- Topic routing logic

**Runtime handles:**
- Danube connection
- Schema validation & serialization
- Message publishing & retries
- Lifecycle & monitoring
- Metrics & health

**Remember:** Configuration uses flattened `ConnectorConfig`. Access schemas via `config.core.schemas`. Never duplicate schema fields. Focus on external system integration and data transformation - runtime handles Danube operations.

