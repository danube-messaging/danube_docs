# Danube-Pubsub CLI - Produce messages

The `produce` command sends messages to a specified topic.

## Usage

```bash
danube-pubsub produce [OPTIONS] --service-addr <SERVICE_ADDR> --message <MESSAGE>
```

### Options

- `-s, --service-addr <SERVICE_ADDR>`  
  Description: The service URL for the Danube broker.  
  Example: `http://127.0.0.1:6650`

- `-t, --topic <TOPIC>`  
  Description: The topic to produce messages to.  
  Default: `/default/test_topic`

- `-p, --schema <SCHEMA>`  
  Description: The schema type of the message.  
  Possible values: `bytes`, `string`, `int64`, `json`

- `-m, --message <MESSAGE>`  
  Description: The message to send.  
  Required: Yes

- `--json-schema <JSON_SCHEMA>`  
  Description: The JSON schema, required if schema type is `json`.

- `-c, --count <COUNT>`  
  Description: Number of times to send the message.  
  Default: `1`

- `-i, --interval <INTERVAL>`  
  Description: Interval between messages in milliseconds.  
  Default: `500`  
  Minimum: `100`

- `-a, --attributes <ATTRIBUTES>`  
  Description: Attributes in the form `'parameter:value'`.  
  Example: `'key1:value1,key2:value2'`

- `-h, --help`  
  Description: Print help information
  
### Example

To send 1000 messages with the content "Hello, Danube!" to the default topic:

```bash
danube-pubsub produce -s http://127.0.0.1:6650 -c 1000 -m "Hello, Danube!"
```
