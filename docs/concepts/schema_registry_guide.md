# Schema Registry User Guide

This guide explains how to use Danube's Schema Registry from both the **client SDK** (application code) and the **admin CLI** (cluster administration). It clarifies what operations are available at which level and who can perform them.

---

## Quick Reference: Who Can Do What?

| Operation | Client SDK | Admin CLI | Level |
|-----------|------------|-----------|-------|
| **Register schema** | ✅ Yes | ✅ Yes | Subject |
| **Set compatibility mode** | ❌ No | ✅ Yes | Subject |
| **Assign schema to topic** | ✅ First producer | ✅ Yes (override) | Topic |
| **Set validation policy** | ❌ No | ✅ Yes | Topic |
| **Enable payload validation** | ❌ No | ✅ Yes | Topic |
| **Choose schema version** | ✅ Yes | N/A | Producer |
| **Fetch schemas** | ✅ Yes | ✅ Yes | - |
| **Delete schema version** | ❌ No | ✅ Yes | Subject |

---

## Understanding Policy Levels

Danube has **two policy levels** that control different aspects of schema management:

### Subject-Level (Applies to Schema Evolution)

**What it controls:** How schemas can evolve over time

**Set by:** Administrators only

**Stored at:** `/schemas/{subject}/compatibility`

**Applies to:** All topics using this subject inherit the same compatibility rules

**Settings:**

- **Compatibility Mode** - `BACKWARD` | `FORWARD` | `FULL` | `NONE`

**Example:**

```
SUBJECT: "user-events-value"
├─ Compatibility Mode: BACKWARD (subject-level)
├─ Version 1, 2, 3...
└─ Used by multiple topics (all inherit BACKWARD mode)
```

### Topic-Level (Applies to Message Validation)

**What it controls:** How strictly the broker enforces schema validation for messages

**Set by:** Administrators only

**Stored at:** `/topics/{topic}/schema_config`

**Applies to:** Only that specific topic

**Settings:**

- **Schema Subject** - Which schema this topic uses
- **Validation Policy** - `NONE` | `WARN` | `ENFORCE`
- **Enable Payload Validation** - `true` | `false`

**Example:**

```
TOPIC: "user-events-dev"
├─ Schema Subject: "user-events-value"
├─ Validation Policy: WARN (lenient for dev)
└─ Payload Validation: false

TOPIC: "user-events-prod"
├─ Schema Subject: "user-events-value" (SAME SUBJECT!)
├─ Validation Policy: ENFORCE (strict for prod)
└─ Payload Validation: true
```

---

## Client SDK Operations (Application Code)

Use the client SDK in your application code to interact with schemas during normal operation.

### 1. Register a New Schema

**Who can:** Any producer/application  
**When:** Before creating producers, or when evolving schemas  
**Level:** Subject

```rust
use danube_client::{DanubeClient, SchemaRegistryClient, SchemaType};

let client = DanubeClient::builder()
    .service_url("http://localhost:6650")
    .build()
    .await?;

let mut schema_client = SchemaRegistryClient::new(&client).await?;

// Register Avro schema
let avro_schema = r#"
{
  "type": "record",
  "name": "UserEvent",
  "fields": [
    {"name": "user_id", "type": "string"},
    {"name": "action", "type": "string"}
  ]
}
"#;

let schema_id = schema_client
    .register_schema("user-events-value")
    .with_type(SchemaType::Avro)
    .with_schema_data(avro_schema.as_bytes())
    .execute()
    .await?;

println!("✅ Registered schema ID: {}", schema_id);
```

**What happens:**

1. Creates new subject if it doesn't exist (with default `BACKWARD` compatibility)
2. OR adds new version to existing subject (checks compatibility automatically)
3. Returns globally unique schema ID
4. **Cannot set compatibility mode** (admin-only)

### 2. Fetch Schema Information

**Who can:** Any consumer/application  
**When:** To deserialize messages or validate payloads  
**Level:** N/A (read-only)

```rust
use danube_client::{SchemaRegistryClient, SchemaInfo};

let mut schema_client = SchemaRegistryClient::new(&client).await?;

// Get latest schema for a subject
let schema: SchemaInfo = schema_client
    .get_latest_schema("user-events-value")
    .await?;

println!("Schema ID: {}", schema.schema_id);
println!("Version: {}", schema.version);
println!("Type: {}", schema.schema_type);

// Get schema by ID (from message metadata)
if let Some(schema_id) = message.schema_id {
    let schema: SchemaInfo = schema_client
        .get_schema_by_id(schema_id)
        .await?;
    
    // Access schema definition
    if let Some(schema_str) = schema.schema_definition_as_string() {
        println!("Schema: {}", schema_str);
    }
}
```

### 3. Create Producer with Schema (First Producer Privilege)

**Who can:** Any producer  
**When:** Creating a new producer  
**Level:** Topic (assigns schema subject to topic)

```rust
use danube_client::DanubeClient;

// Option A: Use latest schema version (most common)
let mut producer = client
    .producer()
    .with_topic("/default/user-events")
    .with_name("user_events_producer")
    .with_schema_subject("user-events-value")  // Links to schema
    .build();

producer.create().await?;

// Option B: Pin to specific version
let mut producer_v2 = client
    .producer()
    .with_topic("/default/user-events")
    .with_name("producer_v2")
    .with_schema_version("user-events-value", 2)  // Use version 2
    .build();

producer_v2.create().await?;

// Option C: Use minimum version
let mut producer_min = client
    .producer()
    .with_topic("/default/user-events")
    .with_name("producer_min")
    .with_schema_min_version("user-events-value", 2)  // v2 or newer
    .build();

producer_min.create().await?;
```

**First Producer Privilege:**

The **first producer** to create a topic automatically assigns its schema subject to that topic:

```rust
// First producer - assigns schema subject
let first = client.producer()
    .with_topic("new-topic")
    .with_schema_subject("user-events-value")  // ✅ Sets topic's schema
    .build();
first.create().await?;

// Second producer - must match
let second = client.producer()
    .with_topic("new-topic")
    .with_schema_subject("user-events-value")  // ✅ Matches, allowed
    .build();
second.create().await?;

// Third producer - mismatch!
let third = client.producer()
    .with_topic("new-topic")
    .with_schema_subject("order-events-value")  // ❌ ERROR: Different subject!
    .build();
third.create().await?;  // Returns error
```

**Error:**

```
Topic 'new-topic' is configured with subject 'user-events-value',
cannot use subject 'order-events-value'. Only admin can change topic schema.
```

### 4. Consumer Schema Validation (Optional)

**Who can:** Any consumer  
**When:** Receiving messages  
**Level:** Client-side validation

```rust
use danube_client::{DanubeClient, SchemaRegistryClient, SchemaInfo, SubType};

let mut consumer = client
    .consumer()
    .with_topic("/default/user-events")
    .with_consumer_name("my_consumer")
    .with_subscription("my-subscription")
    .build();

consumer.subscribe().await?;

// Option 1: Trust broker validation (recommended if ValidationPolicy = ENFORCE)
while let Some(message) = consumer.receive().await? {
    let event: UserEvent = serde_json::from_slice(&message.payload)?;
    process_event(event).await?;
    consumer.ack(&message).await?;
}

// Option 2: Client-side validation (if broker policy is WARN or NONE)
let mut schema_client = SchemaRegistryClient::new(&client).await?;

while let Some(message) = consumer.receive().await? {
    // Fetch schema for validation
    if let Some(schema_id) = message.schema_id {
        let schema: SchemaInfo = schema_client.get_schema_by_id(schema_id).await?;
        
        // Validate payload against schema
        if !validate_payload(&message.payload, &schema) {
            eprintln!("Invalid message: {:?}", message.msg_id);
            consumer.nack(&message).await?;
            continue;
        }
    }
    
    // Process validated message
    let event: UserEvent = serde_json::from_slice(&message.payload)?;
    process_event(event).await?;
    consumer.ack(&message).await?;
}
```

**Validation Strategies:**

| Strategy | Performance | Safety | When to Use |
|----------|-------------|--------|-------------|
| Trust broker | Fastest | High (if Enforce) | Production with strict validation policy |
| Fetch per message | Slowest | Highest | Untrusted sources, audit requirements |
| Cache locally | Medium | Highest | High-throughput + validation needs |

### 5. Check Compatibility Before Registering

**Who can:** Any producer/application  
**When:** Before registering a new schema version  
**Level:** Subject

```rust
use danube_client::{SchemaRegistryClient, SchemaType};

let mut schema_client = SchemaRegistryClient::new(&client).await?;

let new_schema = r#"
{
  "type": "record",
  "name": "UserEvent",
  "fields": [
    {"name": "user_id", "type": "string"},
    {"name": "action", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
"#;

// Check compatibility before registering
let result = schema_client
    .check_compatibility(
        "user-events-value",
        new_schema.as_bytes().to_vec(),
        SchemaType::Avro,
        None,  // Use subject's compatibility mode
    )
    .await?;

if result.is_compatible {
    println!("✅ Schema is compatible!");
    
    // Safe to register
    let schema_id = schema_client
        .register_schema("user-events-value")
        .with_type(SchemaType::Avro)
        .with_schema_data(new_schema.as_bytes())
        .execute()
        .await?;
        
    println!("Registered as schema ID: {}", schema_id);
} else {
    println!("❌ Schema incompatible!");
    for error in result.errors {
        println!("  - {}", error);
    }
}
```

---

## Admin CLI Operations (Cluster Administration)

Use the admin CLI to configure cluster-wide schema settings and topic-level policies.

### 1. Set Compatibility Mode for a Subject

**Who can:** Administrators only  
**When:** During initial schema setup or policy changes  
**Level:** Subject

```bash
# Set compatibility mode for a subject
danube-admin schema set-compatibility \
  --subject user-events-value \
  --mode FULL

# Get current compatibility mode
danube-admin schema get-compatibility \
  --subject user-events-value
```

**Output:**

```
✅ Set compatibility mode for subject 'user-events-value' to FULL
```

**What happens:**

- Updates compatibility mode for the **subject**
- Applies to all future schema registrations
- All topics using this subject inherit the compatibility rules

**Compatibility Modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| `BACKWARD` | New schema reads old data | Consumers upgrade before producers (default) |
| `FORWARD` | Old schema reads new data | Producers upgrade before consumers |
| `FULL` | Both backward + forward | Critical schemas needing both directions |
| `NONE` | No validation | Development/testing only |

### 2. Configure Topic Schema Settings

**Who can:** Administrators only  
**When:** Initial topic setup or policy changes  
**Level:** Topic

```bash
# Configure schema for a topic
danube-admin topic configure-schema \
  --topic /default/user-events-prod \
  --subject user-events-value \
  --validation-policy enforce \
  --enable-payload-validation

# Update only validation policy
danube-admin topic set-validation-policy \
  --topic /default/user-events-dev \
  --policy warn

# View topic schema configuration
danube-admin topic get-schema-config \
  --topic /default/user-events-prod
```

**Output:**

```
Topic: /default/user-events-prod
Schema Subject: user-events-value
Compatibility Mode: BACKWARD (from subject)
Validation Policy: ENFORCE (topic-level)
Payload Validation: ENABLED (topic-level)
```

**What happens:**

1. Associates topic with schema subject
2. Sets validation policy (topic-level)
3. Enables/disables payload validation (topic-level)
4. Stores configuration in ETCD at `/topics/{topic}/schema_config`

**Validation Policies:**

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `none` | No validation | Development topics, unstructured data |
| `warn` | Validate and log errors, accept anyway | Monitoring/debugging production |
| `enforce` | Reject invalid messages | Production requiring strict quality |

### 3. Change Topic's Schema Subject (Dangerous!)

**Who can:** Administrators only  
**When:** Migration or schema refactoring  
**Level:** Topic

```bash
# Change schema subject (dangerous operation)
danube-admin topic update-schema-subject \
  --topic /default/user-events \
  --new-subject user-events-v2-value \
  --force

# Requires --force flag to confirm
```

**⚠️ Warning:**

- Existing producers will fail if their schema subject no longer matches
- Should be done during maintenance window
- Coordinate with application teams

### 4. Different Policies for Different Environments

**Common Pattern:** Same schema, different validation strictness

```bash
# Development: Lenient validation
danube-admin topic configure-schema \
  --topic /default/user-events-dev \
  --subject user-events-value \
  --validation-policy warn \
  --no-payload-validation

# Staging: Moderate validation
danube-admin topic configure-schema \
  --topic /default/user-events-staging \
  --subject user-events-value \
  --validation-policy warn \
  --enable-payload-validation

# Production: Strict validation
danube-admin topic configure-schema \
  --topic /default/user-events-prod \
  --subject user-events-value \
  --validation-policy enforce \
  --enable-payload-validation
```

**Result:**

```
SUBJECT: "user-events-value" (shared)
  ├─ Compatibility Mode: BACKWARD (applies to all)
  └─ Used by:
      ├─ /default/user-events-dev (WARN, no payload check)
      ├─ /default/user-events-staging (WARN, with payload check)
      └─ /default/user-events-prod (ENFORCE, with payload check)
```

### 5. Delete Schema Version (Dangerous!)

**Who can:** Administrators only  
**When:** Deprecating old versions  
**Level:** Subject

```bash
# Delete a specific schema version
danube-admin schema delete-version \
  --subject user-events-value \
  --version 2 \
  --force
```

**⚠️ Warning:**

- May break existing consumers using that version
- Cannot be undone
- Requires `--force` flag

---

## Complete Workflow Example

### Scenario: Setting up Schema Registry for Production

**Step 1: Admin sets up subject with compatibility mode**

```bash
# Admin: Configure subject-level policy
danube-admin schema set-compatibility \
  --subject user-events-value \
  --mode BACKWARD
```

**Step 2: Developer registers initial schema**

```rust
// Developer: Register v1 schema from application
let schema_v1 = r#"
{
  "type": "record",
  "name": "UserEvent",
  "fields": [
    {"name": "user_id", "type": "string"},
    {"name": "action", "type": "string"}
  ]
}
"#;

let schema_id_v1 = schema_client
    .register_schema("user-events-value")
    .with_type(SchemaType::Avro)
    .with_schema_data(schema_v1.as_bytes())
    .execute()
    .await?;

println!("Registered v1 with ID: {}", schema_id_v1);
```

**Step 3: Admin configures topic validation policies**

```bash
# Admin: Configure dev topic (lenient)
danube-admin topic configure-schema \
  --topic /default/user-events-dev \
  --subject user-events-value \
  --validation-policy warn

# Admin: Configure prod topic (strict)
danube-admin topic configure-schema \
  --topic /default/user-events-prod \
  --subject user-events-value \
  --validation-policy enforce \
  --enable-payload-validation
```

**Step 4: Developer creates producer (first producer privilege)**

```rust
// Developer: Create producer (assigns schema to topic if new)
let mut producer = client
    .producer()
    .with_topic("/default/user-events-prod")
    .with_name("prod_producer")
    .with_schema_subject("user-events-value")  // Must match admin config
    .build();

producer.create().await?;

// Send messages
let event = UserEvent {
    user_id: "123".to_string(),
    action: "login".to_string(),
};

let payload = serde_json::to_vec(&event)?;
producer.send(payload, None).await?;
```

**Step 5: Developer evolves schema (backward compatible)**

```rust
// Developer: Check compatibility first
let schema_v2 = r#"
{
  "type": "record",
  "name": "UserEvent",
  "fields": [
    {"name": "user_id", "type": "string"},
    {"name": "action", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
"#;

let compat = schema_client
    .check_compatibility(
        "user-events-value",
        schema_v2.as_bytes().to_vec(),
        SchemaType::Avro,
        None,
    )
    .await?;

if compat.is_compatible {
    // Register new version
    let schema_id_v2 = schema_client
        .register_schema("user-events-value")
        .with_type(SchemaType::Avro)
        .with_schema_data(schema_v2.as_bytes())
        .execute()
        .await?;
        
    println!("Registered v2 with ID: {}", schema_id_v2);
}
```

**Step 6: Developer consumes with validation**

```rust
// Developer: Consumer validates messages
let mut consumer = client
    .consumer()
    .with_topic("/default/user-events-prod")
    .with_consumer_name("prod_consumer")
    .with_subscription("prod-subscription")
    .build();

consumer.subscribe().await?;

// Trust broker validation (topic has ENFORCE policy)
while let Some(message) = consumer.receive().await? {
    // Broker already validated, safe to deserialize
    let event: UserEvent = serde_json::from_slice(&message.payload)?;
    println!("Received: {:?}", event);
    consumer.ack(&message).await?;
}
```

---

## Best Practices

### For Developers (Client SDK)

1. **Always register schemas before creating producers**

   ```rust
   // Register schema first
   let schema_id = schema_client.register_schema("my-subject")...
   
   // Then create producer
   let producer = client.producer()
       .with_schema_subject("my-subject")...
   ```

2. **Check compatibility before evolving schemas**

   ```rust
   let compat = schema_client.check_compatibility(...).await?;
   if compat.is_compatible {
       schema_client.register_schema(...).await?;
   }
   ```

3. **Use version pinning for critical producers**

   ```rust
   // Pin to known-good version
   .with_schema_version("my-subject", 3)
   ```

4. **Cache schemas at consumer startup**

   ```rust
   // Fetch once at startup
   let schema = schema_client.get_latest_schema("my-subject").await?;
   
   // Reuse for all messages
   ```

### For Administrators (Admin CLI)

1. **Set compatibility mode early**

   ```bash
   # Set before first schema registration
   danube-admin schema set-compatibility \
     --subject my-subject \
     --mode BACKWARD
   ```

2. **Use different validation policies per environment**

   ```bash
   # Dev: lenient
   danube-admin topic configure-schema \
     --topic /dev/my-topic \
     --validation-policy warn
   
   # Prod: strict
   danube-admin topic configure-schema \
     --topic /prod/my-topic \
     --validation-policy enforce
   ```

3. **Never delete schema versions in production**
   - Consumers may still reference old versions
   - Breaks message deserialization

4. **Document schema changes**

   ```bash
   # Use descriptive commit messages
   git commit -m "Add optional email field to UserEvent schema (v2)"
   ```

---

## Summary

### Policy Levels

| Setting | Level | Set By | Can Differ Per Topic? |
|---------|-------|--------|----------------------|
| **Compatibility Mode** | Subject | Admin | N/A (applies to subject) |
| **Validation Policy** | Topic | Admin | ✅ Yes |
| **Payload Validation** | Topic | Admin | ✅ Yes |
| **Schema Subject** | Topic | First Producer or Admin | ❌ No (one per topic) |
| **Schema Version** | Producer | Each Producer | ✅ Yes |

### Who Can Do What

| Operation | Client | Admin | Reason |
|-----------|--------|-------|--------|
| Register schema | ✅ | ✅ | Enable schema evolution |
| Set compatibility | ❌ | ✅ | Governance control |
| Assign schema to topic | ✅ (first) | ✅ | Flexibility + control |
| Set validation policy | ❌ | ✅ | Operational policy |
| Choose schema version | ✅ | N/A | Producer autonomy |
| Delete schemas | ❌ | ✅ | Prevent accidents |

This separation ensures **developers can iterate quickly** while **administrators maintain governance and operational safety**.
