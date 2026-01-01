# Consumer Guide

Learn how to consume messages from Danube topics. 📥

## Table of Contents

- [Basic Usage](#basic-usage)
- [Subscription Types](#subscription-types)
- [Schema-Based Consumption](#schema-based-consumption)
- [Advanced Patterns](#advanced-patterns)
- [Practical Examples](#practical-examples)

## Basic Usage

### Simple Consumption

```bash
danube-cli consume \
  --service-addr http://localhost:6650 \
  --subscription my-subscription
```

Consumes from the default topic `/default/test_topic` with a shared subscription.

### Custom Topic

```bash
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/orders \
  -m order-processors
```

### Using Short Flags

```bash
danube-cli consume -s http://localhost:6650 -t /default/events -m event-sub
```

### Custom Consumer Name

```bash
danube-cli consume \
  -s http://localhost:6650 \
  -n order-consumer-1 \
  -m order-subscription
```

## Subscription Types

### Shared (Default)

Multiple consumers share message processing. Messages are distributed across consumers.

```bash
# Consumer 1
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/events \
  -m shared-sub \
  --sub-type shared

# Consumer 2 (run in parallel)
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/events \
  -m shared-sub \
  --sub-type shared
```

**Use case:** Load balancing, parallel processing

### Exclusive

Only one consumer can be active at a time. Ensures ordered processing.

```bash
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/orders \
  -m exclusive-sub \
  --sub-type exclusive
```

**Use case:** Ordered message processing, single consumer workflows

### Failover

Multiple consumers but only one is active. Others act as standby.

```bash
# Primary consumer
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/critical \
  -m ha-sub \
  --sub-type fail-over

# Standby consumer (automatically takes over if primary fails)
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/critical \
  -m ha-sub \
  --sub-type fail-over
```

**Use case:** High availability, ordered processing with failover

## Schema-Based Consumption

### Auto-Detection

The consumer automatically detects and validates against the topic's schema:

```bash
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/orders \
  -m order-sub
```

Output shows schema validation:

```bash
🔍 Checking for schema associated with topic...
✅ Topic has schema: orders (json_schema, version 1)
📥 Consuming with schema validation...
```

### Without Schema

If the topic has no schema:

```bash
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/raw-data \
  -m raw-sub
```

Output:

```
🔍 Checking for schema associated with topic...
ℹ️  Topic has no schema - consuming raw bytes
```

### Schema Evolution

Consumers automatically handle schema evolution:

```bash
# Producer sends message with v1 schema
danube-cli produce \
  -t /default/users \
  --schema-subject users \
  -m '{"user_id":"123","name":"Alice"}'

# Schema evolves to v2 (adds optional "email" field)
danube-cli schema register users \
  --schema-type json_schema \
  --file users-v2.json

# Consumer automatically uses latest schema
danube-cli consume \
  -t /default/users \
  -m user-processors
```

## Advanced Patterns

### Fan-Out Pattern

Multiple subscriptions on the same topic for different purposes:

```bash
# Subscription 1: Process orders
danube-cli consume -t /default/orders -m order-processing &

# Subscription 2: Analytics
danube-cli consume -t /default/orders -m order-analytics &

# Subscription 3: Notifications
danube-cli consume -t /default/orders -m order-notifications &
```

### Worker Pool Pattern

Multiple consumers in a shared subscription for parallel processing:

```bash
# Start 4 workers
for i in {1..4}; do
  danube-cli consume \
    -s http://localhost:6650 \
    -t /default/tasks \
    -n "worker-$i" \
    -m task-workers \
    --sub-type shared &
done
```

### Multi-Stage Pipeline

Process messages through multiple stages:

```bash
# Stage 1: Consume from source
danube-cli consume -t /pipeline/raw-events -m stage1 &

# Stage 2: Consume from enriched
danube-cli consume -t /pipeline/enriched-events -m stage2 &

# Stage 3: Consume from processed
danube-cli consume -t /pipeline/processed-events -m stage3 &
```

## Practical Examples

### E-Commerce Order Processing

```bash
# Multiple workers processing orders
for i in {1..3}; do
  danube-cli consume \
    -s http://localhost:6650 \
    -t /default/orders \
    -n "order-worker-$i" \
    -m order-processors \
    --sub-type shared &
done
```

### Real-Time Analytics

```bash
# Exclusive consumer for ordered analytics
danube-cli consume \
  -s http://localhost:6650 \
  -t /analytics/events \
  -m analytics-processor \
  --sub-type exclusive
```

### Event-Driven Microservices

```bash
# Service 1: User service
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/user-events \
  -m user-service &

# Service 2: Email service
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/user-events \
  -m email-service &

# Service 3: Analytics service
danube-cli consume \
  -s http://localhost:6650 \
  -t /default/user-events \
  -m analytics-service &
```

### IoT Data Collection

```bash
# Consume sensor data with high availability
danube-cli consume \
  -s http://localhost:6650 \
  -t /iot/sensors \
  -m sensor-processor \
  --sub-type fail-over
```

### Log Aggregation

```bash
# Consume logs from multiple sources
danube-cli consume \
  -s http://localhost:6650 \
  -t /logs/application \
  -m log-aggregator \
  --sub-type shared
```

### Notification Service

```bash
# Process notifications with worker pool
for i in {1..5}; do
  danube-cli consume \
    -s http://localhost:6650 \
    -t /notifications/queue \
    -n "notification-worker-$i" \
    -m notification-processors \
    --sub-type shared &
done
```

## Message Output Format

### Text Messages

```bash
Received message: Hello, World!
Size: 13 bytes, Total received: 13 bytes
```

### JSON Messages

```bash
Received message: {"user_id":"123","action":"login"}
Size: 35 bytes, Total received: 35 bytes
```

### Binary Data

```bash
Received message: [binary data - 1024 bytes]
Size: 1024 bytes, Total received: 1024 bytes
```

### With Schema Validation

```bash
✅ Message validated against schema 'orders' (version 1)
Received message: {"order_id":"ord_123","amount":99.99}
Size: 42 bytes, Total received: 42 bytes
```

## Command Reference

### All Consumer Flags

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--service-addr` | `-s` | Broker URL | Required |
| `--topic` | `-t` | Topic name | `/default/test_topic` |
| `--subscription` | `-m` | Subscription name | Required |
| `--consumer-name` | `-n` | Consumer name | `consumer_pubsub` |
| `--sub-type` | - | Subscription type | `shared` |

### Subscription Types

| Type | Description | Use Case |
|------|-------------|----------|
| `shared` | Load balanced across consumers | Parallel processing |
| `exclusive` | Single active consumer | Ordered processing |
| `fail-over` | Active/standby with failover | HA ordered processing |

## Scripting with Consumers

### Basic Shell Script

```bash
#!/bin/bash

danube-cli consume \
  -s http://localhost:6650 \
  -t /default/events \
  -m event-processor | \
while IFS= read -r line; do
  echo "Processing: $line"
  # Your processing logic here
done
```

### Filter Messages

```bash
#!/bin/bash

danube-cli consume \
  -s http://localhost:6650 \
  -t /default/events \
  -m event-filter | \
while IFS= read -r line; do
  # Process only specific messages
  if echo "$line" | grep -q "error"; then
    echo "Error detected: $line"
    # Send alert
  fi
done
```
