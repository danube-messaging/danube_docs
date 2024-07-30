# danube-admin: Topics Commands

The `danube-admin` tool provides commands to manage and view information about topics in your Danube cluster. Below is the documentation for the commands related to topics.

## Commands

### `danube-admin topics list NAMESPACE`

Get the list of topics in a specified namespace.

**Usage:**

```sh
danube-admin topics list NAMESPACE
```

**Description:**

This command retrieves and displays all topics within a specified namespace. Replace `NAMESPACE` with the name of the namespace you want to query.

**Example Output:**

```sh
Topic: topic1
Topic: topic2
Topic: topic3
```

### `danube-admin topics create TOPIC`

Create a non-partitioned topic.

**Usage:**

```sh
danube-admin topics create TOPIC
```

**Description:**

This command creates a new non-partitioned topic with the specified name. Replace `TOPIC` with the desired name for the new topic.

**Example Output:**

```sh
Topic Created: true
```

### `danube-admin topics create-partitioned-topic TOPIC --partitions #`

Create a partitioned topic.

**Usage:**

```sh
danube-admin topics create-partitioned-topic TOPIC --partitions #
```

**Description:**

This command creates a new partitioned topic with the specified name and number of partitions. Replace `TOPIC` with the desired name for the topic and `#` with the number of partitions.

**Example Output:**

```sh
Partitioned Topic Created: true
```

### `danube-admin topics delete TOPIC`

Delete a specified topic.

**Usage:**

```sh
danube-admin topics delete TOPIC
```

**Description:**

This command deletes the specified topic. Replace `TOPIC` with the name of the topic you want to delete.

**Example Output:**

```sh
Topic Deleted: true
```

### `danube-admin topics unsubscribe --subscription SUBSCRIPTION TOPIC`

Delete a subscription from a topic.

**Usage:**

```sh
danube-admin topics unsubscribe --subscription SUBSCRIPTION TOPIC
```

**Description:**

This command deletes a subscription from a specified topic. Replace `SUBSCRIPTION` with the name of the subscription and `TOPIC` with the name of the topic.

**Example Output:**

```sh
Unsubscribed: true
```

### `danube-admin topics subscriptions TOPIC`

Get the list of subscriptions on a specified topic.

**Usage:**

```sh
danube-admin topics subscriptions TOPIC
```

**Description:**

This command retrieves and displays all subscriptions associated with a specified topic. Replace `TOPIC` with the name of the topic you want to query.

**Example Output:**

```sh
Subscriptions: [subscription1, subscription2]
```

### `danube-admin topics create-subscription --subscription SUBSCRIPTION TOPIC`

Create a new subscription for a specified topic.

**Usage:**

```sh
danube-admin topics create-subscription --subscription SUBSCRIPTION TOPIC
```

**Description:**

This command creates a new subscription for a specified topic. Replace `SUBSCRIPTION` with the name of the subscription and `TOPIC` with the name of the topic.

**Example Output:**

```sh
Subscription Created: true
```

## Error Handling

If there is an issue with connecting to the cluster or processing the request, the CLI will output an error message. Ensure your Danube cluster is running and accessible, and check your network connectivity.

## Examples

Here are a few example commands for quick reference:

- List topics in a namespace:

  ```sh
  danube-admin topics list my-namespace
  ```

- Create a non-partitioned topic:

  ```sh
  danube-admin topics create my-topic
  ```

- Create a partitioned topic with 5 partitions:

  ```sh
  danube-admin topics create-partitioned-topic my-partitioned-topic --partitions 5
  ```

- Delete a topic:

  ```sh
  danube-admin topics delete my-topic
  ```

- Unsubscribe from a topic:

  ```sh
  danube-admin topics unsubscribe --subscription my-subscription my-topic
  ```

- List subscriptions for a topic:

  ```sh
  danube-admin topics subscriptions my-topic
  ```

- Create a new subscription for a topic:

  ```sh
  danube-admin topics create-subscription --subscription my-subscription my-topic
  ```

For more detailed information or help with the `danube-admin`, you can use the `--help` flag with any command.

**Example:**

```sh
danube-admin topics --help
```
