# Message Structure

Messages are the fundamental unit of data in Danube. Each message contains a payload plus rich metadata for routing, tracking, acknowledgment, and schema validation.

---

## StreamMessage Structure

```rust
pub struct StreamMessage {
    pub request_id: u64,
    pub msg_id: MessageID,
    pub payload: Vec<u8>,
    pub publish_time: u64,
    pub producer_name: String,
    pub subscription_name: Option<String>,
    pub attributes: HashMap<String, String>,
    pub schema_id: Option<u64>,
    pub schema_version: Option<u32>,
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `u64` | Unique identifier for tracking the message request across the system |
| `msg_id` | `MessageID` | Composite identifier containing routing and location information |
| `payload` | `Vec<u8>` | The actual message content in binary format |
| `publish_time` | `u64` | Unix timestamp (milliseconds) when the message was published |
| `producer_name` | `String` | Name of the producer that sent this message |
| `subscription_name` | `Option<String>` | Name of the subscription (set by broker for consumer delivery) |
| `attributes` | `HashMap<String, String>` | User-defined key-value pairs for custom metadata |
| `schema_id` | `Option<u64>` | Schema Registry ID for message validation (see [Schema Integration](#schema-integration)) |
| `schema_version` | `Option<u32>` | Version of the schema used to serialize this message |

---

## MessageID Structure

The `MessageID` is a composite identifier that enables efficient routing and acknowledgment:

```rust
pub struct MessageID {
    pub producer_id: u64,
    pub topic_name: String,
    pub broker_addr: String,
    pub topic_offset: u64,
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `producer_id` | `u64` | Unique identifier for the producer within this topic |
| `topic_name` | `String` | Full topic name (e.g., `/default/events`) |
| `broker_addr` | `String` | Address of the broker that delivered this message |
| `topic_offset` | `u64` | Monotonic position of the message within the topic |

**Purpose:**

- **Routing:** Broker address enables consumers to send acknowledgments to the correct broker
- **Ordering:** Topic offset provides strict ordering guarantees within a topic
- **Deduplication:** Combination of producer_id + topic_offset creates a unique message identifier
- **Tracking:** Request ID links messages across distributed tracing systems

---

## Schema Integration

Danube integrates with the [Schema Registry](../architecture/schema_registry_architecture.md) to provide type-safe messaging with schema validation.

### Schema Fields in StreamMessage

**`schema_id: Option<u64>`**

Globally unique identifier assigned by the Schema Registry when a schema is registered.

- Present when producer uses schema-validated messages
- Consumers use this ID to fetch the schema from the registry
- Enables schema caching (8-byte overhead vs. sending full schema per message)
- Required when topic has validation policy set to `Enforce`

**`schema_version: Option<u32>`**

Version number of the schema within its subject.

- Tracks which version of the schema was used to serialize this message
- Enables schema evolution tracking and debugging
- Allows consumers to handle multiple schema versions gracefully
- Starts at 1 and auto-increments with each schema update

### How Schema Validation Works

#### 1. Producer Flow

```bash
┌─────────────┐
│  Producer   │
│             │
│ 1. Register │     ┌──────────────────┐
│    Schema   ├────>│ Schema Registry  │
│             │     │                  │
│ 2. Get ID   │<────┤ Returns:         │
│             │     │ - schema_id: 42  │
│             │     │ - version: 1     │
│ 3. Serialize│     └──────────────────┘
│    Message  │
│             │
│ 4. Set      │     StreamMessage {
│    Fields   │       schema_id: Some(42),
│             │       schema_version: Some(1),
│             │       payload: <serialized>,
│ 5. Send     │       ...
│             │     }
└──────┬──────┘
       │
       v
  ┌─────────┐
  │ Broker  │
  └─────────┘
```

#### 2. Broker Validation

When a message arrives, the broker validates schema fields based on the topic's **validation policy**:

| Policy | Behavior |
|--------|----------|
| **None** | No validation; schema fields optional |
| **Warn** | Logs warning if schema_id missing or invalid; allows message |
| **Enforce** | **Rejects** message if schema_id missing, invalid, or doesn't match topic requirements |

#### 3. Consumer Flow

```bash
┌──────────────┐
│   Consumer   │
│              │
│ 1. Receive   │     StreamMessage {
│    Message   │       schema_id: Some(42),
│              │       schema_version: Some(1),
│ 2. Extract   │       payload: <bytes>,
│    schema_id │       ...
│              │     }
│ 3. Fetch     │
│    Schema    ├────>┌──────────────────┐
│              │     │ Schema Registry  │
│ 4. Get Def   │<────┤ Returns schema   │
│              │     │ definition       │
│ 5. Deserialize│    └──────────────────┘
│    Payload   │
│              │
│ 6. Validate  │     { user_id: 123, action: "login" }
│    Struct    │
│              │
└──────────────┘
```

### Benefits of Schema Integration

✅ **Type Safety**  
Messages are validated against a contract, preventing data corruption.

✅ **Bandwidth Efficiency**  
Only 8-12 bytes overhead (schema_id + version) instead of full schema (potentially kilobytes).

✅ **Schema Evolution**  
Consumers can handle multiple schema versions using the `schema_version` field.

✅ **Debugging**  
Knowing which schema version produced a message simplifies troubleshooting.

✅ **Governance**  
Centralized schema management with compatibility checking and audit trails.
