# Schema Registry Integration

The Schema Registry provides type safety, validation, and schema evolution for your messages. This guide shows how to use it from client applications.

**Note:** Schema Registry is currently only available in the Rust client. Go client support is coming soon.

## Overview

**What you get:**

- ✅ Type-safe message serialization/deserialization
- ✅ Automatic schema validation
- ✅ Safe schema evolution with compatibility checking
- ✅ Reduced bandwidth (schema ID vs. full schema)
- ✅ Schema discovery and documentation
- ✅ Flexible version control (latest, pinned, minimum)

**Workflow:**

1. Register schema with Schema Registry
2. Link producer to schema subject (assigns to topic if first producer)
3. Send validated messages (broker enforces schema matching)
4. Consumer fetches schema and deserializes

**What Clients Can Do:**

- ✅ Register new schema versions
- ✅ Check compatibility before registering
- ✅ Assign schema subject to topic (first producer only)
- ✅ Choose schema version for producer (latest/pinned/minimum)
- ✅ Fetch schemas for validation

**What Clients Cannot Do (Admin-Only):**

- ❌ Set compatibility mode (admin CLI only)
- ❌ Change topic's schema subject (admin CLI only)
- ❌ Set topic validation policy (admin CLI only)
- ❌ Delete schema versions (admin CLI only)

---

## Schema Registry Client

### Creating the Client

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SchemaRegistryClient};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut schema_client = SchemaRegistryClient::new(&client).await?;

        Ok(())
    }
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    // Coming soon
    ```

**Note:** Schema Registry client is separate from producer/consumer clients but shares the same connection pool.

---

## Registering Schemas

### JSON Schema

=== "Rust"

    ```rust
    use danube_client::{SchemaRegistryClient, SchemaType};

    let json_schema = r#"{
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
            "timestamp": {"type": "integer"}
        },
        "required": ["user_id", "event", "timestamp"]
    }"#;

    let schema_id = schema_client
        .register_schema("user-events")
        .with_type(SchemaType::JsonSchema)
        .with_schema_data(json_schema.as_bytes())
        .with_description("User activity events v1")
        .execute()
        .await?;

    println!("✅ Registered schema ID: {}", schema_id);
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

### Avro Schema

=== "Rust"

    ```rust
    use danube_client::{SchemaRegistryClient, SchemaType};

    let avro_schema = r#"{
        "type": "record",
        "name": "UserEvent",
        "namespace": "com.example",
        "fields": [
            {"name": "user_id", "type": "string"},
            {"name": "event", "type": "string"},
            {"name": "timestamp", "type": "long"},
            {"name": "metadata", "type": ["null", "string"], "default": null}
        ]
    }"#;

    let schema_id = schema_client
        .register_schema("user-events-avro")
        .with_type(SchemaType::Avro)
        .with_schema_data(avro_schema.as_bytes())
        .execute()
        .await?;

    println!("✅ Registered Avro schema: {}", schema_id);
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

### Idempotent Registration

Registering the same schema content multiple times returns the existing schema ID:

=== "Rust"

    ```rust
    // First registration
    let id1 = schema_client
        .register_schema("events")
        .with_type(SchemaType::JsonSchema)
        .with_schema_data(schema.as_bytes())
        .execute()
        .await?;

    // Subsequent registration of same content
    let id2 = schema_client
        .register_schema("events")
        .with_type(SchemaType::JsonSchema)
        .with_schema_data(schema.as_bytes())
        .execute()
        .await?;

    assert_eq!(id1, id2);  // Same ID returned
    ```

---

## Retrieving Schemas

### Get Latest Version

=== "Rust"

    ```rust
    use danube_client::SchemaInfo;

    let schema: SchemaInfo = schema_client
        .get_latest_schema("user-events")
        .await?;

    println!("Schema ID: {}", schema.schema_id);
    println!("Subject: {}", schema.subject);
    println!("Version: {}", schema.version);
    println!("Type: {}", schema.schema_type);
    
    // Get schema definition as string (for JSON-based schemas)
    if let Some(schema_str) = schema.schema_definition_as_string() {
        println!("Schema: {}", schema_str);
    }
    ```

**Returns:** `SchemaInfo` - A user-friendly wrapper containing:

- `schema_id` - Global schema identifier
- `subject` - Schema subject name  
- `version` - Schema version number
- `schema_type` - Type (avro, json, protobuf)
- `schema_definition` - Raw bytes
- `fingerprint` - Deduplication hash

### Get Schema by ID

=== "Rust"

    ```rust
    // When consuming messages, you get schema_id in message metadata
    if let Some(schema_id) = message.schema_id {
        let schema: SchemaInfo = schema_client
            .get_schema_by_id(schema_id)
            .await?;
            
        println!("Schema subject: {}", schema.subject);
        println!("Schema version: {}", schema.version);
    }
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

### List All Versions

=== "Rust"

    ```rust
    let versions = schema_client
        .list_versions("user-events")
        .await?;

    println!("Available versions: {:?}", versions);  // e.g., [1, 2, 3]
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

---

## Compatibility Checking

### Check Before Registering

=== "Rust"

    ```rust
    use danube_client::{SchemaType, CompatibilityMode};

    let new_schema = r#"{
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
            "timestamp": {"type": "integer"},
            "email": {"type": "string"}
        },
        "required": ["user_id", "event", "timestamp"]
    }"#;

    // Check compatibility before registering
    let result = schema_client
        .check_compatibility(
            "user-events",
            new_schema.as_bytes(),
            SchemaType::JsonSchema,
            None,  // Use subject's default mode
        )
        .await?;

    if result.is_compatible {
        println!("✅ Safe to register!");
        
        // Now register
        schema_client
            .register_schema("user-events")
            .with_type(SchemaType::JsonSchema)
            .with_schema_data(new_schema.as_bytes())
            .execute()
            .await?;
    } else {
        eprintln!("❌ Incompatible: {:?}", result.errors);
    }
    ```

**Compatibility Modes:**

Compatibility modes control schema evolution at the **subject level**. This is an **admin-only** operation via admin CLI.

| Mode | Description | Use Case |
|------|-------------|----------|
| `Backward` | New schema reads old data | Consumers upgrade first (default) |
| `Forward` | Old schema reads new data | Producers upgrade first |
| `Full` | Both backward + forward | Critical schemas |
| `None` | No validation | Development only |

**Note:** Clients cannot set compatibility mode. This is controlled by administrators using the admin CLI:

```bash
# Admin CLI command (not available in client SDK)
danube-admin schema set-compatibility \
  --subject user-events \
  --mode BACKWARD
```

---

## Producer with Schema

### Option 1: Use Latest Schema Version (Most Common)

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SchemaRegistryClient, SchemaType};
    use serde::Serialize;

    #[derive(Serialize)]
    struct UserEvent {
        user_id: String,
        event: String,
        timestamp: i64,
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        // 1. Register schema
        let schema = r#"{
            "type": "object",
            "properties": {
                "user_id": {"type": "string"},
                "event": {"type": "string"},
                "timestamp": {"type": "integer"}
            },
            "required": ["user_id", "event", "timestamp"]
        }"#;

        let mut schema_client = SchemaRegistryClient::new(&client).await?;
        schema_client
            .register_schema("user-events")
            .with_type(SchemaType::JsonSchema)
            .with_schema_data(schema.as_bytes())
            .execute()
            .await?;

        // 2. Create producer with schema reference (uses latest version)
        let mut producer = client
            .producer()
            .with_topic("/default/user-events")
            .with_name("event-producer")
            .with_schema_subject("user-events")  // Uses latest version
            .build();

        producer.create().await?;

        // 3. Send typed messages
        let event = UserEvent {
            user_id: "user-123".to_string(),
            event: "login".to_string(),
            timestamp: 1234567890,
        };

        let json_bytes = serde_json::to_vec(&event)?;
        let msg_id = producer.send(json_bytes, None).await?;
        println!("📤 Sent message: {}", msg_id);

        Ok(())
    }
    ```

### Option 2: Pin to Specific Version

=== "Rust"

    ```rust
    // Pin producer to specific schema version
    let mut producer = client
        .producer()
        .with_topic("/default/user-events")
        .with_name("producer-v2")
        .with_schema_version("user-events", 2)  // Pin to version 2
        .build();

    producer.create().await?;
    
    // This producer will always use version 2, even if v3+ exists
    ```

**Use cases:**

- Legacy applications that haven't upgraded
- Testing specific schema versions
- Gradual rollout of new versions

### Option 3: Use Minimum Version

=== "Rust"

    ```rust
    // Use version 2 or any newer compatible version
    let mut producer = client
        .producer()
        .with_topic("/default/user-events")
        .with_name("producer-min-v2")
        .with_schema_min_version("user-events", 2)  // v2 or newer
        .build();

    producer.create().await?;
    
    // Will use v2, v3, v4, etc. (latest compatible version)
    ```

**Use cases:**

- Require minimum feature set from schema
- Allow automatic upgrades to compatible versions
- Deprecate old schema versions

### First Producer Privilege

**Important:** The first producer to create a topic assigns its schema subject to that topic.

=== "Rust"

    ```rust
    // First producer - assigns schema to topic
    let first = client.producer()
        .with_topic("new-topic")
        .with_schema_subject("user-events")  // ✅ Sets topic's schema
        .build();
    first.create().await?;

    // Second producer - must match
    let second = client.producer()
        .with_topic("new-topic")
        .with_schema_subject("user-events")  // ✅ Matches, allowed
        .build();
    second.create().await?;

    // Third producer - mismatch!
    let third = client.producer()
        .with_topic("new-topic")
        .with_schema_subject("order-events")  // ❌ ERROR: Different subject!
        .build();
    third.create().await?;  // Returns error
    ```

**Error:**

```bash
Topic 'new-topic' is configured with subject 'user-events',
cannot use subject 'order-events'. Only admin can change topic schema.
```

**Note:** Once a topic has a schema subject, only administrators can change it via admin CLI.

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

---

## Consumer with Schema

### Option 1: Trust Broker Validation (Recommended)

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SubType};
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct UserEvent {
        user_id: String,
        event: String,
        timestamp: i64,
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut consumer = client
            .consumer()
            .with_topic("/default/user-events")
            .with_consumer_name("event-consumer")
            .with_subscription("event-sub")
            .with_subscription_type(SubType::Exclusive)
            .build();

        consumer.subscribe().await?;

        // If topic has ValidationPolicy::Enforce, broker validates everything
        while let Some(message) = consumer.receive().await? {
            // Broker already validated schema, safe to deserialize
            let event: UserEvent = serde_json::from_slice(&message.payload)?;
            println!("📥 Event: {:?}", event);
            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

**When to use:**

- Topic has `ValidationPolicy::Enforce` (strict broker validation)
- Trust broker-side validation
- Best performance (no extra schema lookups)

### Option 2: Client-Side Validation (Untrusted Sources)

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SchemaRegistryClient, SchemaInfo, SubType};
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct UserEvent {
        user_id: String,
        event: String,
        timestamp: i64,
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut consumer = client
            .consumer()
            .with_topic("/default/user-events")
            .with_consumer_name("event-consumer")
            .with_subscription("event-sub")
            .build();

        consumer.subscribe().await?;

        // Create schema client for validation
        let mut schema_client = SchemaRegistryClient::new(&client).await?;

        while let Some(message) = consumer.receive().await? {
            // Fetch schema for validation
            if let Some(schema_id) = message.schema_id {
                let schema: SchemaInfo = schema_client
                    .get_schema_by_id(schema_id)
                    .await?;

                // Optional: Validate payload against schema definition
                // (implement your validation logic here)
                
                println!("Message from schema: {} v{}", 
                    schema.subject, schema.version);
            }

            // Deserialize and process
            let event: UserEvent = serde_json::from_slice(&message.payload)?;
            println!("📥 Event: {:?}", event);
            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

**When to use:**

- Topic has `ValidationPolicy::None` or `Warn`
- Need extra validation beyond broker
- Compliance/audit requirements
- Untrusted data sources

### Option 3: Cache Schemas Locally (Best Performance + Validation)

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SchemaRegistryClient, SchemaInfo};
    use std::collections::HashMap;

    // Schema cache
    struct SchemaCache {
        client: SchemaRegistryClient,
        cache: HashMap<u64, SchemaInfo>,
    }

    impl SchemaCache {
        async fn get_schema(&mut self, schema_id: u64) -> Result<SchemaInfo, Box<dyn std::error::Error>> {
            // Check cache first
            if let Some(schema) = self.cache.get(&schema_id) {
                return Ok(schema.clone());  // Cache hit!
            }

            // Cache miss - fetch from registry
            let schema = self.client.get_schema_by_id(schema_id).await?;
            self.cache.insert(schema_id, schema.clone());
            Ok(schema)
        }
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut consumer = client
            .consumer()
            .with_topic("/default/user-events")
            .with_consumer_name("cached-consumer")
            .with_subscription("cached-sub")
            .build();

        consumer.subscribe().await?;

        // Initialize cache
        let mut cache = SchemaCache {
            client: SchemaRegistryClient::new(&client).await?,
            cache: HashMap::new(),
        };

        while let Some(message) = consumer.receive().await? {
            if let Some(schema_id) = message.schema_id {
                // Fast cached lookup
                let schema = cache.get_schema(schema_id).await?;
                println!("Using schema {} v{}", schema.subject, schema.version);
            }

            // Process message
            let event: UserEvent = serde_json::from_slice(&message.payload)?;
            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

**When to use:**

- High-throughput consumers
- Need validation but want performance
- Schemas don't change frequently

### Comparison of Validation Strategies

| Strategy | Latency | Safety | Use Case |
|----------|---------|--------|----------|
| Trust broker | Lowest | High (if Enforce) | Production with strict policy |
| Always fetch | Highest | Highest | Untrusted sources, audit |
| Cache locally | Medium | Highest | High-throughput + validation |

---

## Schema Evolution Example

### Adding Optional Field (Backward Compatible)

=== "Rust"

    ```rust
    use danube_client::{SchemaRegistryClient, SchemaType};

    // V1 Schema
    let schema_v1 = r#"{
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"}
        },
        "required": ["user_id", "event"]
    }"#;

    // Register V1
    let mut schema_client = SchemaRegistryClient::new(&client).await?;
    schema_client
        .register_schema("events")
        .with_type(SchemaType::JsonSchema)
        .with_schema_data(schema_v1.as_bytes())
        .execute()
        .await?;

    // V2 Schema (add optional field)
    let schema_v2 = r#"{
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
            "metadata": {"type": "string"}
        },
        "required": ["user_id", "event"]
    }"#;

    // Check compatibility
    let compat = schema_client
        .check_compatibility(
            "events",
            schema_v2.as_bytes(),
            SchemaType::JsonSchema,
            None,
        )
        .await?;

    if compat.is_compatible {
        // Register V2
        schema_client
            .register_schema("events")
            .with_type(SchemaType::JsonSchema)
            .with_schema_data(schema_v2.as_bytes())
            .execute()
            .await?;

        println!("✅ Successfully evolved schema to V2");
    }
    ```

**Result:**

- Old consumers can still read V2 messages (ignore extra field)
- New consumers can use `metadata` field
- No breaking changes

---

## Schema Types

### Supported Types

| Type | Description | Status |
|------|-------------|--------|
| `SchemaType::JsonSchema` | JSON Schema validation | ✅ Production |
| `SchemaType::Avro` | Apache Avro binary | ✅ Registration ready |
| `SchemaType::Protobuf` | Protocol Buffers | ✅ Registration ready |
| `SchemaType::String` | UTF-8 text | ✅ Basic validation |
| `SchemaType::Number` | Numeric types | ✅ Basic validation |
| `SchemaType::Bytes` | Raw binary | ✅ No validation |

---

## Troubleshooting

### Schema Registration Fails

```bash
Error: Schema validation failed
```

**Solutions:**

- Validate JSON/Avro schema syntax
- Check schema is well-formed
- Ensure schema type matches content

### Compatibility Check Fails

```
Error: Incompatible schema: removing required field
```

**Solutions:**

- Make field optional instead of removing
- Use `CompatibilityMode::None` for development
- Review compatibility mode requirements

### Deserialization Errors

```bash
Error: missing field `new_field`
```

**Solutions:**

- Make new fields `Option<T>` in Rust
- Add `#[serde(default)]` attribute
- Ensure schema evolution is backward compatible
