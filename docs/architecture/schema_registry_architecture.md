# Danube Schema Registry Architecture

[Danube](https://github.com/danube-messaging/danube)'s Schema Registry is a centralized service that manages message schemas with versioning, compatibility checking, and validation capabilities. It ensures data quality, enables safe schema evolution, and provides a governance layer for all messages flowing through the Danube messaging system.

This page explains what the Schema Registry is, why it's essential for Danube messaging systems, how it works at a high level, and how to interact with it through APIs and CLI tools.

---

## What is a Schema Registry?

A **Schema Registry** is a standalone service that stores, versions, and validates schemas, the contracts that define the structure of the messages. Instead of each topic managing its own schema independently, the Schema Registry provides:

- **Centralized schema management** - Single source of truth for all schemas across the messaging infrastructure
- **Schema versioning** - Track changes over time with automatic version management
- **Compatibility enforcement** - Prevent breaking changes that could crash consumers
- **Schema reuse** - Share schemas across multiple topics to reduce duplication
- **Data governance** - Audit who created schemas, when they changed, and track dependencies

Think of it as a "contract repository" where producers and consumers agree on message structure, ensuring everyone speaks the same language.

---

## High-Level Architecture

```bash
┌─────────────┐         ┌──────────────────┐         ┌──────────────┐
│  Producer   │         │  Schema Registry │         │   Consumer   │
│             │         │                  │         │              │
│ 1. Register ├────────>│  • Store schemas │<────────┤ 4. Fetch     │
│    Schema   │         │  • Version ctrl  │         │    Schema    │
│             │         │  • Validate      │         │              │
│ 2. Get ID   │<────────┤  • Check compat  │         │ 5. Deserialize│
│             │         │                  │         │    Messages   │
│ 3. Send msg │         └────────┬─────────┘         │              │
│  (with ID)  │                  │                   │              │
└──────┬──────┘                  │                   └──────▲───────┘
       │                         │                          │
       │                    ┌────▼────┐                     │
       │                    │  ETCD   │                     │
       │                    │Metadata │                     │
       │                    └─────────┘                     │
       │                                                    │
       │         ┌──────────────────┐                       │
       └────────>│  Danube Broker   │───────────────────────┘
                 │                  │
                 │ • Route messages │
                 │ • Validate IDs   │
                 │ • Enforce policy │
                 └──────────────────┘
```

### Component Roles

**Schema Registry Service**  
Standalone gRPC service managing all schema operations. Handles registration, retrieval, versioning, and compatibility checking independently from message routing.

**ETCD Metadata Store**  
Persistent storage for all schema metadata, versions, and compatibility settings. Provides distributed consistency across broker cluster.

**Danube Broker**  
Enforces schema validation on messages. Checks that message schema IDs match topic requirements before accepting or dispatching messages.

**Producers**  
Register schemas, serialize messages according to schema, include schema ID in message metadata.

**Consumers**  
Fetch schemas from registry, deserialize messages using correct schema version, optionally validate data structures.

---

## Core Concepts

### Subjects

A **subject** is a named container for schema versions. Typically, one subject corresponds to one message type (e.g., `user-events`, `payment-transactions`).

- Subjects enable multiple topics to share the same schema
- Subject names follow your naming conventions (commonly matches topic name)
- Each subject tracks its own version history and compatibility settings

### Versions

Every time you register a schema, it creates a **new version** if the content differs from existing versions.

- Versions start at 1 and increment automatically
- Versions are immutable once created
- Full version history is preserved indefinitely
- Duplicate schemas are detected via fingerprinting (no duplicate versions created)

### Compatibility Modes (Subject-Level)

**Compatibility modes** control what schema changes are allowed when registering new versions. This is a **subject-level** setting, meaning all topics sharing the same subject inherit the same compatibility rules.

| Mode | Description | When to Use | Example Change |
|------|-------------|-------------|----------------|
| **Backward** | New schema can read old data | Consumers upgrade before producers | Add optional field |
| **Forward** | Old schema can read new data | Producers upgrade before consumers | Remove optional field |
| **Full** | Both backward and forward | Critical schemas needing both directions | Only add optional fields |
| **None** | No validation | Development/testing only | Any change allowed |

**Default:** Backward (industry standard, covers 90% of use cases)

Each subject has its own compatibility mode, configurable by administrators only.

### Validation Policies (Topic-Level)

**Validation policies** control how the broker enforces schema validation for messages on a specific topic. This is a **topic-level** setting, meaning different topics using the same schema subject can have different validation strictness.

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **None** | No validation | Development topics, unstructured data |
| **Warn** | Validate and log errors, but accept messages | Monitoring/debugging production issues |
| **Enforce** | Reject invalid messages | Production topics requiring strict data quality |

**Default:** None

**Example:** A `user-events-dev` topic might use `Warn` policy for flexibility, while `user-events-prod` uses `Enforce` for strict validation—both sharing the same schema subject.

### Schema Types

[Danube](https://github.com/danube-messaging/danube) supports multiple schema formats:

- **JSON Schema** - Fully validated and production-ready
- **Avro** - Apache Avro format (registration and storage ready)
- **Protobuf** - Protocol Buffers (registration and storage ready)
- **String** - UTF-8 text validation
- **Number** - Numeric types
- **Bytes** - Raw binary (no validation)

### Topic-Schema Binding

Every topic can be associated with exactly one schema subject:

- **One Topic → One Subject** - Each topic uses a single schema subject
- **One Subject → Many Topics** - Multiple topics can share the same schema subject
- **First Producer Privilege** - The first producer to create a topic assigns its schema subject
- **Immutable Binding** - Once set, only administrators can change a topic's schema subject

**Example:**

```bash
SUBJECT: "user-events-value"
  ├─ Version 1, 2, 3... (evolves over time)
  ├─ Compatibility: BACKWARD (subject-level)
  └─ Used by:
      ├─ TOPIC: "user-events-dev" (ValidationPolicy: WARN)
      └─ TOPIC: "user-events-prod" (ValidationPolicy: ENFORCE)
```

---

## How It Works

### 1. Schema Registration

When you register a schema:

**Using CLI:**

```bash
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file user-events.json \
  --description "User activity events"
```

**Using Rust SDK:**

```rust
use danube_client::{SchemaRegistryClient, SchemaType};

let mut schema_client = SchemaRegistryClient::new(&client).await?;

// Register the schema and get schema ID
let schema_id = schema_client
    .register_schema("user-events")
    .with_type(SchemaType::Avro)
    .with_schema_data(avro_schema.as_bytes())
    .execute()
    .await?;

println!("✅ Registered schema with ID: {}", schema_id);
```

The Schema Registry:

1. Validates the schema definition (syntax, structure)
2. Checks compatibility with existing versions (if mode != None)
3. Computes fingerprint to detect duplicates
4. Assigns a unique schema ID (global across all subjects)
5. Creates new version number (auto-increment)
6. Stores in ETCD with full metadata (creator, timestamp, description, tags)
7. Returns schema ID and version to client

### 2. Schema Evolution

When you update a schema:

**Using CLI:**

```bash
# Test compatibility first
danube-admin-cli schemas check user-events \
  --file user-events-v2.json \
  --schema-type json_schema

# If compatible, register new version
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file user-events-v2.json
```

**Using Rust SDK:**

```rust
// Check compatibility before registering
let compatibility_result = schema_client
    .check_compatibility(
        "user-events",
        schema_v2.as_bytes().to_vec(),
        SchemaType::Avro,
        None,
    )
    .await?;

if compatibility_result.is_compatible {
    // Safe to register new version
    let schema_id_v2 = schema_client
        .register_schema("user-events")
        .with_type(SchemaType::Avro)
        .with_schema_data(schema_v2.as_bytes())
        .execute()
        .await?;
    println!("✅ Schema v2 registered with ID: {}", schema_id_v2);
} else {
    println!("❌ Schema incompatible: {:?}", compatibility_result.errors);
}
```

The compatibility checker:

1. Retrieves latest version for the subject
2. Compares old vs. new schema based on compatibility mode
3. Validates the change is safe (e.g., adding optional field in Backward mode)
4. Rejects incompatible changes with detailed error message
5. If compatible, allows registration as new version

### 3. Message Production

Producers reference schemas when sending messages:

**Using CLI:**

```bash
danube-cli produce \
  -t /default/user-events \
  --schema-subject user-events \
  -m '{"user_id": "123", "action": "login"}'
```

**Using Rust SDK:**

```rust
use danube_client::DanubeClient;

// Create producer with schema reference
let mut producer = client
    .producer()
    .with_topic("/default/user-events")
    .with_name("user_events_producer")
    .with_schema_subject("user-events")  // Links to schema (latest version)
    .build();

producer.create().await?;

// Or pin to specific version
let mut producer_v2 = client
    .producer()
    .with_topic("/default/user-events")
    .with_name("user_events_producer_v2")
    .with_schema_version("user-events", 2)  // Pin to version 2
    .build();

producer_v2.create().await?;

// Serialize and send message
let event = UserEvent { user_id: "123", action: "login", ... };
let avro_data = serde_json::to_vec(&event)?;

let message_id = producer.send(avro_data, None).await?;
println!("📤 Sent message: {}", message_id);
```

**First Producer Privilege:**

The first producer to create a topic automatically assigns its schema subject to that topic. Subsequent producers must use the same subject:

```rust
// First producer - assigns schema subject to topic
let first = client.producer()
    .with_topic("new-topic")
    .with_schema_subject("user-events")  // ✅ Sets topic's schema
    .build();

// Second producer - must match
let second = client.producer()
    .with_topic("new-topic")
    .with_schema_subject("user-events")  // ✅ Matches, allowed
    .build();

// Third producer - mismatched subject
let third = client.producer()
    .with_topic("new-topic")
    .with_schema_subject("order-events")  // ❌ ERROR: Subject mismatch
    .build();
```

The flow:

1. Producer validates schema subject exists in registry
2. If topic doesn't exist, creates it with the specified schema subject
3. If topic exists, validates subject matches topic's assigned subject
4. Retrieves schema ID from registry (cached locally)
5. Serializes message according to schema (validation happens client-side)
6. Includes schema ID in message metadata (8 bytes overhead)
7. Sends message to broker
8. Broker validates schema ID matches topic's subject
9. Applies topic's validation policy (None/Warn/Enforce)
10. Message is accepted and routed to consumers

### 4. Message Consumption

Consumers fetch schemas to deserialize messages:

**Using CLI:**

```bash
danube-cli consume \
  -t /default/user-events \
  -m my-subscription
```

**Using Rust SDK:**

```rust
use danube_client::{DanubeClient, SchemaRegistryClient, SchemaInfo, SubType};

// Create consumer
let mut consumer = client
    .consumer()
    .with_topic("/default/user-events")
    .with_consumer_name("user_events_consumer")
    .with_subscription("my-subscription")
    .with_subscription_type(SubType::Exclusive)
    .build();

consumer.subscribe().await?;
let mut message_stream = consumer.receive().await?;

// Create schema client for validation (optional)
let mut schema_client = SchemaRegistryClient::new(&client).await?;

// Receive and deserialize messages
while let Some(message) = message_stream.recv().await {
    // Option 1: Trust broker validation (most common)
    let event = serde_json::from_slice::<UserEvent>(&message.payload)?;
    println!("📥 Received: {:?}", event);
    
    // Option 2: Client-side validation (if broker policy is Warn/None)
    if let Some(schema_id) = message.schema_id {
        let schema: SchemaInfo = schema_client.get_schema_by_id(schema_id).await?;
        // Validate payload against schema if needed
    }
    
    // Acknowledge
    consumer.ack(&message).await?;
}
```

The flow:

1. Consumer receives message with schema ID and version
2. Can fetch schema definition from registry (cached locally) if needed
3. Deserializes message using schema
4. Optional: Client-side validation for extra safety
5. Processes validated message

**Consumer Validation Options:**

- **Trust broker** - If topic has `Enforce` policy, no client validation needed
- **Cache + validate** - Fetch schemas once, cache locally, validate on consumption
- **Always fetch** - Query registry for every message (high latency, not recommended)

---

## Validation Layers

Danube provides **three validation layers** for maximum flexibility:

### Producer-Side Validation

- Applications serialize data according to schema before sending
- Schema validation happens during serialization
- Catches errors at source before data enters system
- **Recommended:** Always validate at producer

### Broker-Side Validation (Topic-Level Policy)

- Broker checks message schema ID matches topic's assigned subject
- Three policy levels: None, Warn, Enforce (configured per topic by admin)
- **Enforce mode:** Rejects invalid messages before routing (production)
- **Warn mode:** Logs warnings but allows message (monitoring/debugging)
- **None mode:** No validation (development only)
- Controlled by `ValidationPolicy` (topic-level setting)
- Configurable independently per topic using same schema subject

### Consumer-Side Validation

- Consumers deserialize messages using schema
- Optional struct validation at startup to ensure compatibility
- Prevents runtime deserialization errors
- **Recommended:** Validate structs at consumer startup

This multi-layer approach ensures data quality at every stage while maintaining flexibility.

---

## Storage Model

### ETCD Organization

Schemas and topic configurations are stored hierarchically in ETCD:

```bash
/schemas/
  ├── {subject}/
  │   ├── metadata                    # Subject-level metadata
  │   │   ├── compatibility_mode      # BACKWARD/FORWARD/FULL/NONE
  │   │   ├── created_at
  │   │   └── created_by
  │   ├── compatibility               # Current compatibility mode
  │   └── versions/
  │       ├── 1                       # Version 1 data
  │       ├── 2                       # Version 2 data
  │       └── 3                       # Version 3 data
  ├── _global/
  │   └── next_schema_id              # Global schema ID counter
  └── _index/
      └── by_id/{schema_id}           # Reverse index: ID → subject

/topics/
  └── {topic_name}/
      └── schema_config               # Topic-level schema settings
          ├── subject                 # Schema subject for this topic
          ├── validation_policy       # NONE/WARN/ENFORCE
          └── enable_payload_validation  # true/false
```

### Caching Strategy

To optimize performance, Danube uses distributed caching:

- **Writes** go directly to ETCD (source of truth)
- **Reads** served from local cache (eventually consistent)
- **Updates** propagated via ETCD watch mechanism (automatic invalidation)
- **Schema IDs** cached in topics for fast validation (no registry lookup per message)

This pattern ensures cluster-wide consistency while maintaining low-latency reads.

---

## Use Cases

### Microservices Event Bus

Share schemas across microservices to ensure contract compliance:

- **User Service** publishes `user-registered` events
- **Email Service** subscribes and validates against schema
- **Analytics Service** subscribes and validates against same schema
- **CRM Service** subscribes and validates against same schema

All services share one schema subject, ensuring consistent message structure.

### Schema Evolution During Upgrades

Safely evolve schemas during rolling deployments:

1. **v1 Schema:** `{user_id, action}`
2. **Add optional field:** `{user_id, action, email?}` → Backward compatible
3. Deploy consumers first (can read old + new messages)
4. Deploy producers second (send new format)
5. All consumers upgraded → Make field required in v3 if needed

### Multi-Environment Management

Use different compatibility modes per environment:

- **Development:** `mode: none` - Fast iteration, no restrictions
- **Staging:** `mode: backward` - Test compatibility checks
- **Production:** `mode: full` - Strictest safety

Same schemas, different governance levels based on environment needs.

### Data Governance and Compliance

Track all schema changes with audit trail:

- Who created each schema version (created_by)
- When it was created (timestamp)
- What changed (description, tags)
- Which topics use which schemas
- Full version history for compliance audits

---

## Metrics and Monitoring

The Schema Registry exposes Prometheus metrics for observability:

**Schema Validation Metrics:**

- `schema_validation_total` - Total validation attempts
- `schema_validation_failures_total` - Failed validations

**Labels:**

- `topic` - Which topic validation occurred on
- `policy` - Validation policy (Warn/Enforce)
- `reason` - Failure reason (missing_schema_id, schema_mismatch)

Use these metrics to:

- Monitor schema adoption across topics
- Track validation failure rates
- Alert on breaking changes or misconfigurations
- Measure impact of schema updates

---

## Summary

The [Danube](https://github.com/danube-messaging/danube) Schema Registry transforms schema management from an ad-hoc, error-prone process into a robust, centralized governance layer. It enables:

- **Safe evolution** of message contracts without breaking consumers
- **Data quality guarantees** through multi-layer validation
- **Operational visibility** with full audit trails and versioning
- **Developer confidence** with compatibility checking and type safety
- **Production readiness** following industry-standard patterns

By centralizing schema management, Danube ensures that all participants in the messaging infrastructure speak the same language, evolving together safely over time.
