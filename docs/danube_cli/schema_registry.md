# Schema Registry Guide

Master schema management for reliable, validated messaging! 📋

## Table of Contents

- [What is a Schema Registry?](#what-is-a-schema-registry)
- [Schema Management](#schema-management)
- [Schema Types](#schema-types)
- [Schema Evolution](#schema-evolution)
- [Compatibility Modes](#compatibility-modes)
- [Complete Workflows](#complete-workflows)

## What is a Schema Registry?

The Schema Registry is a centralized repository that stores and manages schemas for your messages.

### Why Use Schemas?

**Without Schemas:**

```bash
# Producer sends anything
danube-cli produce -s http://localhost:6650 -m '{"user":123}'  # number
danube-cli produce -s http://localhost:6650 -m '{"user":"abc"}' # string
danube-cli produce -s http://localhost:6650 -m '{"usr":"xyz"}'  # typo!

# Consumer has no idea what to expect! ❌
```

**With Schemas:**

```bash
# Schema defines the contract
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["user_id", "email"]
}

# Only valid messages are accepted ✅
# Consumers know exactly what to expect ✅
# Breaking changes are prevented ✅
```

### Benefits

| Benefit | Description |
|---------|-------------|
| **Type Safety** | Prevent invalid data at the source |
| **Documentation** | Schema serves as living documentation |
| **Evolution** | Safe schema updates with compatibility checking |
| **Validation** | Automatic validation for producers and consumers |
| **Versioning** | Track schema changes over time |

## Schema Management

### Register a Schema

```bash
danube-cli schema register <subject> \
  --schema-type <schema-type> \
  --file <schema-file>
```

**Example:**

```bash
# Create a JSON schema file
cat > user-schema.json << 'EOF'
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "email": {"type": "string", "format": "email"},
    "age": {"type": "integer", "minimum": 0}
  },
  "required": ["user_id", "email"]
}
EOF

# Register the schema
danube-cli schema register user-events \
  --schema-type json_schema \
  --file user-schema.json
```

**Output:**

```
📤 Registering schema 'user-events' (type: JsonSchema)...
✅ Schema registered successfully!
   Subject: user-events
   Schema ID: 1
   Version: 1
```

### Get Schema Details

```bash
danube-cli schema get <subject>
```

**Example:**

```bash
danube-cli schema get user-events
```

**Output:**

```bash
✅ Schema Details
==================================================
Subject:       user-events
Version:       1
Schema ID:     1
Type:          json_schema
==================================================
Schema Definition:
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "email": {"type": "string", "format": "email"},
    "age": {"type": "integer", "minimum": 0}
  },
  "required": ["user_id", "email"]
}
==================================================
```

### List Schema Versions

```bash
danube-cli schema versions <subject>
```

**Example:**

```bash
danube-cli schema versions user-events
```

**Output:**

```bash
✅ Schema Versions for 'user-events'
==================================================
Version 1 (ID: 1) - Current
Version 2 (ID: 2)
Version 3 (ID: 3) - Latest
==================================================
Total versions: 3
```

### Check Compatibility

Before registering a new version, check compatibility:

```bash
danube-cli schema check <subject> \
  --schema-type <schema-type> \
  --file <new-schema-file>
```

**Example:**

```bash
# Create updated schema (v2)
cat > user-schema-v2.json << 'EOF'
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "email": {"type": "string", "format": "email"},
    "age": {"type": "integer", "minimum": 0},
    "name": {"type": "string"}
  },
  "required": ["user_id", "email"]
}
EOF

# Check compatibility
danube-cli schema check user-events \
  --schema-type json_schema \
  --file user-schema-v2.json
```

**Output (Compatible):**

```bash
✅ Schema is compatible!
   Subject: user-events
   Compatibility Mode: backward
   
Schema can be safely registered.
```

**Output (Incompatible):**

```bash
❌ Schema is NOT compatible!
   Subject: user-events
   Compatibility Mode: backward
   
Compatibility errors:
- Required field 'name' added (breaks backward compatibility)

Cannot register this schema version.
```

## Schema Types

### JSON Schema

Most common for JSON messages.

**Create Schema:**

```json
{
  "type": "object",
  "properties": {
    "event_type": {"type": "string"},
    "timestamp": {"type": "string", "format": "date-time"},
    "user_id": {"type": "string"}
  },
  "required": ["event_type", "timestamp"]
}
```

**Register:**

```bash
danube-cli schema register events \
  --schema-type json_schema \
  --file events-schema.json
```

**Use:**

```bash
# Produce with validation
danube-cli produce \
  -s http://localhost:6650 \
  --schema-subject events \
  -m '{"event_type":"login","timestamp":"2024-01-01T10:00:00Z","user_id":"u123"}'
```

### Avro Schema

For compact binary serialization.

**Create Schema:**

```json
{
  "type": "record",
  "name": "User",
  "namespace": "com.example",
  "fields": [
    {"name": "id", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "age", "type": ["null", "int"], "default": null}
  ]
}
```

**Register:**

```bash
danube-cli schema register users \
  --schema-type avro \
  --file user-schema.avsc
```

**Use:**

```bash
danube-cli produce \
  -s http://localhost:6650 \
  --schema-subject users \
  -m '{"id":"u123","email":"user@example.com","age":25}'
```

### Protobuf Schema

For Google Protocol Buffers.

**Create Schema (message.proto):**

```protobuf
syntax = "proto3";

message User {
  string id = 1;
  string email = 2;
  int32 age = 3;
}
```

**Register:**

```bash
danube-cli schema register users \
  --schema-type protobuf \
  --file message.proto
```

**Use:**

```bash
# Send compiled protobuf binary
danube-cli produce \
  -s http://localhost:6650 \
  --schema-subject users \
  --file compiled-message.bin
```

## Schema Evolution

### Evolution Scenarios

#### Adding Optional Fields (Safe)

**V1:**

```json
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"}
  },
  "required": ["user_id"]
}
```

**V2 (Add optional field):**

```json
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["user_id"]
}
```

✅ **Backward compatible** - Old consumers can read new messages
✅ **Forward compatible** - New consumers can read old messages

#### Removing Optional Fields (Safe)

**V1:**

```json
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "temp_field": {"type": "string"}
  },
  "required": ["user_id"]
}
```

**V2 (Remove optional field):**

```json
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"}
  },
  "required": ["user_id"]
}
```

✅ **Backward compatible** - Old consumers still work

#### Adding Required Fields (Unsafe)

**V1:**

```json
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"}
  },
  "required": ["user_id"]
}
```

**V2 (Add required field):**

```json
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["user_id", "email"]
}
```

❌ **NOT backward compatible** - Old producers can't provide required field

### Safe Evolution Workflow

```bash
# Step 1: Check current schema
danube-cli schema get orders

# Step 2: Create new schema version
cat > orders-v2.json << 'EOF'
{
  "type": "object",
  "properties": {
    "order_id": {"type": "string"},
    "amount": {"type": "number"},
    "currency": {"type": "string", "default": "USD"}
  },
  "required": ["order_id", "amount"]
}
EOF

# Step 3: Check compatibility
danube-cli schema check orders \
  --schema-type json_schema \
  --file orders-v2.json

# Step 4: If compatible, register
danube-cli schema register orders \
  --schema-type json_schema \
  --file orders-v2.json

# Step 5: Verify versions
danube-cli schema versions orders
```

## Compatibility Modes

Compatibility modes control how schemas can evolve.

> **⚠️ Note:** Setting compatibility mode is an **admin-only** operation using `danube-admin-cli`. Clients can only **check** compatibility, not set it.

### Backward (Default)

New schema can read data written with old schema.

**Use when:** Consumers are upgraded before producers

```bash
# Check backward compatibility (client operation)
danube-cli schema check orders \
  --schema-type json_schema \
  --file orders-v2.json

# Set compatibility mode (admin-only - use danube-admin-cli)
# danube-admin-cli schemas set-compatibility orders --mode backward
```

**Allowed changes:**

- ✅ Add optional fields
- ✅ Remove required fields

**Forbidden changes:**

- ❌ Add required fields
- ❌ Remove optional fields

### Forward

Old schema can read data written with new schema.

**Use when:** Producers are upgraded before consumers

**Allowed changes:**

- ✅ Remove optional fields
- ✅ Add required fields

**Forbidden changes:**

- ❌ Add optional fields
- ❌ Remove required fields

### Full

Both backward and forward compatible.

**Use when:** Consumers and producers upgrade independently

**Allowed changes:**

- ✅ Add optional fields with defaults
- ✅ Remove optional fields

**Forbidden changes:**

- ❌ Add required fields
- ❌ Remove required fields
- ❌ Change field types

### None

No compatibility checking.

**Use when:** Breaking changes are acceptable

⚠️ **Warning:** Can break consumers!

## Complete Workflows

### Workflow 1: New Schema from Scratch

```bash
# Step 1: Create schema file
cat > payment-events.json << 'EOF'
{
  "type": "object",
  "properties": {
    "payment_id": {"type": "string"},
    "amount": {"type": "number", "minimum": 0},
    "currency": {"type": "string"},
    "status": {"type": "string", "enum": ["pending", "completed", "failed"]}
  },
  "required": ["payment_id", "amount", "currency", "status"]
}
EOF

# Step 2: Register schema
danube-cli schema register payment-events \
  --schema-type json_schema \
  --file payment-events.json

# Step 3: Verify registration
danube-cli schema get payment-events

# Step 4: Start producer with schema
danube-cli produce \
  -s http://localhost:6650 \
  -t /production/payments \
  --schema-subject payment-events \
  -m '{"payment_id":"pay_123","amount":99.99,"currency":"USD","status":"completed"}'

# Step 5: Start consumer (automatic schema fetching and validation)
danube-cli consume \
  -s http://localhost:6650 \
  -t /production/payments \
  -m payment-processor
# Consumer automatically fetches schema using schema_id from message metadata
```

### Workflow 2: Schema Evolution

```bash
# Step 1: Check current schema
danube-cli schema get user-events
danube-cli schema versions user-events

# Step 2: Create new schema version
cat > user-events-v2.json << 'EOF'
{
  "type": "object",
  "properties": {
    "user_id": {"type": "string"},
    "event": {"type": "string"},
    "timestamp": {"type": "string"},
    "metadata": {"type": "object"}
  },
  "required": ["user_id", "event", "timestamp"]
}
EOF

# Step 3: Check compatibility
danube-cli schema check user-events \
  --schema-type json_schema \
  --file user-events-v2.json

# Step 4: Register if compatible
danube-cli schema register user-events \
  --schema-type json_schema \
  --file user-events-v2.json

# Step 5: Verify new version
danube-cli schema versions user-events

# Step 6: Test with new schema
danube-cli produce \
  -s http://localhost:6650 \
  --schema-subject user-events \
  -m '{"user_id":"u123","event":"login","timestamp":"2024-01-01T10:00:00Z","metadata":{"ip":"127.0.0.1"}}'
```

## Troubleshooting

### Schema Not Found

```bash
# Check if schema is registered
danube-cli schema get my-subject

# If not found, register it
danube-cli schema register my-subject --schema-type json_schema --file schema.json
```

### Validation Failures

```bash
# Get current schema
danube-cli schema get my-subject --output json

# Verify your message matches the schema
# Check required fields, types, formats
```

### Compatibility Issues

```bash
# Check what compatibility mode is set
danube-cli schema get my-subject

# Check compatibility
danube-cli schema check my-subject \
  --schema-type json_schema \
  --file new-schema.json
```

## Client vs Admin Operations

### What Clients Can Do (danube-cli)

✅ **Register schemas** - Add new schemas or versions
✅ **Get schema details** - Fetch schema information
✅ **List versions** - View version history
✅ **Check compatibility** - Validate before registering
✅ **Choose schema version** - Producers can pin to specific versions
✅ **Auto-register schemas** - Register during production

### What Requires Admin (danube-admin-cli)

❌ **Set compatibility mode** - Governance control (use `danube-admin-cli schemas set-compatibility`)
❌ **Configure topic schemas** - Topic-level validation policies (use `danube-admin-cli topics configure-schema`)
❌ **Delete schemas** - Dangerous operation (use `danube-admin-cli schemas delete`)

**See Also:**

- [Admin Schema Registry Guide](../danube_admin/admin_cli/schema_registry.md) - For admin-only operations
- [Admin Topics Guide](../danube_admin/admin_cli/topics.md) - For topic schema configuration

---

## Consumer Schema Fetching

Consumers automatically fetch and validate schemas:

```bash
danube-cli consume -s http://localhost:6650 -t /default/events -m my-sub
```

**How it works:**

1. Consumer receives message with `schema_id` in metadata
2. Automatically fetches schema from registry using `schema_id`
3. Caches schema for performance
4. Validates JSON messages against schema (if JSON Schema type)
5. Pretty-prints validated JSON messages

**Benefits:**

- No manual schema configuration needed
- Always uses the exact schema the producer used
- Handles schema evolution automatically
- Efficient caching reduces registry calls

---

## JSON Output for Automation

All schema commands support JSON output:

```bash
# Get schema as JSON
danube-cli schema get user-events --output json | jq .

# List versions as JSON
danube-cli schema versions user-events --output json | jq .

# Check compatibility with JSON output
danube-cli schema check user-events \
  --schema-type json_schema \
  --file new-schema.json \
  --output json | jq .
```
