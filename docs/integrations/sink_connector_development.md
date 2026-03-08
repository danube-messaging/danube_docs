# Sink Connector Development Guide

**Build sink connectors to export data from Danube to any external system**

---

## Overview

A **sink connector** exports data from Danube topics to external systems. The connector consumes typed `SinkRecord` values, transforms them, and writes them to the target system. The runtime owns Danube consumption, buffering, batch flush timing, schema-aware deserialization, acknowledgments, health scheduling, and metrics export.

**Examples:** Danube→Delta Lake, Danube→ClickHouse, Danube→Vector DB, Danube→HTTP API

---

## Core Concepts

### Division of Responsibilities

| **You Handle** | **Runtime Handles** |
|----------------|---------------------|
| Connect to external system | Connect to Danube broker |
| Transform `SinkRecord` data | Consume from Danube topics |
| Write to target system | Schema deserialization & expected-subject checks |
| Connector-specific routing/grouping | Message buffering, batching, and acknowledgments |
| Error classification for target failures | Lifecycle, health scheduling, and retry policy |
| External-system cleanup | Metrics export and observability |

**Key insight:** You receive typed `serde_json::Value` data already deserialized by the runtime. No manual schema operations needed.

---

## SinkConnector Trait

```rust
#[async_trait]
pub trait SinkConnector: Send + Sync {
    async fn initialize(&mut self, config: ConnectorConfig) -> ConnectorResult<()>;
    async fn consumer_configs(&self) -> ConnectorResult<Vec<ConsumerConfig>>;
    async fn process_batch(&mut self, records: Vec<SinkRecord>) -> ConnectorResult<()>;

    async fn shutdown(&mut self) -> ConnectorResult<()> { Ok(()) }
    async fn health_check(&self) -> ConnectorResult<()> { Ok(()) }
}
```

**Required:** `initialize`, `consumer_configs`, `process_batch`  
**Optional:** `shutdown`, `health_check`

---

## Configuration Architecture

### Unified Configuration Pattern

Use a **single configuration struct** combining core Danube settings with connector-specific settings:

```rust
#[derive(Deserialize)]
pub struct MySinkConfig {
    #[serde(flatten)]
    pub core: ConnectorConfig,  // Danube settings (no schemas for sinks!)
    
    pub my_sink: MySettings,    // Connector-specific
}
```

**Critical for Sink Connectors:**

- ✅ Sink connectors typically leave `config.core.schemas` empty
- ✅ Runtime deserializes payloads before your connector sees them
- ✅ Per-route `expected_schema_subject` can enforce an expected contract at consume time
- ✅ Schema info remains available through `record.schema()` and `record.context()`

### TOML Structure

```toml
# At root level (flattened from ConnectorConfig)
danube_service_url = "http://danube-broker:6650"
connector_name = "my-sink"

[processing]
batch_size = 500
batch_timeout_ms = 1000
metrics_port = 9090

# Connector-specific section
[my_sink]
target_url = "http://database.example.com"
connection_pool_size = 10

# Routes with subscription details
[[my_sink.routes]]
from = "/events/users"
subscription = "sink-group-1"
subscription_type = "Shared"
to = "users"
expected_schema_subject = "user-events-v1"  # Optional: verify schema

[[my_sink.routes]]
from = "/events/orders"
subscription = "sink-group-1"
subscription_type = "Shared"
to = "orders"
```

### Configuration Loading

**Mechanism:**

1. Load TOML file from `CONNECTOR_CONFIG_PATH` environment variable
2. Apply environment variable overrides (secrets only)
3. Validate configuration
4. Pass to connector and runtime

**Environment overrides:**

- Core: `DANUBE_SERVICE_URL`, `CONNECTOR_NAME`
- Secrets: Database passwords, API keys, credentials

**Not overridable:** `routes`, `from`, `to`, subscription settings, and shared processing behavior (must stay in TOML)

---

## Implementation Steps

### 1. Initialize Connection

**`async fn initialize(&mut self, config: ConnectorConfig)`**

**Purpose:** Establish connection to external system and prepare for writing.

**What to do:**

- Create client/connection to target system
- Configure authentication (credentials, API tokens, certificates)
- Set up connection pools for parallel writes
- Validate write permissions with test write/query
- Create tables/collections/indexes if needed
- Store client in struct for later use in `process_batch()`

**Error handling:**

- Return `ConnectorError::config` for configuration mistakes
- Return `ConnectorError::fatal` for connection or setup failures
- Log detailed error context
- Fail fast - don't retry in `initialize()`

**Examples:**

- **Database:** Create connection pool, validate table schema, test insert
- **HTTP API:** Set up client with auth headers, test endpoint availability
- **Object Storage:** Configure S3/GCS client, verify bucket access
- **Search Engine:** Connect, create index if needed, validate mappings

---

### 2. Define Consumer Configs

**`async fn consumer_configs(&self) -> ConnectorResult<Vec<ConsumerConfig>>`**

**Purpose:** Tell the runtime which Danube topics to consume from and how.

**What to do:**

- Map your topic configuration to Danube consumer configs
- For each consumed topic or route, create a `ConsumerConfig`
- Specify subscription type (Shared, Exclusive, FailOver)
- Set consumer name and subscription name
- Optionally specify expected schema subject for validation

**Consumer config structure:**

```rust
ConsumerConfig {
    topic: "/events/users",
    consumer_name: "my-sink-1",
    subscription: "my-sink-group",
    subscription_type: SubscriptionType::Shared,
    expected_schema_subject: Some("user-events-v1"),  // Optional
}
```

**Subscription types:**

- **Shared:** Load balancing across multiple consumers (most common)
- **Exclusive:** Single consumer, ordered processing
- **FailOver:** Active-passive, automatic failover

**Called once** at startup after `initialize()`. Runtime creates consumers based on these configs.

---

### 3. Process Records

**`async fn process_batch(&mut self, records: Vec<SinkRecord>)`**

**Purpose:** Transform a runtime-managed batch and write it to the external system.

**Called:** When runtime buffering reaches the configured batch size or batch timeout.

**What to do:**

1. **Inspect the batch** - Records are already deserialized by the runtime
2. **Group if needed** - Group by route, table, collection, or target entity
3. **Transform** - Convert records into the external system's write model
4. **Bulk write** - Execute the connector's external write path
5. **Handle partials intentionally** - Skip bad records inside the batch if that is your policy
6. **Return appropriate error type** - Retryable only for truly transient batch failures

**SinkRecord API:**

```rust
let payload = record.payload();           // &serde_json::Value
let topic = record.topic();               // &str
let timestamp = record.publish_time();    // u64 (microseconds)
let attributes = record.attributes();     // &HashMap<String, String>
let context = record.context();           // generic routing + schema context

// Check schema info if message has schema
if let Some(schema) = record.schema() {
    // schema.subject, schema.version, schema.schema_type
}

// Deserialize to struct (type-safe)
let event: MyStruct = record.as_type()?;
```

**Runtime provides:**

- Pre-deserialized JSON data
- Expected-schema checks when configured
- Type-safe access methods
- Logical topic metadata and generic record context

---

### 4. Runtime-Managed Batching

The sink runtime manages buffering and flush timing using the shared root `[processing]` settings.

**Controls:**

- `processing.batch_size`
- `processing.batch_timeout_ms`

**What this means for connector authors:**

- Do **not** add connector-owned batch size or flush timer settings unless the external platform truly requires a second, platform-specific layer
- Treat `process_batch()` as the primary sink write path
- If your connector supports multiple routes, group the incoming batch by `record.topic()` or by route target inside `process_batch()`
- If you need lower latency or higher throughput, tune the shared `[processing]` block rather than inventing connector-local batching knobs

**Example:**

```toml
[processing]
batch_size = 1000
batch_timeout_ms = 1000
```

---

### 5. Error Handling

**Error types guide runtime behavior:**

| Error Type | When to Use | Runtime Action |
|------------|-------------|----------------|
| `Retryable` | Temporary failures (rate limit, connection) | Retry with backoff |
| `Fatal` | Permanent failures (auth, bad config) | Shutdown connector |
| `InvalidData` | Batch content is malformed or unusable | Non-retryable failure |
| `Ok(())` | Batch succeeded | Acknowledge message |

**Important:** The runtime only retries `Retryable` errors. If you return `InvalidData`, that batch is treated as a non-retryable failure.

**Recommended pattern:**

- Handle per-record skippable bad data **inside** `process_batch()` when possible
- Return `Retryable` only when the whole batch should be retried
- Return `Fatal` when continuing would be unsafe or pointless

**Partial batch failures:**
For systems that support partial success, handle the split inside the connector and only return an error for the portion that truly failed your chosen policy.

---

### 6. Graceful Shutdown

**`async fn shutdown(&mut self)`**

**Purpose:** Clean up resources and flush pending data before stopping.

For most connectors, this means finishing in-flight external work and closing clients cleanly. The runtime already owns Danube-side buffering.

**What to do:**

- Complete any in-flight external writes
- Close connections to external system
- Save any state if needed
- Log shutdown completion

**Runtime guarantees:**

- `shutdown()` called on SIGTERM/SIGINT
- No new messages delivered after shutdown starts
- Time to complete graceful shutdown

---

## Schema Handling in Sink Connectors

### How Schemas Work

**For sink connectors, schemas are handled differently than sources:**

1. **Producer side** (source connector or app) validates and attaches schema
2. **Message stored** in Danube with schema ID
3. **Runtime fetches schema** from registry when consuming
4. **Runtime deserializes** payload based on schema type
5. **You receive** validated, typed `serde_json::Value`

**You never:**

- ❌ Validate schemas (producer already did)
- ❌ Deserialize raw bytes (runtime already did)
- ❌ Fetch schemas from registry (runtime caches them)
- ❌ Configure schemas in `ConnectorConfig` (no `[[schemas]]` section)

**You can:**

- ✅ Access schema metadata via `record.schema()`
- ✅ Specify `expected_schema_subject` for validation
- ✅ Use different logic based on schema version
- ✅ Access pre-validated typed data

### Expected Schema Subject

Optional verification that messages have expected schema:

```toml
[[my_sink.routes]]
from = "/events/users"
expected_schema_subject = "user-events-v1"  # Verify schema subject
```

**What happens:**

- Runtime checks if message schema subject matches
- Mismatch results in a non-retryable consume-side failure for that batch
- Useful for ensuring data contracts

**When to use:**

- ✅ Strict data contracts required
- ✅ Multiple producers on same topic
- ✅ Critical data pipelines

**When to skip:**

- Schema evolution in progress
- Flexible data formats
- Mixed content topics

---

## Subscription Types

### Shared (Most Common)

**Behavior:** Messages load-balanced across all active consumers

**Use cases:**

- High-volume parallel processing
- Horizontal scaling
- Order doesn't matter within topic

**Example:** Processing events where each event is independent

### Exclusive

**Behavior:** Only one consumer receives messages, strictly ordered

**Use cases:**

- Order-dependent processing
- Sequential operations
- Single-threaded requirements

**Example:** Bank transactions that must be processed in order

### Failover

**Behavior:** Active-passive setup with automatic failover

**Use cases:**

- Ordered processing with high availability
- Standby consumer for reliability
- Graceful failover

**Example:** Critical pipeline with backup consumer

---

## Multi-Topic Routing

### Pattern

Each route is independent:

- Different target entities (tables, collections, indexes)
- Different transformations
- Different schema expectations

### Mechanism

1. **Store topic → config mapping** in connector
2. **Lookup configuration** for each record's topic
3. **Apply topic-specific logic** for transformation
4. **Route to target entity** (table, collection, etc.)

### Configuration Example

```toml
[[my_sink.routes]]
from = "/events/clicks"
subscription = "analytics-clicks"
to = "clicks"

[[my_sink.routes]]
from = "/events/orders"
subscription = "analytics-orders"
to = "orders"
```

---

## Main Entry Point

```rust
#[tokio::main]
async fn main() -> ConnectorResult<()> {
    // 1. Initialize logging
    tracing_subscriber::fmt::init();
    
    // 2. Load configuration
    let config = MySinkConfig::load()?;
    
    // 3. Create connector with settings
    let connector = MySinkConnector::new(config.my_sink)?;
    
    // 4. Create and run runtime (handles everything else)
    let mut runtime = SinkRuntime::new(connector, config.core).await?;
    runtime.run().await
}
```

**Runtime handles:**

- Danube connection
- Consumer creation with subscription configs
- Message consumption and buffering
- Schema fetching and deserialization
- Calling `process_batch()`
- Retries and error handling
- Lifecycle & monitoring
- Metrics & health

---

## Best Practices

### Configuration

- ✅ Use flattened `ConnectorConfig` at the root
- ✅ Keep secrets in environment variables, not TOML
- ✅ Validate configuration at startup
- ✅ Keep route structure consistent with `routes`, `from`, and `to`

### Data Processing

- ✅ Use `process_batch()` for performance (10-100x faster)
- ✅ Group records by target entity in multi-topic scenarios
- ✅ Make metadata addition optional (configurable)
- ✅ Handle missing fields gracefully

### Runtime Settings

- ✅ Tune `[processing].batch_size` and `[processing].batch_timeout_ms`
- ✅ Use `metrics_port` and health-check settings from shared config
- ✅ Avoid duplicate connector-local batching knobs unless platform-specific
- ✅ Document shared runtime tuning instead of connector-owned flush policy

### Error Handling

- ✅ `Retryable` for temporary failures (rate limits, network)
- ✅ `Fatal` for permanent failures (auth, config)
- ✅ Return `Ok(())` only after the batch succeeded or any skippable records were handled inside the connector
- ✅ Log detailed error context

### Subscription Types

- ✅ Use `Shared` for scalable parallel processing (default)
- ✅ Use `Exclusive` only when ordering is critical
- ✅ Use `FailOver` for ordered + high availability
- ✅ Document ordering requirements

### Performance

- ✅ Use connection pooling for concurrent writes
- ✅ Batch operations whenever possible
- ✅ Configure appropriate timeouts
- ✅ Monitor backpressure and lag

---

## Common Patterns

### Direct Write

Receive data, transform, write directly. Simplest approach for low-volume or low-latency requirements.

### Buffered Batch Write

Let the runtime buffer records until batch size/timeout is reached, then bulk write in `process_batch()`. Best for throughput.

### Grouped Multi-Topic Write

Group records by target entity, write each group as batch. Efficient for multi-topic routing.

### Enriched Write

Add metadata from Danube (topic, publish time, producer, attributes) to records before writing. Useful for auditing.

### Upsert Pattern

Use record attributes or payload fields to determine insert vs. update. Common for CDC scenarios.

---

## Testing

### Local Development

1. Start Danube: `docker-compose up -d`
2. Set config: `export CONNECTOR_CONFIG_PATH=./config/connector.toml`
3. Run: `cargo run`
4. Produce: `danube-cli produce --topic /events/test --message '{"test":"data"}'`
5. Verify: Check target system for data

### Unit Tests

- Test transformation logic with sample `SinkRecord`s
- Test topic routing and configuration lookup
- Mock external system for isolated testing
- Validate error handling paths

### Integration Tests

- Full stack: Danube + Connector + Target system
- Produce messages with schemas
- Verify data in target system
- Test batching behavior
- Test error scenarios

---

## Summary

**You implement:**

- `SinkConnector` trait (3 required, 2 optional methods)
- Configuration with flattened `ConnectorConfig`
- External system integration (connection and writes)
- Message transformation from `SinkRecord` to target format
- Topic routing and multi-topic handling

**Runtime handles:**

- Danube connection & consumption
- Schema fetching & deserialization
- Message buffering, batching, and delivery
- Calling your `process_batch` method
- Retries & error handling
- Lifecycle & monitoring
- Metrics & health

**Remember:**

- Configuration uses flattened `ConnectorConfig` plus connector-specific `routes`
- Schemas are handled by the runtime - you receive typed data
- `expected_schema_subject` is optional verification, not validation
- Use `process_batch()` for best performance
- Focus on external system integration and data transformation
