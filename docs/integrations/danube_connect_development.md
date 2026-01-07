# Building Danube Connectors

**Create custom connectors to integrate Danube with any external system**

---

## Overview

Building a connector involves several steps:

1. **Implement the trait** - Either `SourceConnector` or `SinkConnector`
2. **Handle external system communication** - Connect, read/write, handle errors
3. **Transform data** - Convert external formats to Danube messages
4. **Create the configuration** - Define Danube topics and external system settings
5. **Let the SDK do the rest** - Lifecycle, Danube communication, retries, metrics

---

## Decision: Source or Sink?

### Source Connector (Import into Danube)

**When to use:**

- You want to **bring data into** Danube from an external system
- External system generates or stores data

**Examples:** MQTT→Danube, PostgreSQL CDC→Danube, HTTP webhooks→Danube, Kafka→Danube

### Sink Connector (Export from Danube)

**When to use:**

- You want to **send data from** Danube to an external system
- External system consumes or stores data

**Examples:** Danube→Delta Lake, Danube→ClickHouse, Danube→HTTP API, Danube→Vector DB

---

## Source Connector Development

### Trait Signature

Implement the `SourceConnector` trait from `danube-connect-core`:

```rust
use danube_connect_core::{SourceConnector, SourceRecord, ProducerConfig, ConnectorResult};
use async_trait::async_trait;

#[async_trait]
pub trait SourceConnector: Send + Sync {
    /// Initialize external system connection
    /// Called once at startup
    async fn initialize(&mut self, config: ConnectorConfig) -> ConnectorResult<()>;
    
    /// Define all destination Danube topics and their configurations
    /// Called once at startup
    async fn producer_configs(&self) -> ConnectorResult<Vec<ProducerConfig>>;
    
    /// Poll external system for new data
    /// Called repeatedly at configured interval (e.g., every 100ms)
    /// Return empty Vec if no data available (non-blocking)
    async fn poll(&mut self) -> ConnectorResult<Vec<SourceRecord>>;
    
    /// Optional: Commit offsets after successful publish to Danube
    /// Only needed if you track offsets for exactly-once semantics
    async fn commit(&mut self, offsets: Vec<Offset>) -> ConnectorResult<()> {
        Ok(())  // Default: no-op
    }
    
    /// Optional: Cleanup on graceful shutdown
    async fn shutdown(&mut self) -> ConnectorResult<()> {
        Ok(())  // Default: no-op
    }
    
    /// Optional: Health check for external system connectivity
    async fn health_check(&self) -> ConnectorResult<()> {
        Ok(())  // Default: always healthy
    }
}
```

---

### Your Responsibilities (Third-Party Integration)

#### 1. **External System Connection**

**In `initialize()`:**

- Create client connection to external system
- Configure authentication (API keys, tokens, certificates)
- Set up connection pools if needed
- Validate connectivity

**Example considerations:**

- MQTT: Connect to broker, authenticate, configure QoS
- HTTP API: Set up HTTP client with retry middleware, auth headers
- Database: Establish connection pool, test query
- Kafka: Configure consumer group, deserializers

#### 2. **Data Polling/Listening**

**In `poll()`:**

- Read data from external system
- Transform external format to `SourceRecord`
- Route records to appropriate Danube topics
- Handle "no data available" gracefully (return empty vec)

**Example patterns:**

- **Pull-based (PostgreSQL CDC):** Query for new rows, track last offset
- **Push-based (HTTP webhook):** Return buffered requests from in-memory queue
- **Subscribe-based (MQTT):** Return messages from subscription buffer
- **Stream-based (Kafka):** Poll consumer, convert messages

#### 3. **Message Transformation**

Transform external data to Danube `SourceRecord` with typed JSON data:

```rust
use serde_json::json;

// From JSON-serializable struct (recommended)
#[derive(Serialize)]
struct SensorData {
    device_id: String,
    temperature: f64,
    timestamp: u64,
}

let data = SensorData { /* ... */ };
let record = SourceRecord::from_json(&danube_topic, &data)?
    .with_attribute("source", "mqtt")
    .with_attribute("device_type", "sensor")
    .with_key(&data.device_id);  // For partitioning

// From JSON value directly
let record = SourceRecord::new(&danube_topic, json!({
    "device_id": device_id,
    "temperature": 23.5,
    "timestamp": now
})).with_attribute("source", "mqtt");

// From string (for text messages)
let record = SourceRecord::from_string(&danube_topic, &text)
    .with_attribute("content-type", "text/plain");
```

**Important:** The runtime will serialize your `serde_json::Value` payload based on the topic's configured schema type (JSON, String, Bytes, etc.). You work with typed data, not raw bytes.

#### 4. **Topic Routing**

Specify destination topics in `producer_configs()`:

```rust
async fn producer_configs(&self) -> ConnectorResult<Vec<ProducerConfig>> {
    Ok(vec![
        ProducerConfig {
            topic: "/iot/sensors".to_string(),
            partitions: 8,
            reliable_dispatch: true,  // At-least-once delivery
        },
        ProducerConfig {
            topic: "/iot/debug".to_string(),
            partitions: 1,
            reliable_dispatch: false,  // Best-effort delivery
        },
    ])
}
```

#### 5. **Error Handling**

Return appropriate error types:

```rust
async fn poll(&mut self) -> ConnectorResult<Vec<SourceRecord>> {
    match self.external_client.fetch_data().await {
        Ok(data) => {
            // Transform and return
            Ok(transform_to_records(data))
        },
        Err(e) if e.is_temporary() => {
            // Temporary network issue, rate limit, etc.
            Err(ConnectorError::Retryable(e.to_string()))
        },
        Err(e) if e.is_invalid_data() => {
            // Malformed response, skip and continue
            warn!("Skipping invalid data: {}", e);
            Ok(vec![])
        },
        Err(e) => {
            // Fatal error, shutdown needed
            Err(ConnectorError::Fatal(e.to_string()))
        }
    }
}
```

#### 6. **Offset Management (Optional)**

For exactly-once semantics, implement `commit()`:

```rust
async fn commit(&mut self, offsets: Vec<Offset>) -> ConnectorResult<()> {
    // Save offsets to external system or local storage
    self.checkpoint_store.save(offsets).await?;
    Ok(())
}
```

---

## Sink Connector Development

### Trait Signature

Implement the `SinkConnector` trait from `danube-connect-core`:

```rust
use danube_connect_core::{SinkConnector, SinkRecord, ConsumerConfig, ConnectorResult};
use async_trait::async_trait;

#[async_trait]
pub trait SinkConnector: Send + Sync {
    /// Initialize external system connection
    /// Called once at startup
    async fn initialize(&mut self, config: ConnectorConfig) -> ConnectorResult<()>;
    
    /// Define all Danube topics to consume from
    /// Called once at startup
    async fn consumer_configs(&self) -> ConnectorResult<Vec<ConsumerConfig>>;
    
    /// Process a single message from Danube
    /// Return Ok(()) to acknowledge, Err to retry or fail
    async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()>;
    
    /// Optional: Process batch for better performance
    /// Default calls process() for each record
    async fn process_batch(&mut self, records: Vec<SinkRecord>) -> ConnectorResult<()> {
        for record in records {
            self.process(record).await?;
        }
        Ok(())
    }
    
    /// Optional: Cleanup on graceful shutdown
    async fn shutdown(&mut self) -> ConnectorResult<()> {
        Ok(())
    }
    
    /// Optional: Health check for external system connectivity
    async fn health_check(&self) -> ConnectorResult<()> {
        Ok(())
    }
}
```

---

### Your Responsibilities (Third-Party Integration)

#### 1. **External System Connection**

**In `initialize()`:**

- Establish connection to database, API, or service
- Configure write settings (batch size, timeouts, retries)
- Create connection pools for parallel writes
- Validate write permissions

**Example considerations:**

- Database: Connection pool, table schema validation
- HTTP API: Client setup, auth headers, endpoint validation
- File system: Create directories, check write permissions
- Object storage: S3 client, bucket access verification

#### 2. **Message Consumption Configuration**

**In `consumer_configs()`:**

- Specify which Danube topics to consume from
- Configure subscription type (Exclusive, Shared, Failover)
- Set consumer names

```rust
async fn consumer_configs(&self) -> ConnectorResult<Vec<ConsumerConfig>> {
    Ok(vec![
        ConsumerConfig {
            topic: "/analytics/events".to_string(),
            consumer_name: "clickhouse-sink-1".to_string(),
            subscription: "clickhouse-sub".to_string(),
            subscription_type: SubscriptionType::Shared,  // Multiple consumers
        },
        ConsumerConfig {
            topic: "/analytics/metrics".to_string(),
            consumer_name: "clickhouse-sink-2".to_string(),
            subscription: "clickhouse-metrics".to_string(),
            subscription_type: SubscriptionType::Exclusive,  // Single consumer
        },
    ])
}
```

#### 3. **Message Transformation**

Transform Danube `SinkRecord` to external format.

**The runtime automatically deserializes messages based on their schema.** You receive typed `serde_json::Value` data, not raw bytes:

```rust
async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    // Payload is already deserialized by runtime based on schema
    let payload = record.payload();  // &serde_json::Value
    
    // Option 1: Access JSON value directly
    let user_id = payload["user_id"].as_str()
        .ok_or_else(|| ConnectorError::invalid_data("Missing user_id", vec![]))?;
    let action = payload["action"].as_str().unwrap_or("unknown");
    
    // Option 2: Deserialize to struct (type-safe)
    #[derive(Deserialize)]
    struct Event {
        user_id: String,
        action: String,
        timestamp: u64,
    }
    
    let event: Event = record.as_type()?;
    
    // Extract metadata
    let topic = record.topic();
    let offset = record.offset();
    let timestamp = record.publish_time();
    let attributes = record.attributes();
    
    // Check schema info (if message has schema)
    if let Some(schema) = record.schema() {
        info!("Processing with schema: {} v{}", schema.subject, schema.version);
    }
    
    // Transform to external format
    let external_row = DatabaseRow {
        user_id: event.user_id,
        action: event.action,
        source_topic: topic.to_string(),
        ingested_at: timestamp,
    };
    
    // Write to external system
    self.database.insert(external_row).await?;
    
    Ok(())
}
```

#### 4. **Batching for Performance**

Implement `process_batch()` for better throughput:

```rust
use danube_connect_core::Batcher;

struct MyConnector {
    batcher: Batcher<SinkRecord>,
    client: ExternalClient,
}

async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    // Add to batch
    self.batcher.add(record);
    
    // Flush when batch is full or timeout reached
    if self.batcher.should_flush() {
        let batch = self.batcher.flush();
        self.write_batch(batch).await?;
    }
    
    Ok(())
}

async fn write_batch(&self, records: Vec<SinkRecord>) -> ConnectorResult<()> {
    // Transform entire batch
    let rows: Vec<_> = records.iter()
        .map(|r| transform_to_row(r))
        .collect::<Result<_>>()?;
    
    // Single bulk write
    self.client.bulk_insert(rows).await?;
    
    Ok(())
}
```

#### 5. **Error Handling**

Return appropriate error types for SDK retry logic:

```rust
async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    match self.client.write(&data).await {
        Ok(_) => Ok(()),
        Err(e) if e.is_rate_limited() => {
            // Temporary, will retry with backoff
            Err(ConnectorError::Retryable("Rate limited".to_string()))
        },
        Err(e) if e.is_invalid_data() => {
            // Malformed message, skip it
            warn!("Skipping invalid record: {}", e);
            Ok(())
        },
        Err(e) => {
            // Fatal error, shutdown connector
            Err(ConnectorError::Fatal(e.to_string()))
        }
    }
}
```

#### 6. **Cleanup on Shutdown**

Flush pending data and close connections:

```rust
async fn shutdown(&mut self) -> ConnectorResult<()> {
    // Flush any buffered records
    if !self.batcher.is_empty() {
        let batch = self.batcher.flush();
        self.write_batch(batch).await?;
    }
    
    // Close connection
    self.client.disconnect().await?;
    
    Ok(())
}
```

---

## Configuration Management

### Unified Configuration Pattern

Each connector defines a **single configuration struct** combining core Danube settings with connector-specific settings:

```rust
use serde::Deserialize;
use danube_connect_core::ConnectorConfig;

#[derive(Deserialize)]
pub struct MyConnectorConfig {
    /// Core Danube settings (flattened at root level)
    #[serde(flatten)]
    pub core: ConnectorConfig,
    
    /// Connector-specific settings
    pub my_connector: MySettings,
}

#[derive(Deserialize)]
pub struct MySettings {
    pub api_url: String,
    pub api_key: Option<String>,
    pub batch_size: usize,
}

impl MyConnectorConfig {
    pub fn load() -> Result<Self> {
        // Load from TOML file
        let config_path = std::env::var("CONNECTOR_CONFIG_PATH")?;
        let mut config = Self::from_file(&config_path)?;
        
        // Apply environment variable overrides
        if let Ok(url) = std::env::var("DANUBE_SERVICE_URL") {
            config.core.danube_service_url = url;
        }
        if let Ok(key) = std::env::var("API_KEY") {
            config.my_connector.api_key = Some(key);
        }
        
        Ok(config)
    }
}
```

### Configuration File Example

```toml
# connector.toml - Single file with everything

# Core Danube settings (at root level)
danube_service_url = "http://danube-broker:6650"
connector_name = "my-connector"

# Connector-specific settings
[my_connector]
api_url = "https://api.example.com"
batch_size = 100

# Additional sections as needed
[[my_connector.topic_mappings]]
source_pattern = "sensors/#"
destination_topic = "/iot/sensors"
partitions = 8
```

**See:** [Configuration Guide](https://github.com/danube-messaging/danube-connect/blob/main/info/unified_configuration_guide.md) for complete details

---

## Schema Registry Configuration

**Critical for production:** Configure schemas for your connector topics to enable automatic serialization/deserialization and schema validation.

### Why Use Schemas?

**Without schemas:**

- Manual serialization/deserialization in every connector
- No data validation or type safety
- Schema drift between producers and consumers
- Difficult schema evolution

**With schemas:**

- ✅ Automatic serialization/deserialization by runtime
- ✅ Schema validation at message ingestion
- ✅ Type safety and data contracts
- ✅ Managed schema evolution with versions
- ✅ Your connector works with typed `serde_json::Value`, not raw bytes

### Schema Configuration Format

Add schema mappings to your `ConnectorConfig`:

```toml
# Core Danube settings
danube_service_url = "http://danube-broker:6650"
connector_name = "my-connector"

# Schema registry configuration
[[schemas]]
topic = "/events/users"              # Danube topic
subject = "user-events-schema"       # Schema registry subject
schema_type = "json_schema"          # Schema type
schema_file = "schemas/user.json"    # Path to schema file (relative to config)
version_strategy = "latest"          # Version strategy

[[schemas]]
topic = "/iot/sensors"
subject = "sensor-data"
schema_type = "json_schema"
schema_file = "schemas/sensor.json"
version_strategy = { pinned = 2 }    # Pin to specific version

[[schemas]]
topic = "/raw/telemetry"
subject = "telemetry-bytes"
schema_type = "bytes"                # Binary data (no schema file needed)
```

### Schema Types

| Type | Description | When to Use | Schema File Required |
|------|-------------|-------------|---------------------|
| `json_schema` | JSON Schema validation | Structured data, events | ✅ Yes |
| `string` | UTF-8 text | Logs, plain text | ❌ No |
| `bytes` | Binary data | Images, binary formats | ❌ No |
| `number` | Numeric values | Metrics, counters | ❌ No |
| `avro` | Apache Avro | High-performance serialization | ✅ Yes (coming soon) |
| `protobuf` | Protocol Buffers | gRPC, microservices | ✅ Yes (coming soon) |

### Version Strategies

Control how your connector handles schema versions:

```toml
# 1. Latest (default) - Always use the latest version
version_strategy = "latest"

# 2. Pinned - Lock to specific version
version_strategy = { pinned = 3 }

# 3. Minimum - Use version >= specified version
version_strategy = { minimum = 2 }
```

**Use cases:**

- **Latest:** Development, flexible evolution
- **Pinned:** Production stability, controlled upgrades
- **Minimum:** Backward compatibility with version floor

### Schema File Examples

**JSON Schema (`schemas/user-event.json`):**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["user_id", "action", "timestamp"],
  "properties": {
    "user_id": {
      "type": "string",
      "description": "Unique user identifier"
    },
    "action": {
      "type": "string",
      "enum": ["login", "logout", "signup", "update"]
    },
    "timestamp": {
      "type": "integer",
      "description": "Unix timestamp in milliseconds"
    },
    "metadata": {
      "type": "object",
      "additionalProperties": true
    }
  }
}
```

### How It Works in Your Connector

#### For Source Connectors (Publishing)

```rust
// You create records with typed JSON data
let record = SourceRecord::from_json("/events/users", &UserEvent {
    user_id: "123".to_string(),
    action: "login".to_string(),
    timestamp: now(),
})?;

// Runtime automatically:
// 1. Serializes payload based on /events/users topic's schema_type
// 2. Validates against JSON schema (if configured)
// 3. Registers schema with registry (if new)
// 4. Attaches schema ID to message
// 5. Sends to Danube
```

#### For Sink Connectors (Consuming)

```rust
async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    // Runtime already:
    // 1. Fetched schema from registry (cached)
    // 2. Deserialized payload based on schema_type
    // 3. Validated against schema
    
    // You receive typed data ready to use
    let payload = record.payload();  // &serde_json::Value
    let event: UserEvent = record.as_type()?;
    
    // Check which schema was used
    if let Some(schema) = record.schema() {
        println!("Schema: {} v{} ({})", 
                 schema.subject, schema.version, schema.schema_type);
    }
    
    Ok(())
}
```

**The runtime automatically connects to the schema registry via the Danube client** - no additional configuration needed in your connector!

### Project Structure with Schemas

```bash
connectors/my-connector/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── connector.rs
│   └── config.rs
├── config/
│   └── connector.toml
├── schemas/                    # Schema files
│   ├── user-event.json
│   ├── sensor-data.json
│   └── README.md
├── Dockerfile
└── README.md
```

### Schema Evolution Best Practices

1. **Backward compatible changes** (safe):
   - Add optional fields
   - Remove required fields (make them optional first)
   - Widen validation (e.g., increase max length)

2. **Breaking changes** (requires version bump):
   - Remove fields
   - Change field types
   - Add required fields
   - Narrow validation

3. **Version strategy recommendations:**
   - **Development:** Use `latest` for flexibility
   - **Staging:** Use `pinned` to specific version for testing
   - **Production:** Use `pinned` for stability, upgrade deliberately

### Testing with Schemas

**Start Danube with schema registry:**

```bash
cd danube-connect/docker
docker-compose up -d  # Includes schema registry
```

**Register your schema manually (for testing):**

```bash
# Using danube-cli (if available)
danube-admin-cli schema register \
  --subject user-events-schema \
  --file schemas/user-event.json \
  --type json_schema
```

**Or let the runtime register automatically** when your connector starts!

---

### Minimal main.rs

```rust
use danube_connect_core::{SourceRuntime, ConnectorResult};

#[tokio::main]
async fn main() -> ConnectorResult<()> {
    // Load configuration
    let config = MyConnectorConfig::load()?;
    
    // Create connector
    let connector = MyConnector::new(config.my_connector)?;
    
    // Run with SDK runtime (handles everything else)
    let runtime = SourceRuntime::new(connector, config.core).await?;
    runtime.run().await
}
```

---

## Testing & Examples

### Testing Your Connector

**Step 1: Start local Danube cluster**

```bash
cd danube-connect/docker
docker-compose up -d
```

**Step 2: Run your connector**

```bash
export CONNECTOR_CONFIG_PATH=./connector.toml
cargo run
```

**Step 3: Verify data flow**

For **source connectors** - consume from Danube:

```bash
danube-cli consume \
  --service-addr http://localhost:6650 \
  --topic /your/topic \
  --subscription test-sub
```

For **sink connectors** - publish to Danube:

```bash
danube-cli produce \
  --service-addr http://localhost:6650 \
  --topic /your/topic \
  --message '{"test": "data"}'
```

**Step 4: Check metrics**

```bash
curl http://localhost:9090/metrics
```

## Next Steps

- **[Architecture Guide](danube_connect_architecture.md)** - Understand the framework design
- **[Github Connector Core](https://github.com/danube-messaging/danube-connect-core)** - Connector SDK source code
- **[GitHub Connectors Repo](https://github.com/danube-messaging/danube-connectors)** - Source code and full examples
- **[API Documentation](https://docs.rs/danube-connect-core)** - SDK reference
