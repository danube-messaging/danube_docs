# Schema Registry Integration

The Schema Registry provides type safety, validation, and schema evolution for your messages. This guide shows how to use it from client applications.

The **Rust**, **Go**, and **Python** client libraries all support the Schema Registry.

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
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let schema_client = client.schema();

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }

        schemaClient := client.Schema()
        _ = schemaClient
    }
    ```

=== "Python"

    ```python
    import asyncio
    from danube import DanubeClientBuilder

    async def main():
        client = await (
            DanubeClientBuilder()
            .service_url("http://127.0.0.1:6650")
            .build()
        )

        schema_client = client.schema()

    asyncio.run(main())
    ```

**Note:** The schema client is obtained from `DanubeClient` via `.Schema()` (Go), `.schema()` (Rust/Python), sharing the same connection pool — just like producers and consumers.

---

## Registering Schemas

### JSON Schema

=== "Rust"

    ```rust
    use danube_client::SchemaType;

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
        .execute()
        .await?;

    println!("✅ Registered schema ID: {}", schema_id);
    ```

=== "Go"

    ```go
    import (
        "context"
        "fmt"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }

        ctx := context.Background()

        jsonSchema := `{"type": "object", "properties": {"user_id": {"type": "string"}, "event": {"type": "string"}, "timestamp": {"type": "integer"}}, "required": ["user_id", "event", "timestamp"]}`

        schemaID, err := client.Schema().RegisterSchema("user-events").
            WithType(danube.SchemaTypeJSONSchema).
            WithSchemaData([]byte(jsonSchema)).
            Execute(ctx)
        if err != nil {
            log.Fatalf("failed to register schema: %v", err)
        }

        fmt.Printf("Registered schema ID: %d\n", schemaID)
    }
    ```

=== "Python"

    ```python
    import json
    from danube import SchemaType

    json_schema = json.dumps({
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
            "timestamp": {"type": "integer"},
        },
        "required": ["user_id", "event", "timestamp"],
    })

    schema_id = await (
        schema_client.register_schema("user-events")
        .with_type(SchemaType.JSON_SCHEMA)
        .with_schema_data(json_schema.encode())
        .execute()
    )

    print(f"Registered schema ID: {schema_id}")
    ```

### Avro Schema

=== "Rust"

    ```rust
    use danube_client::SchemaType;

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
    avroSchema := `{"type": "record", "name": "UserEvent", "namespace": "com.example", "fields": [{"name": "user_id", "type": "string"}, {"name": "event", "type": "string"}, {"name": "timestamp", "type": "long"}, {"name": "metadata", "type": ["null", "string"], "default": null}]}`

    schemaID, err := client.Schema().RegisterSchema("user-events-avro").
        WithType(danube.SchemaTypeAvro).
        WithSchemaData([]byte(avroSchema)).
        Execute(ctx)
    if err != nil {
        log.Fatalf("failed to register schema: %v", err)
    }

    fmt.Printf("Registered Avro schema: %d\n", schemaID)
    ```

=== "Python"

    ```python
    import json
    from danube import SchemaType

    avro_schema = json.dumps({
        "type": "record",
        "name": "UserEvent",
        "namespace": "com.example",
        "fields": [
            {"name": "user_id", "type": "string"},
            {"name": "event", "type": "string"},
            {"name": "timestamp", "type": "long"},
            {"name": "metadata", "type": ["null", "string"], "default": None},
        ],
    })

    schema_id = await (
        schema_client.register_schema("user-events-avro")
        .with_type(SchemaType.AVRO)
        .with_schema_data(avro_schema.encode())
        .execute()
    )

    print(f"Registered Avro schema: {schema_id}")
    ```

### Idempotent Registration

Registering the same schema content multiple times returns the existing schema ID:

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

=== "Go"

    ```go
    schema, err := client.Schema().GetLatestSchema(ctx, "user-events")
    if err != nil {
        log.Fatalf("failed to get schema: %v", err)
    }

    fmt.Printf("Schema ID: %d\n", schema.SchemaID)
    fmt.Printf("Subject: %s\n", schema.Subject)
    fmt.Printf("Version: %d\n", schema.Version)
    fmt.Printf("Type: %s\n", schema.SchemaType)

    // Get schema definition as string
    if def, err := schema.SchemaDefinitionAsString(); err == nil {
        fmt.Printf("Schema: %s\n", def)
    }
    ```

=== "Python"

    ```python
    schema = await schema_client.get_latest_schema("user-events")

    print(f"Schema ID: {schema.schema_id}")
    print(f"Subject: {schema.subject}")
    print(f"Version: {schema.version}")
    print(f"Type: {schema.schema_type}")

    # Get schema definition as string
    print(f"Schema: {schema.schema_definition_as_string()}")
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
    schema, err := client.Schema().GetLatestSchema(ctx, "user-events")
    if err != nil {
        log.Fatalf("failed to get schema: %v", err)
    }

    fmt.Printf("Schema subject: %s\n", schema.Subject)
    fmt.Printf("Schema version: %d\n", schema.Version)
    ```

=== "Python"

    ```python
    schema = await schema_client.get_schema_by_id(schema_id)

    print(f"Schema subject: {schema.subject}")
    print(f"Schema version: {schema.version}")
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
    versions, err := client.Schema().ListSchemaVersions(ctx, "user-events")
    if err != nil {
        log.Fatalf("failed to list versions: %v", err)
    }

    fmt.Printf("Available versions: %v\n", versions)
    ```

=== "Python"

    ```python
    versions = await schema_client.list_versions("user-events")

    print(f"Available versions: {versions}")  # e.g., [1, 2, 3]
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

=== "Python"

    ```python
    import json
    from danube import SchemaType

    new_schema = json.dumps({
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
            "timestamp": {"type": "integer"},
            "email": {"type": "string"},
        },
        "required": ["user_id", "event", "timestamp"],
    })

    # Check compatibility before registering
    is_compatible, errors = await schema_client.check_compatibility(
        "user-events",
        new_schema.encode(),
        SchemaType.JSON_SCHEMA,
        None,  # Use subject's default mode
    )

    if is_compatible:
        print("✅ Safe to register!")

        # Now register
        await (
            schema_client.register_schema("user-events")
            .with_type(SchemaType.JSON_SCHEMA)
            .with_schema_data(new_schema.encode())
            .execute()
        )
    else:
        print(f"❌ Incompatible: {errors}")
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
    use danube_client::{DanubeClient, SchemaType};
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

        let schema_client = client.schema();
        schema_client
            .register_schema("user-events")
            .with_type(SchemaType::JsonSchema)
            .with_schema_data(schema.as_bytes())
            .execute()
            .await?;

        // 2. Create producer with schema reference (uses latest version)
        let mut producer = client
            .new_producer()
            .with_topic("/default/user-events")
            .with_name("event-producer")
            .with_schema_subject("user-events")  // Uses latest version
            .build()?;

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

=== "Go"

    ```go
    import (
        "context"
        "encoding/json"
        "fmt"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }

        ctx := context.Background()

        // 1. Register schema
        jsonSchema := `{"type": "object", "properties": {"user_id": {"type": "string"}, "event": {"type": "string"}, "timestamp": {"type": "integer"}}, "required": ["user_id", "event", "timestamp"]}`

        _, err = client.Schema().RegisterSchema("user-events").
            WithType(danube.SchemaTypeJSONSchema).
            WithSchemaData([]byte(jsonSchema)).
            Execute(ctx)
        if err != nil {
            log.Fatalf("failed to register schema: %v", err)
        }

        // 2. Create producer with schema reference (uses latest version)
        producer, err := client.NewProducer().
            WithTopic("/default/user-events").
            WithName("event-producer").
            WithSchemaSubject("user-events").
            Build()
        if err != nil {
            log.Fatalf("failed to build producer: %v", err)
        }

        if err := producer.Create(ctx); err != nil {
            log.Fatalf("failed to create producer: %v", err)
        }

        // 3. Send typed messages
        event := map[string]interface{}{
            "user_id":   "user-123",
            "event":     "login",
            "timestamp": 1234567890,
        }

        jsonBytes, _ := json.Marshal(event)
        msgID, err := producer.Send(ctx, jsonBytes, nil)
        if err != nil {
            log.Fatalf("failed to send: %v", err)
        }

        fmt.Printf("Sent message: %v\n", msgID)
    }
    ```

=== "Python"

    ```python
    import asyncio
    import json
    from danube import DanubeClientBuilder, SchemaType

    async def main():
        client = await (
            DanubeClientBuilder()
            .service_url("http://127.0.0.1:6650")
            .build()
        )

        # 1. Register schema
        json_schema = json.dumps({
            "type": "object",
            "properties": {
                "user_id": {"type": "string"},
                "event": {"type": "string"},
                "timestamp": {"type": "integer"},
            },
            "required": ["user_id", "event", "timestamp"],
        })

        schema_client = client.schema()
        await (
            schema_client.register_schema("user-events")
            .with_type(SchemaType.JSON_SCHEMA)
            .with_schema_data(json_schema.encode())
            .execute()
        )

        # 2. Create producer with schema reference (uses latest version)
        producer = (
            client.new_producer()
            .with_topic("/default/user-events")
            .with_name("event-producer")
            .with_schema_subject("user-events")  # Uses latest version
            .build()
        )

        await producer.create()

        # 3. Send typed messages
        event = {
            "user_id": "user-123",
            "event": "login",
            "timestamp": 1234567890,
        }

        json_bytes = json.dumps(event).encode()
        msg_id = await producer.send(json_bytes)
        print(f"📤 Sent message: {msg_id}")

        await producer.close()

    asyncio.run(main())
    ```

### Option 2: Pin to Specific Version

=== "Rust"

    ```rust
    // Pin producer to specific schema version
    let mut producer = client
        .new_producer()
        .with_topic("/default/user-events")
        .with_name("producer-v2")
        .with_schema_version("user-events", 2)  // Pin to version 2
        .build()?;

    producer.create().await?;
    
    // This producer will always use version 2, even if v3+ exists
    ```

=== "Go"

    ```go
    // Pin producer to specific schema version
    producer, err := client.NewProducer().
        WithTopic("/default/user-events").
        WithName("producer-v2").
        WithSchemaVersion("user-events", 2).  // Pin to version 2
        Build()
    if err != nil {
        log.Fatalf("failed to build producer: %v", err)
    }

    if err := producer.Create(ctx); err != nil {
        log.Fatalf("failed to create producer: %v", err)
    }
    // This producer will always use version 2, even if v3+ exists
    ```

=== "Python"

    ```python
    # Pin producer to specific schema version
    producer = (
        client.new_producer()
        .with_topic("/default/user-events")
        .with_name("producer-v2")
        .with_schema_version("user-events", 2)  # Pin to version 2
        .build()
    )

    await producer.create()
    # This producer will always use version 2, even if v3+ exists
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
        .new_producer()
        .with_topic("/default/user-events")
        .with_name("producer-min-v2")
        .with_schema_min_version("user-events", 2)  // v2 or newer
        .build()?;

    producer.create().await?;
    
    // Will use v2, v3, v4, etc. (latest compatible version)
    ```

=== "Go"

    ```go
    // Use version 2 or any newer compatible version
    producer, err := client.NewProducer().
        WithTopic("/default/user-events").
        WithName("producer-min-v2").
        WithSchemaMinVersion("user-events", 2).  // v2 or newer
        Build()
    if err != nil {
        log.Fatalf("failed to build producer: %v", err)
    }

    if err := producer.Create(ctx); err != nil {
        log.Fatalf("failed to create producer: %v", err)
    }
    // Will use v2, v3, v4, etc. (latest compatible version)
    ```

=== "Python"

    ```python
    # Use version 2 or any newer compatible version
    producer = (
        client.new_producer()
        .with_topic("/default/user-events")
        .with_name("producer-min-v2")
        .with_schema_min_version("user-events", 2)  # v2 or newer
        .build()
    )

    await producer.create()
    # Will use v2, v3, v4, etc. (latest compatible version)
    ```

**Use cases:**

- Require minimum feature set from schema
- Allow automatic upgrades to compatible versions
- Deprecate old schema versions

## Schema Evolution Example

### Adding Optional Field (Backward Compatible)

=== "Rust"

    ```rust
    use danube_client::SchemaType;

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
    let schema_client = client.schema();
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

=== "Python"

    ```python
    import json
    from danube import SchemaType

    # V1 Schema
    schema_v1 = json.dumps({
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
        },
        "required": ["user_id", "event"],
    })

    # Register V1
    schema_client = client.schema()
    await (
        schema_client.register_schema("events")
        .with_type(SchemaType.JSON_SCHEMA)
        .with_schema_data(schema_v1.encode())
        .execute()
    )

    # V2 Schema (add optional field)
    schema_v2 = json.dumps({
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "event": {"type": "string"},
            "metadata": {"type": "string"},
        },
        "required": ["user_id", "event"],
    })

    # Check compatibility
    is_compatible, errors = await schema_client.check_compatibility(
        "events",
        schema_v2.encode(),
        SchemaType.JSON_SCHEMA,
        None,
    )

    if is_compatible:
        # Register V2
        await (
            schema_client.register_schema("events")
            .with_type(SchemaType.JSON_SCHEMA)
            .with_schema_data(schema_v2.encode())
            .execute()
        )

        print("✅ Successfully evolved schema to V2")
    ```

**Result:**

- Old consumers can still read V2 messages (ignore extra field)
- New consumers can use `metadata` field
- No breaking changes

---

## Schema Types

### Supported Types

| Rust | Go | Python | Description | Status |
|------|-----|--------|-------------|--------|
| `SchemaType::JsonSchema` | `SchemaTypeJSONSchema` | `SchemaType.JSON_SCHEMA` | JSON Schema validation | ✅ Production |
| `SchemaType::Avro` | `SchemaTypeAvro` | `SchemaType.AVRO` | Apache Avro binary | ✅ Registration ready |
| `SchemaType::Protobuf` | `SchemaTypeProtobuf` | `SchemaType.PROTOBUF` | Protocol Buffers | ✅ Registration ready |
| `SchemaType::String` | `SchemaTypeString` | `SchemaType.STRING` | UTF-8 text | ✅ Basic validation |
| `SchemaType::Number` | `SchemaTypeNumber` | `SchemaType.NUMBER` | Numeric types | ✅ Basic validation |
| `SchemaType::Bytes` | `SchemaTypeBytes` | `SchemaType.BYTES` | Raw binary | ✅ No validation |

