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

Transform external data to Danube `SourceRecord`:

```rust
// From JSON data
let record = SourceRecord::from_json(&danube_topic, &external_data)
    .with_attribute("source", "mqtt")
    .with_attribute("timestamp", &timestamp)
    .with_key(&device_id);  // For partitioning

// From raw bytes
let record = SourceRecord::new(danube_topic, payload_bytes)
    .with_attribute("content-type", "application/octet-stream");

// From string
let record = SourceRecord::from_string(&danube_topic, &text)
    .with_attribute("encoding", "utf-8");
```

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

Transform Danube `SinkRecord` to external format:

```rust
async fn process(&mut self, record: SinkRecord) -> ConnectorResult<()> {
    // Extract payload
    let payload_bytes = record.payload();  // Raw bytes
    let payload_str = record.payload_str()?;  // UTF-8 string
    let event: Event = record.payload_json()?;  // Deserialize to struct
    
    // Extract metadata
    let topic = record.topic();
    let offset = record.offset();
    let timestamp = record.publish_time();
    let attributes = record.attributes();
    
    // Transform to external format
    let external_row = DatabaseRow {
        id: event.id,
        data: event.data,
        source_topic: topic,
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

## Project Structure

```
connectors/my-connector/
├── Cargo.toml
├── src/
│   ├── main.rs          # Entry point with runtime
│   ├── connector.rs     # Trait implementation
│   ├── config.rs        # Configuration structs
│   └── client.rs        # External system client
├── config/
│   └── connector.toml   # Example configuration
├── Dockerfile
└── README.md
```

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

### Development Guides

For detailed patterns and best practices:

- **[Connector Development Guide](https://github.com/danube-messaging/danube-connect/blob/main/info/connector-development-guide.md)** - Complete implementation guide
- **[Configuration Guide](https://github.com/danube-messaging/danube-connect/blob/main/info/unified_configuration_guide.md)** - Configuration patterns
- **[Message Patterns](https://github.com/danube-messaging/danube-connect/blob/main/info/connector-message-patterns.md)** - Message transformation strategies

## Next Steps

- **[Architecture Guide](danube_connect_architecture.md)** - Understand the framework design
- **[GitHub Repository](https://github.com/danube-messaging/danube-connect)** - Source code and full examples
- **[API Documentation](https://docs.rs/danube-connect-core)** - SDK reference
- **[MQTT Example](https://github.com/danube-messaging/danube-connect/tree/main/examples/source-mqtt)** - Complete working setup
