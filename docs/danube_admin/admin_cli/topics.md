# Topics Management

Create and manage topics in your Danube cluster.

## Overview

Topics are the fundamental messaging primitive in Danube. They provide:

- Named channels for publishing and subscribing to messages
- Schema enforcement via Schema Registry
- Partitioning for horizontal scaling
- Reliable or non-reliable delivery modes

## Commands

### List Topics

View topics in a namespace or on a specific broker.

```bash
danube-admin-cli topics list [OPTIONS]
```

**By Namespace:**

```bash
# List all topics in a namespace
danube-admin-cli topics list --namespace default

# JSON output for automation
danube-admin-cli topics list --namespace default --output json
```

**By Broker:**

```bash
# List topics on a specific broker
danube-admin-cli topics list --broker broker-001

# JSON output
danube-admin-cli topics list --broker broker-001 --output json
```

**Example Output (Plain Text):**

```bash
Topics in namespace 'default':
  /default/user-events
  /default/payment-transactions
  /default/analytics-stream
```

**Example Output (JSON):**

```json
[
  "/default/user-events",
  "/default/payment-transactions",
  "/default/analytics-stream"
]
```

---

### Create a Topic

Create a new topic with optional schema validation.

```bash
danube-admin-cli topics create <TOPIC> [OPTIONS]
```

#### Basic Topic Creation

**Simple Topic (No Schema):**

```bash
# Create topic without schema
danube-admin-cli topics create /default/logs

# Create with reliable delivery
danube-admin-cli topics create /default/events --dispatch-strategy reliable
```

**Using Namespace Flag:**

```bash
# Specify namespace separately
danube-admin-cli topics create my-topic --namespace default

# Equivalent to
danube-admin-cli topics create /default/my-topic
```

#### Schema-Validated Topics

**With Schema Registry:**

```bash
# First, register a schema
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file user-schema.json

# Create topic with schema validation
danube-admin-cli topics create /default/user-events \
  --schema-subject user-events \
  --dispatch-strategy reliable
```

**Example Output:**

```
✅ Topic created: /default/user-events
   Schema subject: user-events
```

#### Partitioned Topics

**Create with Partitions:**

```bash
# Create partitioned topic (3 partitions)
danube-admin-cli topics create /default/high-throughput \
  --partitions 3

# With schema and partitions
danube-admin-cli topics create /default/user-events \
  --partitions 5 \
  --schema-subject user-events \
  --dispatch-strategy reliable
```

**Example Output:**

```bash
✅ Partitioned topic created: /default/high-throughput
   Schema subject: user-events
   Partitions: 5
```

#### Options Reference

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `--namespace` | Namespace (if not in topic path) | - | `--namespace default` |
| `--partitions` | Number of partitions | 1 | `--partitions 3` |
| `--schema-subject` | Schema subject from registry | None | `--schema-subject user-events` |
| `--dispatch-strategy` | Delivery mode | `non_reliable` | `--dispatch-strategy reliable` |

**Dispatch Strategies:**

- **non_reliable**: Fast, at-most-once delivery (fire-and-forget)
  - Use for: Logs, metrics, non-critical events
  - Pros: Low latency, high throughput
  - Cons: Messages may be lost

- **reliable**: Slower, at-least-once delivery (with acknowledgments)
  - Use for: Transactions, orders, critical events
  - Pros: Guaranteed delivery
  - Cons: Higher latency

---

### Describe a Topic

View detailed information about a topic including schema and subscriptions.

```bash
danube-admin-cli topics describe <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli topics describe /default/user-events
```

**Output Formats:**

```bash
# Plain text (default) - human-readable
danube-admin-cli topics describe /default/user-events

# JSON format - for automation
danube-admin-cli topics describe /default/user-events --output json
```

**Example Output (Plain Text):**

```bash
Topic: /default/user-events
Broker ID: broker-001
Delivery: Reliable

📋 Schema Registry:
  Subject: user-events
  Schema ID: 12345
  Version: 2
  Type: json_schema
  Compatibility: BACKWARD

Subscriptions: ["analytics-consumer", "audit-logger"]
```

**Example Output (JSON):**

```json
{
  "topic": "/default/user-events",
  "broker_id": "broker-001",
  "delivery": "Reliable",
  "schema_subject": "user-events",
  "schema_id": 12345,
  "schema_version": 2,
  "schema_type": "json_schema",
  "compatibility_mode": "BACKWARD",
  "subscriptions": [
    "analytics-consumer",
    "audit-logger"
  ]
}
```

**Without Schema:**

```bash
Topic: /default/logs
Broker ID: broker-002
Delivery: NonReliable

📋 Schema: None

Subscriptions: []
```

---

### List Subscriptions

View all active subscriptions for a topic.

```bash
danube-admin-cli topics subscriptions <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli topics subscriptions /default/user-events
```

**Output Formats:**

```bash
# Plain text
danube-admin-cli topics subscriptions /default/user-events

# JSON format
danube-admin-cli topics subscriptions /default/user-events --output json
```

**Example Output:**

```bash
Subscriptions: ["consumer-1", "consumer-2", "analytics-team"]
```

---

### Delete a Topic

Permanently remove a topic and all its messages.

```bash
danube-admin-cli topics delete <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli topics delete /default/old-topic
```

**With Namespace:**

```bash
danube-admin-cli topics delete old-topic --namespace default
```

**Example Output:**

```bash
✅ Topic deleted: /default/old-topic
```

**⚠️ Important Warnings:**

1. **Data Loss**: All messages in the topic are permanently deleted
2. **No Confirmation**: Operation is immediate and irreversible
3. **Active Subscriptions**: All consumers will be disconnected
4. **Schema Intact**: The schema in the registry is NOT deleted

**Safety Checklist:**

```bash
# 1. Check subscriptions
danube-admin-cli topics subscriptions /default/my-topic

# 2. Verify topic details
danube-admin-cli topics describe /default/my-topic

# 3. Backup if needed (application-level)

# 4. Delete topic
danube-admin-cli topics delete /default/my-topic
```

---

### Unsubscribe

Remove a specific subscription from a topic.

```bash
danube-admin-cli topics unsubscribe <TOPIC> --subscription <NAME> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli topics unsubscribe /default/user-events \
  --subscription old-consumer
```

**With Namespace:**

```bash
danube-admin-cli topics unsubscribe my-topic \
  --namespace default \
  --subscription old-consumer
```

**Example Output:**

```bash
✅ Unsubscribed: true
```

**Use Cases:**

- Remove inactive consumers
- Clean up test subscriptions
- Force consumer reconnection

---

### Unload a Topic

Gracefully unload a topic from its current broker.

```bash
danube-admin-cli topics unload <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin-cli topics unload /default/user-events
```

**Example Output:**

``` bash
✅ Topic unloaded: /default/user-events
```

**Use Cases:**

- Rebalance topics across brokers
- Prepare for broker maintenance
- Move topic to different broker

---

## Schema Configuration Commands (Admin-Only)

### Configure Topic Schema

Configure complete schema settings for a topic including schema subject, validation policy, and payload validation (admin-only operation).

```bash
danube-admin-cli topics configure-schema <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
# Configure topic with schema and strict validation
danube-admin-cli topics configure-schema /default/user-events \
  --subject user-events \
  --validation-policy enforce \
  --enable-payload-validation
```

**Options:**

| Option | Required | Values | Description |
|--------|----------|--------|-------------|
| `--subject` | Yes | String | Schema subject name from registry |
| `--validation-policy` | No | `none`, `warn`, `enforce` | Validation strictness (default: `none`) |
| `--enable-payload-validation` | No | Flag | Enable deep payload validation |
| `--namespace` | No | String | Namespace if not in topic path |

**Validation Policies:**

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `none` | No validation | Development topics, unstructured data |
| `warn` | Validate and log errors, accept messages | Monitoring/debugging production |
| `enforce` | Reject invalid messages | Production requiring strict data quality |

**Examples:**

```bash
# Production: Strict validation
danube-admin-cli topics configure-schema /production/orders \
  --subject order-events \
  --validation-policy enforce \
  --enable-payload-validation

# Staging: Warn on validation errors
danube-admin-cli topics configure-schema /staging/orders \
  --subject order-events \
  --validation-policy warn \
  --enable-payload-validation

# Development: No validation
danube-admin-cli topics configure-schema /dev/orders \
  --subject order-events \
  --validation-policy none

# Using namespace flag
danube-admin-cli topics configure-schema my-topic \
  --namespace production \
  --subject events \
  --validation-policy enforce
```

**Example Output:**

```
✅ Schema configuration set for topic '/production/orders'
   Schema Subject: order-events
   Validation Policy: ENFORCE
   Payload Validation: ENABLED
```

**⚠️ Important Notes:**

1. **Admin-Only**: Only administrators can change topic schema configuration
2. **Schema Must Exist**: The schema subject must be registered before configuring
3. **Overrides First Producer**: This overrides the schema set by first producer
4. **Affects All Producers**: All producers must use the configured schema subject

---

### Set Validation Policy

Update the validation policy for a topic without changing its schema subject (admin-only operation).

```bash
danube-admin-cli topics set-validation-policy <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
# Change validation policy to warn
danube-admin-cli topics set-validation-policy /production/events \
  --policy warn \
  --enable-payload-validation
```

**Options:**

| Option | Required | Values | Description |
|--------|----------|--------|-------------|
| `--policy` | Yes | `none`, `warn`, `enforce` | Validation policy to set |
| `--enable-payload-validation` | No | Flag | Enable/disable payload validation |
| `--namespace` | No | String | Namespace if not in topic path |

**Examples:**

```bash
# Enable strict validation for production
danube-admin-cli topics set-validation-policy /production/events \
  --policy enforce \
  --enable-payload-validation

# Switch to warn mode for debugging
danube-admin-cli topics set-validation-policy /production/events \
  --policy warn \
  --enable-payload-validation

# Disable validation temporarily
danube-admin-cli topics set-validation-policy /production/events \
  --policy none
```

**Example Output:**

```
✅ Validation policy updated for topic '/production/events'
   Policy: WARN
   Payload Validation: ENABLED
```

**Use Cases:**

- **Production Issue**: Temporarily switch to `warn` mode to allow messages through while debugging
- **Gradual Rollout**: Start with `warn`, monitor errors, then switch to `enforce`
- **Performance Tuning**: Disable payload validation if schema ID matching is sufficient
- **Emergency Override**: Switch to `none` if validation is blocking critical messages

---

### Get Schema Configuration

Retrieve the current schema configuration for a topic.

```bash
danube-admin-cli topics get-schema-config <TOPIC> [OPTIONS]
```

**Basic Usage:**

```bash
# Get schema configuration
danube-admin-cli topics get-schema-config /production/events
```

**Output Formats:**

```bash
# Plain text (default)
danube-admin-cli topics get-schema-config /production/events

# JSON format for automation
danube-admin-cli topics get-schema-config /production/events --output json
```

**Example Output (Plain Text):**

```
Topic: /production/events
Schema Subject: user-events
Validation Policy: ENFORCE
Payload Validation: ENABLED
Cached Schema ID: 12345
```

**Example Output (No Schema Configured):**

```
Topic: /production/logs
No schema configured for this topic
```

**Example Output (JSON):**

```json
{
  "topic": "/production/events",
  "schema_subject": "user-events",
  "validation_policy": "ENFORCE",
  "enable_payload_validation": true,
  "schema_id": 12345
}
```

**Use Cases:**

- Verify schema configuration after changes
- Audit which topics have schema validation enabled
- Automation scripts that need to query topic settings
- Troubleshoot validation issues by checking current configuration

---

## Common Workflows

### 1. Create Topic with Schema Validation

**Step-by-step:**

```bash
# Step 1: Register schema
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file schemas/user-events.json \
  --description "User event schema" \
  --tags users analytics

# Step 2: Verify schema
danube-admin-cli schemas get --subject user-events

# Step 3: Create topic
danube-admin-cli topics create /production/user-events \
  --schema-subject user-events \
  --dispatch-strategy reliable \
  --partitions 5

# Step 4: Verify topic
danube-admin-cli topics describe /production/user-events
```

### 2. Schema Evolution

**Update schema for existing topic:**

```bash
# Step 1: Check current compatibility mode
danube-admin-cli schemas get --subject user-events

# Step 2: Test new schema compatibility
danube-admin-cli schemas check user-events \
  --file schemas/user-events-v2.json \
  --schema-type json_schema

# Step 3: If compatible, register new version
danube-admin-cli schemas register user-events \
  --schema-type json_schema \
  --file schemas/user-events-v2.json \
  --description "Added email field"

# Step 4: Verify topic picked up new version
danube-admin-cli topics describe /production/user-events
```

### 3. Multi-Environment Deployment

**Create same topics across environments:**

```bash
# Production
danube-admin-cli topics create /production/user-events \
  --schema-subject user-events \
  --dispatch-strategy reliable \
  --partitions 10

# Staging
danube-admin-cli topics create /staging/user-events \
  --schema-subject user-events \
  --dispatch-strategy reliable \
  --partitions 3

# Development
danube-admin-cli topics create /development/user-events \
  --schema-subject user-events-dev \
  --partitions 1
```

### 4. Topic Migration

**Move topic to new namespace:**

```bash
# Step 1: Create new namespace
danube-admin-cli namespaces create new-namespace

# Step 2: Get old topic schema
OLD_SCHEMA=$(danube-admin-cli topics describe /old/topic --output json | jq -r '.schema_subject')

# Step 3: Create new topic with same schema
danube-admin-cli topics create /new-namespace/topic \
  --schema-subject $OLD_SCHEMA \
  --dispatch-strategy reliable

# Step 4: Migrate consumers (application-level)

# Step 5: Delete old topic
danube-admin-cli topics delete /old/topic
```

### 5. Admin Schema Configuration (Production)

**Configure validation policies for existing topics:**

```bash
# Step 1: Register schema with full compatibility
danube-admin-cli schemas register order-events \
  --schema-type json_schema \
  --file schemas/orders.json \
  --description "Order transaction schema"

danube-admin-cli schemas set-compatibility order-events --mode full

# Step 2: Create topic (no schema initially)
danube-admin-cli topics create /production/orders \
  --dispatch-strategy reliable \
  --partitions 5

# Step 3: Configure schema with strict validation (admin-only)
danube-admin-cli topics configure-schema /production/orders \
  --subject order-events \
  --validation-policy enforce \
  --enable-payload-validation

# Step 4: Verify configuration
danube-admin-cli topics get-schema-config /production/orders

# Step 5: Monitor for issues, adjust if needed
# If issues found, temporarily switch to warn mode
danube-admin-cli topics set-validation-policy /production/orders \
  --policy warn \
  --enable-payload-validation

# Step 6: After fixes, re-enable strict validation
danube-admin-cli topics set-validation-policy /production/orders \
  --policy enforce \
  --enable-payload-validation
```

## Related Commands

### Schema Management
- `danube-admin-cli schemas register` - Register schemas for validation
- `danube-admin-cli schemas get` - View schema details
- `danube-admin-cli schemas get-compatibility` - Get compatibility mode
- `danube-admin-cli schemas set-compatibility` - Set compatibility mode
- `danube-admin-cli schemas delete` - Delete schema versions

### Topic Schema Configuration (Admin)
- `danube-admin-cli topics configure-schema` - Configure topic schema settings
- `danube-admin-cli topics set-validation-policy` - Update validation policy
- `danube-admin-cli topics get-schema-config` - Get topic schema configuration

### Cluster Management
- `danube-admin-cli namespaces create` - Create namespaces for topics
- `danube-admin-cli brokers list` - View broker topology
