# Producer Guide

Learn how to produce messages to Danube topics. 📤

## Table of Contents

- [Basic Usage](#basic-usage)
- [Message Content Types](#message-content-types)
- [Multiple Messages](#multiple-messages)
- [Message Attributes](#message-attributes)
- [Schema-Based Production](#schema-based-production)
- [Partitioned Topics](#partitioned-topics)
- [Reliable Delivery](#reliable-delivery)

## Basic Usage

### Simple Message

```bash
danube-cli produce \
  --service-addr http://localhost:6650 \
  --message "Hello, World!"
```

Sends one message to the default topic `/default/test_topic`.

### Custom Topic

```bash
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/orders \
  -m "Order received"
```

### Using Short Flags

```bash
danube-cli produce -s http://localhost:6650 -t /default/events -m "Quick message"
```

## Message Content Types

### Text Messages

```bash
danube-cli produce -s http://localhost:6650 -m "Simple text message"
```

### JSON Messages

```bash
danube-cli produce \
  -s http://localhost:6650 \
  -m '{"user_id":"123","action":"login","timestamp":"2024-01-15T10:30:00Z"}'
```

### Binary Files

```bash
# Send an image
danube-cli produce -s http://localhost:6650 --file image.png

# Send any binary file
danube-cli produce -s http://localhost:6650 --file data.bin
```

## Multiple Messages

### Send Multiple Times

```bash
danube-cli produce \
  -s http://localhost:6650 \
  -m "Repeated message" \
  --count 10
```

### With Interval

```bash
# Send 100 messages with 500ms delay between each
danube-cli produce \
  -s http://localhost:6650 \
  -m "Message" \
  --count 100 \
  --interval 500
```

### Rapid Fire

```bash
# Minimum interval is 100ms
danube-cli produce \
  -s http://localhost:6650 \
  -m "Fast message" \
  --count 1000 \
  --interval 100
```

## Message Attributes

### Single Attribute

```bash
danube-cli produce \
  -s http://localhost:6650 \
  -m "Alert!" \
  --attributes "priority:high"
```

### Multiple Attributes

```bash
danube-cli produce \
  -s http://localhost:6650 \
  -m "User action" \
  --attributes "user_id:123,region:us-west,priority:high"
```

## Schema-Based Production

### With Pre-Registered Schema (Latest Version)

```bash
# First, register the schema (one time)
danube-cli schema register orders \
  --schema-type json_schema \
  --file order-schema.json

# Produce with schema validation (uses latest version)
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/orders \
  --schema-subject orders \
  -m '{"order_id":"ord_123","amount":99.99,"currency":"USD"}'
```

### Pin to Specific Schema Version

Use a specific version instead of latest (useful for compatibility testing or controlled rollouts):

```bash
# Use version 2 of the schema
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/orders \
  --schema-subject orders \
  --schema-version 2 \
  -m '{"order_id":"ord_456","amount":149.99,"currency":"USD"}'
```

### Use Minimum Schema Version

Require a minimum schema version or newer:

```bash
# Use version 3 or newer
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/orders \
  --schema-subject orders \
  --schema-min-version 3 \
  -m '{"order_id":"ord_789","amount":199.99,"currency":"USD"}'
```

**Version Control Notes:**
- `--schema-subject` alone uses the **latest** version
- `--schema-version` pins to a **specific** version
- `--schema-min-version` uses the specified version **or newer**
- Cannot use both `--schema-version` and `--schema-min-version` together

### Auto-Register Schema

```bash
# Schema will be registered automatically if it doesn't exist
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/events \
  --schema-file event-schema.json \
  --schema-type json_schema \
  -m '{"event":"user_signup","user_id":"user_456"}'
```

### Schema Types

**JSON Schema:**

```bash
danube-cli produce \
  --schema-subject my-schema \
  --schema-type json_schema \
  -m '{"key":"value"}'
```

**Avro:**

```bash
danube-cli produce \
  --schema-subject my-avro-schema \
  --schema-type avro \
  --file message.avro
```

**Protobuf:**

```bash
danube-cli produce \
  --schema-subject my-proto-schema \
  --schema-type protobuf \
  --file message.pb
```

## Partitioned Topics

### Create Partitioned Topic

```bash
# Specify number of partitions
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/events \
  --partitions 8 \
  -m "Partitioned message"
```

### High Throughput Example

```bash
# 16 partitions for parallel processing
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/high-volume \
  --partitions 16 \
  --schema-subject events \
  -m '{"event":"data"}' \
  --count 10000 \
  --interval 100
```

## Reliable Delivery

### Enable Reliable Delivery

```bash
# Messages are persisted to disk before acknowledgment
danube-cli produce \
  -s http://localhost:6650 \
  -m "Critical message" \
  --reliable
```

### Reliable + Partitioned

```bash
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/transactions \
  --partitions 4 \
  --reliable \
  --schema-subject transactions \
  -m '{"tx_id":"tx_789","amount":1000.00}'
```

## Practical Examples

### E-Commerce Orders

```bash
# Register order schema
danube-cli schema register orders \
  --schema-type json_schema \
  --file order-schema.json

# Send order
danube-cli produce \
  -s http://localhost:6650 \
  -t /default/orders \
  --schema-subject orders \
  --reliable \
  -m '{
    "order_id":"ord_123",
    "customer_id":"cust_456",
    "items":[{"sku":"ITEM1","qty":2,"price":29.99}],
    "total":59.98
  }'
```

### Event Streaming

```bash
# High-volume event stream
danube-cli produce \
  -s http://localhost:6650 \
  -t /analytics/events \
  --partitions 16 \
  --schema-subject user-events \
  -m '{"event":"page_view","user_id":"user_789","page":"/home"}' \
  --count 100000 \
  --interval 100
```

### IoT Sensor Data

```bash
# Send sensor readings
danube-cli produce \
  -s http://localhost:6650 \
  -t /iot/sensors \
  --attributes "sensor_id:temp_001,location:warehouse-a" \
  -m '{"temperature":22.5,"humidity":45,"timestamp":"2024-01-15T10:30:00Z"}' \
  --count 1440 \
  --interval 60000  # Every minute
```

### Log Aggregation

```bash
# Send application logs
danube-cli produce \
  -s http://localhost:6650 \
  -t /logs/application \
  --attributes "app:api-server,env:production,level:error" \
  -m '{"timestamp":"2024-01-15T10:30:00Z","message":"Database connection failed","stack_trace":"..."}'
```

## Command Reference

### All Producer Flags

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--service-addr` | `-s` | Broker URL | `http://127.0.0.1:6650` |
| `--topic` | `-t` | Topic name | `/default/test_topic` |
| `--message` | `-m` | Message content | Required* |
| `--file` | `-f` | Binary file path | - |
| `--producer-name` | `-n` | Producer name | `test_producer` |
| `--schema-subject` | - | Schema subject (latest version) | - |
| `--schema-version` | - | Pin to specific schema version | - |
| `--schema-min-version` | - | Use minimum version or newer | - |
| `--schema-file` | - | Schema file (auto-register) | - |
| `--schema-type` | - | Schema type | - |
| `--count` | `-c` | Number of messages | `1` |
| `--interval` | `-i` | Interval in ms | `500` |
| `--partitions` | `-p` | Number of partitions | - |
| `--attributes` | `-a` | Message attributes | - |
| `--reliable` | - | Reliable delivery | `false` |

*Required unless `--file` is provided
