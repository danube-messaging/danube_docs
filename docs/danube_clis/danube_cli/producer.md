# Danube-Pubsub CLI - Produce messages

The `produce` command sends messages to a specified topic with support for reliable delivery, custom schemas, and message attributes.

## Basic Usage

```bash
danube-cli produce [OPTIONS] --service-addr <SERVICE_ADDR> --message <MESSAGE>
```

### Required Arguments

- `-s, --service-addr <SERVICE_ADDR>`
  The service URL for the Danube broker (e.g., `http://127.0.0.1:6650`)

- `-m, --message <MESSAGE>`
  The message content to send

- `-f, --file <FILE_PATH>`
  Binary file path to send (takes precedence over --message when specified)

### Basic Options

- `-n, --producer-name <PRODUCER_NAME>`
  Producer identifier (default: `test_producer`)

- `-t, --topic <TOPIC>`
  Destination topic (default: `/default/test_topic`)

- `-p, --partitions <NUMBER>`
  Number of topic partitions

### Message Configuration

- `-y, --schema <SCHEMA>`
  Message schema type: `bytes`, `string`, `int64`, `json` (default: `string`)

- `--json-schema <JSON_SCHEMA>`
  Required JSON schema definition when using `json` schema type

- `-a, --attributes <ATTRIBUTES>`
  Message attributes in `key1:value1,key2:value2` format

- `-c, --count <COUNT>`
  Number of messages to send (default: `1`)

- `-i, --interval <INTERVAL>`
  Delay between messages in milliseconds (default: 500, minimum: 100)

### Reliable Delivery Options

- `--reliable`
  Enable reliable message delivery with storage persistence

- `--segment-size <SIZE>`
  Segment size in MB for reliable delivery (default: 20)

- `--retention <POLICY>`
  Retention policy: `ack` (retain until acknowledged) or `expire` (retain until time expires) (default: expire)

- `--retention-period <SECONDS>`
  Retention period in seconds for reliable delivery (default: `3600`)

- `-h, --help`  
  Description: Print help information
  
## Example

## Basic Message Production

```bash
danube-cli produce --service-addr http://localhost:6650 --count 100 --message "Hello Danube"
```

### JSON Messages with Schema

```bash
danube-cli produce -s http://localhost:6650 -c 100 \
  -y json \
  --json-schema '{"type": "object", "properties": {"field1": {"type": "string"}}}' \
  -m '{"field1":"Hello Danube"}'
```

### Reliable Message Delivery

```bash
danube-cli produce -s http://localhost:6650 -m "Hello Danube" -c 100 \
  --reliable \
  --segment-size 10 \
  --retention expire \
  --retention-period 7200
```

### Binary File send with Reliable Delivery

```bash
danube-cli produce -s http://localhost:6650 -m "none" -f ./data.blob -c 100 \
  --reliable \
  --segment-size 5 \
  --retention expire \
  --retention-period 7200
```
