# danube-admin: Brokers Commands

The `danube-admin-cli` tool provides commands to manage and view information about brokers in your Danube cluster. Below is the documentation for the commands related to brokers.

## Commands

### `danube-admin-cli brokers list`

List all active brokers in the cluster.

**Usage:**

```sh
danube-admin-cli brokers list [--output json]
```

**Description:**

This command retrieves and displays a list of all active brokers in the cluster. The output is formatted into a table with the following columns:

- **BROKER ID**: The unique identifier for the broker.
- **BROKER ADDRESS**: The network address of the broker.
- **BROKER ROLE**: The role assigned to the broker (e.g., "leader", "follower").

Use `--output json` to print JSON instead of a table.

**Example Output:**

```sh
+------------+---------------------+-------------+
| BROKER ID  | BROKER ADDRESS      | BROKER ROLE |
+------------+---------------------+-------------+
| 1          | 192.168.1.1:6650    | leader      |
| 2          | 192.168.1.2:6650    | follower    |
+------------+---------------------+-------------+
```

### `danube-admin-cli brokers leader-broker`

Get information about the leader broker in the cluster.

**Usage:**

```sh
danube-admin-cli brokers leader-broker
```

**Description:**

This command fetches and displays the details of the current leader broker in the cluster. The information includes the broker ID, address, and role of the leader.

**Example Output:**

```sh
Leader Broker: BrokerId: 1, Address: 192.168.1.1:6650, Role: leader
```

### `danube-admin-cli brokers namespaces`

List all namespaces in the cluster.

**Usage:**

```sh
danube-admin-cli brokers namespaces [--output json]
```

**Description:**

This command retrieves and lists all namespaces associated with the cluster. Each namespace is printed on a new line.

Use `--output json` to print JSON instead of plain text.

**Example Output:**

```sh
Namespace: default
Namespace: public
Namespace: my-namespace
```

## Error Handling

If there is an issue with connecting to the cluster or processing the request, the CLI will output an error message. Make sure your Danube cluster is running and accessible, and check your network connectivity.

## Examples

Here are a few example commands for quick reference:

- List all brokers:

  ```sh
  danube-admin-cli brokers list
  ```

- Get the leader broker:

  ```sh
  danube-admin-cli brokers leader-broker
  ```

- List all namespaces:

  ```sh
  danube-admin-cli brokers namespaces
  ```

For more detailed information or help with the `danube-admin-cli`, you can use the `--help` flag with any command.

**Example:**

```sh
danube-admin-cli brokers --help
```
