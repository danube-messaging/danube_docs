# Schema Registry Management

Manage schemas for data validation and evolution in Danube.

## Overview

The Schema Registry provides centralized schema management for your Danube topics. It enables:

- **Type Safety**: Validate messages against defined schemas
- **Schema Evolution**: Track and manage schema versions over time
- **Compatibility Checking**: Ensure new schemas don't break existing consumers
- **Documentation**: Schemas serve as living documentation for your data

### Why Use Schema Registry?

**Without Schema Registry:**

```bash
# No validation - anything goes
producer.send('{"nam": "John"}')  # Typo: "nam" instead of "name"
# Message accepted ❌ - consumers break
```

**With Schema Registry:**

```bash
# Schema enforces structure
producer.send('{"nam": "John"}')  # Typo detected
# Error: Field 'name' is required ✅
```

---

## Commands

### Register a Schema

Register a new schema or create a new version of an existing schema.

```bash
danube-admin-cli schemas register <SUBJECT> [OPTIONS]
```

#### Basic Schema Registration

**From File (Recommended):**

```bash
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file schemas/user-events.json
```

**Inline Schema:**

```bash
danube-admin-cli schemas register simple-events \
  --schema-type json_schema \
  --schema '{"type": "object", "properties": {"id": {"type": "string"}}}'
```

**Example Output:**

```bash
✅ Registered new schema version
Subject: user-events
Schema ID: 12345
Version: 1
Fingerprint: sha256:abc123...
```

#### With Metadata

**Add Description and Tags:**

```bash
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file schemas/user-events.json \
  --description "User registration and login events" \
  --tags users \
  --tags authentication \
  --tags analytics
```

**Tags for Organization:**

- `production`, `staging`, `development` - Environment
- `team-analytics`, `team-platform` - Ownership
- `pii`, `sensitive` - Data classification
- `v1`, `v2` - Version tracking
- `deprecated` - Lifecycle status

#### Schema Types

| Type | Description | Use Cases | Extension |
|------|-------------|-----------|-----------|
| `json_schema` | JSON Schema (Draft 7) | Web APIs, JavaScript/TypeScript | `.json` |
| `avro` | Apache Avro | Big data, Kafka integration | `.avsc` |
| `protobuf` | Protocol Buffers | gRPC, high performance | `.proto` |
| `string` | Plain string (no validation) | Simple text messages | `.txt` |
| `bytes` | Raw bytes (no validation) | Binary data | - |

#### JSON Schema Example

**schemas/user-events.json:**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "UserEvent",
  "type": "object",
  "required": ["event_type", "user_id", "timestamp"],
  "properties": {
    "event_type": {
      "type": "string",
      "enum": ["login", "logout", "register"]
    },
    "user_id": {
      "type": "string",
      "format": "uuid"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "ip_address": { "type": "string" },
        "user_agent": { "type": "string" }
      }
    }
  }
}
```

**Register:**

```bash
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file schemas/user-events.json \
  --description "User authentication events" \
  --tags users authentication
```

#### Avro Schema Example

**schemas/payment.avsc:**

```json
{
  "type": "record",
  "name": "Payment",
  "namespace": "com.example.payments",
  "fields": [
    {"name": "payment_id", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "currency", "type": "string"},
    {"name": "timestamp", "type": "long"}
  ]
}
```

**Register:**

```bash
danube-admin-cli schemas register payment-events \
  --schema-type avro \
  --file schemas/payment.avsc \
  --description "Payment transaction events"
```

---

### Get a Schema

Retrieve schema details by subject or ID.

```bash
danube-admin-cli schemas get [OPTIONS]
```

#### By Subject (Latest Version)

```bash
# Get latest version
danube-admin-cli schemas get --subject user-events
```

**Example Output:**

```bash
Schema ID: 12345
Version: 2
Subject: user-events
Type: json_schema
Compatibility Mode: BACKWARD
Description: User registration and login events
Tags: users, authentication, analytics

Schema Definition:
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "UserEvent",
  "type": "object",
  "required": ["event_type", "user_id", "timestamp"],
  ...
}
```

#### By Schema ID

```bash
# Get specific schema by ID
danube-admin-cli schemas get --id 12345

# Get specific version of a schema
danube-admin-cli schemas get --id 12345 --version 1
```

#### JSON Output

```bash
danube-admin-cli schemas get --subject user-events --output json
```

**Example JSON Output:**

```json
{
  "schema_id": 12345,
  "version": 2,
  "subject": "user-events",
  "schema_type": "json_schema",
  "schema_definition": "{ ... }",
  "description": "User registration and login events",
  "created_at": 1704067200,
  "created_by": "admin",
  "tags": ["users", "authentication", "analytics"],
  "fingerprint": "sha256:abc123...",
  "compatibility_mode": "BACKWARD"
}
```

---

### List Schema Versions

View all versions for a schema subject.

```bash
danube-admin-cli schemas versions <SUBJECT> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli schemas versions user-events
```

**Example Output:**

```
Versions for subject 'user-events':
  Version 1: schema_id=12344, fingerprint=sha256:old123...
    Created by: alice
    Description: Initial schema
  Version 2: schema_id=12345, fingerprint=sha256:abc123...
    Created by: bob
    Description: Added email field
  Version 3: schema_id=12346, fingerprint=sha256:new456...
    Created by: charlie
    Description: Made phone optional
```

**JSON Output:**

```bash
danube-admin-cli schemas versions user-events --output json
```

**Example JSON:**

```json
[
  {
    "version": 1,
    "schema_id": 12344,
    "created_at": 1704067200,
    "created_by": "alice",
    "description": "Initial schema",
    "fingerprint": "sha256:old123..."
  },
  {
    "version": 2,
    "schema_id": 12345,
    "created_at": 1704153600,
    "created_by": "bob",
    "description": "Added email field",
    "fingerprint": "sha256:abc123..."
  }
]
```

---

### Check Schema Compatibility

Verify if a new schema is compatible with existing versions.

```bash
danube-admin-cli schemas check <SUBJECT> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli schemas check user-events \
  --file schemas/user-events-v2.json \
  --schema-type json_schema
```

**Example Output (Compatible):**

```bash
✅ Schema is compatible with subject 'user-events'
```

**Example Output (Incompatible):**

```bash
❌ Schema is NOT compatible with subject 'user-events'

Compatibility errors:
  - Field 'user_id' was removed (breaking change)
  - Field 'email' is now required (breaking change for existing consumers)
```

#### Override Compatibility Mode

```bash
# Check with specific mode (overrides subject's default)
danube-admin-cli schemas check user-events \
  --file schemas/user-events-v2.json \
  --schema-type json_schema \
  --mode full
```

#### Workflow Example

```bash
# Step 1: Create new schema version
vim schemas/user-events-v2.json

# Step 2: Check compatibility BEFORE registering
danube-admin-cli schemas check user-events \
  --file schemas/user-events-v2.json \
  --schema-type json_schema

# Step 3: If compatible, register it
if [ $? -eq 0 ]; then
  danube-admin-cli schemas register user-events \
    --schema-type json_schema \
    --file schemas/user-events-v2.json \
    --description "Added email field"
fi
```

---

### Get Compatibility Mode

Retrieve the current compatibility mode for a schema subject.

```bash
danube-admin-cli schemas get-compatibility <SUBJECT> [OPTIONS]
```

**Basic Usage:**

```bash
# Get compatibility mode for a subject
danube-admin-cli schemas get-compatibility user-events
```

**Example Output:**

```
Subject: user-events
Compatibility Mode: BACKWARD
```

**JSON Output:**

```bash
danube-admin-cli schemas get-compatibility user-events --output json
```

**Example JSON:**

```json
{
  "subject": "user-events",
  "compatibility_mode": "BACKWARD"
}
```

**Use Cases:**

- Verify current compatibility settings before making changes
- Audit schema governance across subjects
- Automation scripts that need to check compatibility mode

---

### Set Compatibility Mode

Configure how schema evolution is enforced.

```bash
danube-admin-cli schemas set-compatibility <SUBJECT> --mode <MODE>
```

**Compatibility Modes:**

| Mode | Description | Allows | Use Case |
|------|-------------|--------|----------|
| `none` | No compatibility checks | Any changes | Development, testing |
| `backward` | New schema can read old data | Add optional fields, remove fields | Most common - new consumers, old producers |
| `forward` | Old schema can read new data | Remove optional fields, add fields | New producers, old consumers |
| `full` | Both backward and forward | Add/remove optional fields only | Strict compatibility |

**Examples:**

```bash
# Set backward compatibility (most common)
danube-admin-cli schemas set-compatibility user-events --mode backward

# Set full compatibility (strict)
danube-admin-cli schemas set-compatibility payment-events --mode full

# Disable compatibility (development only)
danube-admin-cli schemas set-compatibility test-events --mode none
```

**Example Output:**

```
✅ Compatibility mode set for subject 'user-events'
Mode: BACKWARD
```

#### Backward Compatibility (Recommended)

**Allows:**

- ✅ Adding optional fields
- ✅ Removing fields
- ✅ Adding enum values

**Prevents:**

- ❌ Removing required fields
- ❌ Changing field types
- ❌ Making optional fields required

**Example:**

```bash
# Old schema
{
  "properties": {
    "user_id": {"type": "string"},
    "name": {"type": "string"}
  },
  "required": ["user_id", "name"]
}

# New schema (backward compatible)
{
  "properties": {
    "user_id": {"type": "string"},
    "name": {"type": "string"},
    "email": {"type": "string"}  // ✅ Added optional field
  },
  "required": ["user_id", "name"]
}
```

#### Forward Compatibility

**Allows:**

- ✅ Removing optional fields
- ✅ Adding fields

**Prevents:**

- ❌ Adding required fields
- ❌ Changing field types

#### Full Compatibility (Strictest)

**Allows:**

- ✅ Only changes that are both backward AND forward compatible
- ✅ Adding optional fields with defaults
- ✅ Removing optional fields

**Prevents:**

- ❌ Most breaking changes
- ❌ Required field modifications

---

### Delete Schema Version

Remove a specific version of a schema.

```bash
danube-admin-cli schemas delete <SUBJECT> --version <VERSION> --confirm
```

**Basic Usage:**

```bash
danube-admin-cli schemas delete user-events --version 1 --confirm
```

**Example Output:**

```
✅ Deleted version 1 of subject 'user-events'
```

**⚠️ Important Notes:**

1. **Requires Confirmation**: Must use `--confirm` flag to prevent accidents
2. **Cannot Delete Active**: Cannot delete version currently used by topics
3. **No Undo**: Deletion is permanent
4. **Version History**: Gaps in version numbers are normal after deletion

**Safety Checks:**

```bash
# Step 1: List all versions
danube-admin-cli schemas versions user-events

# Step 2: Check which version is active
danube-admin-cli topics describe /production/events | grep "Version:"

# Step 3: Only delete if not active and confirmed safe
danube-admin-cli schemas delete user-events --version 1 --confirm
```
