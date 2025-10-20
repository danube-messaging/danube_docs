# danube-admin: Topics Commands

The `danube-admin-cli` tool provides commands to manage and view information about topics in your Danube cluster. Below is the documentation for the commands related to topics.

## Commands

### `danube-admin-cli topics list NAMESPACE`

Get the list of topics in a specified namespace.

**Usage:**

```sh
danube-admin-cli topics list NAMESPACE [--output json]
```

**Description:**

This command retrieves and displays all topics within a specified namespace. Replace `NAMESPACE` with the name of the namespace you want to query.

Use `--output json` to print JSON instead of plain text.

**Example Output:**

```sh
Topic: topic1
Topic: topic2
Topic: topic3
```

### `danube-admin-cli topics create TOPIC`

Create a topic (non‑partitioned or partitioned). You can also set schema and dispatch strategy.

**Usage:**

```sh
danube-admin-cli topics create TOPIC [--namespace NS] [--partitions N] \
  [-s, --schema String|Bytes|Int64|Json] [--schema-file PATH | --schema-data JSON] \
  [--dispatch-strategy non_reliable|reliable]
```

**Description:**

This command creates a new topic. `TOPIC` accepts either `/namespace/topic` or `topic` (when `--namespace` is provided). Use `--partitions` to create a partitioned topic. For `Json` schema, provide the schema via `--schema-file` or `--schema-data`.

**Example Output:**

```sh
Topic Created: true
```

### `danube-admin-cli topics delete TOPIC`

Delete a specified topic.

**Usage:**

```sh
danube-admin-cli topics delete TOPIC [--namespace NS]
```

**Description:**

This command deletes the specified topic. `TOPIC` accepts `/namespace/topic` or `topic` with `--namespace`.

**Example Output:**

```sh
Topic Deleted: true
```

### `danube-admin-cli topics subscriptions TOPIC`

Get the list of subscriptions on a specified topic.

**Usage:**

```sh
danube-admin-cli topics subscriptions TOPIC [--namespace NS] [--output json]
```

**Description:**

This command retrieves and displays all subscriptions associated with a specified topic. `TOPIC` accepts `/namespace/topic` or `topic` with `--namespace`. Use `--output json` for JSON output.

**Example Output:**

```sh
Subscriptions: [subscription1, subscription2]
```

### `danube-admin-cli topics describe TOPIC`

Describe a topic: schema and subscriptions.

**Usage:**

```sh
danube-admin-cli topics describe TOPIC [--namespace NS] [--output json]
```

**Description:**

Shows topic name, schema (pretty-printed when JSON), and subscriptions. `TOPIC` accepts `/namespace/topic` or `topic` with `--namespace`.

### `danube-admin-cli topics unsubscribe --subscription SUBSCRIPTION TOPIC`

Delete a subscription from a topic.

**Usage:**

```sh
danube-admin-cli topics unsubscribe --subscription SUBSCRIPTION TOPIC [--namespace NS]
```

**Description:**

This command deletes a subscription from a specified topic. `TOPIC` accepts `/namespace/topic` or `topic` with `--namespace`.

**Example Output:**

```sh
Unsubscribed: true
```

## Error Handling

If there is an issue with connecting to the cluster or processing the request, the CLI will output an error message. Ensure your Danube cluster is running and accessible, and check your network connectivity.

## Examples

Here are a few example commands for quick reference:

- List topics in a namespace:

  ```sh
  danube-admin-cli topics list my-namespace --output json
  ```

- Create a topic:

  ```sh
  danube-admin-cli topics create /default/my-topic --dispatch-strategy reliable
  ```

- Delete a topic:

  ```sh
  danube-admin-cli topics delete my-topic --namespace default
  ```

- Unsubscribe from a topic:

  ```sh
  danube-admin-cli topics unsubscribe --subscription my-subscription my-topic --namespace default
  ```

- List subscriptions for a topic:

  ```sh
  danube-admin-cli topics subscriptions my-topic --namespace default --output json
  ```

For more detailed information or help with the `danube-admin-cli`, you can use the `--help` flag with any command.

**Example:**

```sh
danube-admin-cli topics --help
```
