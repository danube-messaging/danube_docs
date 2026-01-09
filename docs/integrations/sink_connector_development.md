# Sink Connector Development Guide

**Build sink connectors to export data from Danube to any external system**

---

## Overview

A **sink connector** exports data from Danube topics to external systems. The connector consumes messages, transforms data, and writes to the target system with batching and error handling. The runtime handles consumption, schema deserialization, and delivery guarantees.

**Examples:** Danube→Delta Lake, Danube→ClickHouse, Danube→Vector DB, Danube→HTTP API

---

## Core Concepts

### Division of Responsibilities

| **You Handle** | **Runtime Handles** |
|----------------|---------------------|
| Connect to external system | Connect to Danube broker |
| Transform `SinkRecord` data | Consume from Danube topics |
| Write to target system | Schema deserialization & validation |
| Batch operations | Message buffering & delivery |
| Error handling per system | Lifecycle & health monitoring |
| Flush buffered data | Metrics & observability |

**Key insight:** You receive typed `serde_json::Value` data already deserialized by the runtime. No manual schema operations needed.

---

## SinkConnector Trait

```rust
#[async_trait]
pub trait SinkConnector: Send + Sync {
    async fn initialize(&mut self, config: ConnectorConfig) -> ConnectorResult<()>;
    async fn consumer_configs(&self) -> ConnectorResult<Vec<ConsumerConfig>>;
    async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()>;
    
    // Optional methods with defaults
    async fn process_batch(&mut self, records: Vec<SinkRecord>) -> ConnectorResult<()>;
    async fn shutdown(&mut self) -> ConnectorResult<()> { Ok(()) }
    async fn health_check(&self) -> ConnectorResult<()> { Ok(()) }
}
```

**Required:** `initialize`, `consumer_configs`, `process`  
**Optional but recommended:** `process_batch` (for performance)  
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

- ✅ Sink connectors **do not** use `config.core.schemas`
- ✅ Schema validation happens on the **producer side** (source connectors)
- ✅ You receive already-deserialized, validated data
- ✅ Schema info available via `record.schema()` if message has schema metadata

### TOML Structure

```toml
# At root level (flattened from ConnectorConfig)
danube_service_url = "http://danube-broker:6650"
connector_name = "my-sink"

# No [[schemas]] section - sinks consume, don't validate

# Connector-specific section
[my_sink]
target_url = "http://database.example.com"
connection_pool_size = 10

# Topic mappings with subscription details
[[my_sink.topic_mappings]]
topic = "/events/users"
subscription = "sink-group-1"
subscription_type = "Shared"
target_table = "users"
expected_schema_subject = "user-events-v1"  # Optional: verify schema
batch_size = 100
flush_interval_ms = 1000

[[my_sink.topic_mappings]]
topic = "/events/orders"
subscription = "sink-group-1"
subscription_type = "Shared"
target_table = "orders"
batch_size = 500
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

**Not overridable:** Topic mappings, subscription settings (must be in TOML)

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
- Store client in struct for later use in `process()`

**Error handling:**

- Return `ConnectorError::Initialization` for connection failures
- Log detailed error context
- Fail fast - don't retry in `initialize()`

**Examples:**

- **Database:** Create connection pool, validate table schema, test insert
- **HTTP API:** Set up client with auth headers, test endpoint availability
- **Object Storage:** Configure S3/GCS client, verify bucket access
- **Search Engine:** Connect, create index if needed, validate mappings

---

### 2. Define Consumer Configs

**`async fn consumer_configs(&self) -> Vec<ConsumerConfig>`**

**Purpose:** Tell the runtime which Danube topics to consume from and how.

**What to do:**

- Map your topic configuration to Danube consumer configs
- For each source topic, create a `ConsumerConfig`
- Specify subscription type (Shared, Exclusive, Failover)
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
- **Failover:** Active-passive, automatic failover

**Called once** at startup after `initialize()`. Runtime creates consumers based on these configs.

---

### 3. Process Records

**`async fn process(&mut self, record: SinkRecord)`**

**Purpose:** Transform a single message and write to external system.

**Called:** For each message consumed from Danube (or use `process_batch` instead)

**What to do:**

1. **Extract data** - Already deserialized by runtime
2. **Access metadata** - Topic, offset, timestamp, attributes
3. **Transform** - Convert to target system format
4. **Write** - Send to external system
5. **Handle errors** - Return appropriate error type

**SinkRecord API:**

```rust
let payload = record.payload();           // &serde_json::Value
let topic = record.topic();               // &str
let offset = record.offset();             // u64
let timestamp = record.publish_time();    // u64 (microseconds)
let attributes = record.attributes();     // &HashMap<String, String>

// Check schema info if message has schema
if let Some(schema) = record.schema() {
    // schema.subject, schema.version, schema.schema_type
}

// Deserialize to struct (type-safe)
let event: MyStruct = record.as_type()?;
```

**Runtime provides:**

- Pre-deserialized JSON data
- Schema validation already done
- Type-safe access methods
- Metadata extraction

---

### 4. Batch Processing (Recommended)

**`async fn process_batch(&mut self, records: Vec<SinkRecord>)`**

**Purpose:** Process multiple records together for better throughput.

**Why batching:**

- 10-100x better performance for bulk operations
- Reduced network round trips
- More efficient resource usage
- Lower latency at scale

**What to do:**

1. **Group by target** - If multi-topic, group by destination table/collection
2. **Transform batch** - Convert all records to target format
3. **Bulk write** - Single operation for entire batch
4. **Handle partial failures** - Some systems support partial success

**Batch triggers:**

- **Size:** `batch_size` records buffered (e.g., 100)
- **Time:** `flush_interval_ms` elapsed (e.g., 1000ms)
- **Shutdown:** Flush remaining records

**Performance guidance:**

- High throughput: 500-1000 records per batch
- Low latency: 10-50 records per batch
- Balanced: 100 records per batch

**Per-topic configuration:**
Different topics can have different batch sizes based on workload characteristics.

---

### 5. Error Handling

**Error types guide runtime behavior:**

| Error Type | When to Use | Runtime Action |
|------------|-------------|----------------|
| `Retryable` | Temporary failures (rate limit, connection) | Retry with backoff |
| `Fatal` | Permanent failures (auth, bad config) | Shutdown connector |
| `Ok(())` | Success or skippable errors | Acknowledge message |

**Mechanism:**

- **Temporary errors:** Rate limits, timeouts, transient failures → `Retryable`
- **Invalid data:** Log warning, skip record → `Ok()` (acknowledge anyway)
- **Fatal errors:** Auth failure, connection lost → `Fatal` (stops connector)

**The runtime:**

- Retries `Retryable` errors with exponential backoff
- Stops connector on `Fatal` errors
- Acknowledges and continues on `Ok(())`

**Partial batch failures:**
For systems that support it, process successful records and retry only failed ones.

---

### 6. Graceful Shutdown

**`async fn shutdown(&mut self)`**

**Purpose:** Clean up resources and flush pending data before stopping.

**What to do:**

- Flush any buffered records (if using internal batching)
- Complete any in-flight writes
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
[[my_sink.topic_mappings]]
topic = "/events/users"
expected_schema_subject = "user-events-v1"  # Verify schema subject
```

**What happens:**

- Runtime checks if message schema subject matches
- Mismatch results in error (configurable behavior)
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

Each topic mapping is independent:

- Different target entities (tables, collections, indexes)
- Different batch sizes and flush intervals
- Different transformations
- Different schema expectations

### Mechanism

1. **Store topic → config mapping** in connector
2. **Lookup configuration** for each record's topic
3. **Apply topic-specific logic** for transformation
4. **Route to target entity** (table, collection, etc.)

### Configuration Example

```toml
[[my_sink.topic_mappings]]
topic = "/events/clicks"
target_table = "clicks"
batch_size = 1000  # High volume

[[my_sink.topic_mappings]]
topic = "/events/orders"
target_table = "orders"
batch_size = 100   # Lower latency
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
    let runtime = SinkRuntime::new(connector, config.core).await?;
    runtime.run().await
}
```

**Runtime handles:**

- Danube connection
- Consumer creation with subscription configs
- Message consumption and buffering
- Schema fetching and deserialization
- Calling `process()` or `process_batch()`
- Retries and error handling
- Metrics and health monitoring
- Graceful shutdown on signals

---

## Best Practices

### Configuration

- ✅ Use flattened `ConnectorConfig` - no schemas field needed
- ✅ Keep secrets in environment variables, not TOML
- ✅ Validate configuration at startup
- ✅ Provide per-topic batch configuration

### Data Processing

- ✅ Use `process_batch()` for performance (10-100x faster)
- ✅ Group records by target entity in multi-topic scenarios
- ✅ Make metadata addition optional (configurable)
- ✅ Handle missing fields gracefully

### Batching

- ✅ Configure both size and time triggers
- ✅ Adjust per-topic for different workloads
- ✅ Flush on shutdown to avoid data loss
- ✅ Document batch size recommendations

### Error Handling

- ✅ `Retryable` for temporary failures (rate limits, network)
- ✅ `Fatal` for permanent failures (auth, config)
- ✅ `Ok(())` for success or skippable data issues
- ✅ Log detailed error context

### Subscription Types

- ✅ Use `Shared` for scalable parallel processing (default)
- ✅ Use `Exclusive` only when ordering is critical
- ✅ Use `Failover` for ordered + high availability
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

Buffer records until batch size/timeout reached, then bulk write. Best for throughput.

### Grouped Multi-Topic Write

Group records by target entity, write each group as batch. Efficient for multi-topic routing.

### Enriched Write

Add metadata from Danube (topic, offset, timestamp) to records before writing. Useful for auditing.

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

- `SinkConnector` trait (3 required, 2-3 optional methods)
- Configuration with flattened `ConnectorConfig` (no schemas!)
- External system integration (connection, writes, batching)
- Message transformation from `SinkRecord` to target format
- Topic routing and multi-topic handling

**Runtime handles:**

- Danube connection & consumption
- Schema fetching & deserialization
- Message buffering & delivery
- Calling your `process` methods
- Retries & error handling
- Lifecycle & monitoring
- Metrics & health

**Remember:**

- Configuration uses flattened `ConnectorConfig` without schemas field
- Schemas already validated by producers - you receive typed data
- `expected_schema_subject` is optional verification, not validation
- Use `process_batch()` for best performance
- Focus on external system integration and data transformation
