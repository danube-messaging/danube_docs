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

**Workflow:**

1. Register schema with Schema Registry
2. Link producer to schema subject
3. Send validated messages
4. Consumer fetches schema and deserializes

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
    let schema = schema_client
        .get_latest_schema("user-events")
        .await?;

    println!("Schema version: {}", schema.version);
    println!("Schema type: {:?}", schema.schema_type);
    println!("Schema definition: {}", String::from_utf8_lossy(&schema.data));
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

### Set Compatibility Mode

=== "Rust"

    ```rust
    use danube_client::CompatibilityMode;

    // Set compatibility mode for a subject
    schema_client
        .set_compatibility_mode("user-events", CompatibilityMode::Full)
        .await?;

    println!("✅ Compatibility mode set to Full");
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

**Available modes:**

- `CompatibilityMode::None` - No checking (development only)
- `CompatibilityMode::Backward` - New schema reads old data (default)
- `CompatibilityMode::Forward` - Old schema reads new data
- `CompatibilityMode::Full` - Both backward and forward

---

## Producer with Schema

### Basic Pattern

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

        // 2. Create producer with schema reference
        let mut producer = client
            .new_producer()
            .with_topic("/default/user-events")
            .with_name("event-producer")
            .with_schema_subject("user-events")  // Link to schema
            .build();

        producer.create().await?;

        // 3. Send typed messages
        let event = UserEvent {
            user_id: "user-123".to_string(),
            event: "login".to_string(),
            timestamp: 1234567890,
        };

        // Serialize to JSON
        let json_bytes = serde_json::to_vec(&event)?;

        // Send (schema ID automatically included)
        let msg_id = producer.send(json_bytes, None).await?;
        println!("📤 Sent message: {}", msg_id);

        Ok(())
    }
    ```

=== "Go"

    ```go
    // Schema Registry not yet available in Go client
    ```

---

## Consumer with Schema

### Basic Pattern

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
            .new_consumer()
            .with_topic("/default/user-events")
            .with_consumer_name("event-consumer")
            .with_subscription("event-sub")
            .with_subscription_type(SubType::Exclusive)
            .build();

        consumer.subscribe().await?;
        let mut stream = consumer.receive().await?;

        while let Some(message) = stream.recv().await {
            // Deserialize JSON message
            match serde_json::from_slice::<UserEvent>(&message.payload) {
                Ok(event) => {
                    println!("📥 Event: {:?}", event);
                    consumer.ack(&message).await?;
                }
                Err(e) => {
                    eprintln!("❌ Deserialization failed: {}", e);
                }
            }
        }

        Ok(())
    }
    ```

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
