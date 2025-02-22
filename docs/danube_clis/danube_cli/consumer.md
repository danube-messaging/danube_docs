# Danube-Pubsub CLI - Consume messages

The `consume` command subscribes to a topic and receives messages with support for different subscription types, schema validation, and message attributes tracking.

## Basic Usage

```bash
danube-cli consume [OPTIONS] --service-addr <SERVICE_ADDR> --subscription <SUBSCRIPTION>
```

### Required Arguments

- `-s, --service-addr <SERVICE_ADDR>`
  The service URL for the Danube broker (e.g., `http://127.0.0.1:6650`)

- `-m, --subscription <SUBSCRIPTION>`
  The subscription name to use for consuming messages

### Optional Arguments

- `-t, --topic <TOPIC>`
  Topic to consume messages from (default: /default/test_topic)

- `-n, --consumer <CONSUMER>`
  Consumer identifier (default: `consumer_pubsub`)

- `--sub-type <TYPE>`
  Subscription type: `exclusive`, `shared`, `fail-over` (default: `shared`)

- `-h, --help`  
  Print help information.

## Message Output Format

### Standard Messages

```bash
Received message: "message content"
Size: <size> bytes, Total received: <total> bytes
Attributes: key1=value1, key2=value2
```

### Reliable messages

```bash
Received reliable message: "message content"
Segment: <id>, Offset: <offset>, Size: <size> bytes, Total received: <total> bytes
Producer: <producer_id>, Topic: <topic_name>
Attributes: key1=value1, key2=value2
```

### Features

- **Schema Validation**: Automatically validates messages against topic schema
- **Large Message Handling**: Messages over 1KB are displayed as [binary data]
- **Message Tracking**: Tracks total bytes received and message segments
- **Attribute Display**: Shows message attributes if present
- **Multiple Schema Types**: Supports `bytes`, `string`, `int64`, and `json` schemas

## Examples

### Shared Subscription (Default)

```bash
danube-cli consume --service-addr http://localhost:6650 --subscription my_shared_subscription
```

### Exclusive Subscription

```bash
danube-cli consume -s http://localhost:6650 -m my_exclusive --sub-type exclusive
```

**To create a new exclusive subscription on the same topic:**

```bash
danube-cli consume -s http://127.0.0.1:6650 -m my_exclusive2 --sub-type exclusive
```

### Custom Consumer Name

```bash
danube-cli consume -s http://localhost:6650 -n my_consumer -m my_subscription
```

### Specific Topic

```bash
danube-cli consume -s http://localhost:6650 -t my_topic -m my_subscription
```

## Notes

- Messages are automatically acknowledged after processing
- JSON messages are pretty-printed and validated against schema if available
- The consumer maintains message ordering for reliable delivery
- Connection errors and message processing failures are reported to stderr
