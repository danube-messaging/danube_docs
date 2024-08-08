# Danube-Pubsub CLI - Consume messages

The `consume` command retrieves messages from a specified topic.

## Usage

```bash
danube-pubsub consume [OPTIONS] --service-addr <SERVICE_ADDR> --subscription <SUBSCRIPTION>
```

### Options

- `-s, --service-addr <SERVICE_ADDR>`  
  **Description:** The service URL for the Danube broker.  
  **Example:** `http://127.0.0.1:6650`

- `-t, --topic <TOPIC>`  
  **Description:** The topic to consume messages from.  
  **Default:** `/default/test_topic`

- `-c, --consumer <CONSUMER>`  
  **Description:** The consumer name.  
  **Default:** `consumer_pubsub`

- `-m, --subscription <SUBSCRIPTION>`  
  **Description:** The subscription name.  
  **Required:** Yes

- `--sub-type <SUB_TYPE>`  
  **Description:** The subscription type.  
  **Default:** `shared`  
  **Possible values:** `exclusive`, `shared`, `fail-over`

- `-h, --help`  
  **Description:** Print help information.

### Example: Shared Subscription

To receive messages from a shared subscription:

```bash
danube-pubsub consume -s http://127.0.0.1:6650 -m my_shared_subscription
```

### Example: Exclusive Subscription

To receive messages from an exclusive subscription:

```bash
danube-pubsub consume -s http://127.0.0.1:6650 -m my_exclusive --sub-type exclusive
```

**To create a new exclusive subscription:**

```bash
danube-pubsub consume -s http://127.0.0.1:6650 -m my_exclusive2 --sub-type exclusive
```

## Resources

Check [this article](https://dev-state.com/posts/danube_pubsub/) that covers how to combine the subscription types in order to obtain message queueing or fan-out pub-sub messaging patterns.
