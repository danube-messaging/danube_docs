# danube-admin: Namespaces Commands

The `danube-admin` tool provides commands to manage and view information about namespaces in your Danube cluster. Below is the documentation for the commands related to namespaces.

## Commands

### `danube-admin namespaces topics NAMESPACE`

Get the list of topics for a specified namespace.

**Usage:**

```sh
danube-admin namespaces topics NAMESPACE
```

**Description:**

This command retrieves and displays all topics associated with a specific namespace. Replace `NAMESPACE` with the name of the namespace you want to query.

**Example Output:**

```sh
Topic: topic1
Topic: topic2
Topic: topic3
```

### `danube-admin namespaces policies NAMESPACE`

Get the configuration policies for a specified namespace.

**Usage:**

```sh
danube-admin namespaces policies NAMESPACE
```

**Description:**

This command fetches and displays the configuration policies for a specific namespace. Replace `NAMESPACE` with the name of the namespace you want to query.

**Example Output:**

```sh
Policy Name: policy1
Policy Description: Description of policy1
Policy Name: policy2
Policy Description: Description of policy2
```

### `danube-admin namespaces create NAMESPACE`

Create a new namespace.

**Usage:**

```sh
danube-admin namespaces create NAMESPACE
```

**Description:**

This command creates a new namespace with the specified name. Replace `NAMESPACE` with the desired name for the new namespace.

**Example Output:**

```sh
Namespace Created: true
```

### `danube-admin namespaces delete NAMESPACE`

Delete a specified namespace. The namespace must be empty.

**Usage:**

```sh
danube-admin namespaces delete NAMESPACE
```

**Description:**

This command deletes a namespace. The specified namespace must be empty before it can be deleted. Replace `NAMESPACE` with the name of the namespace you wish to delete.

**Example Output:**

```sh
Namespace Deleted: true
```

## Error Handling

If there is an issue with connecting to the cluster or processing the request, the CLI will output an error message. Make sure your Danube cluster is running and accessible, and check your network connectivity.

## Examples

Here are a few example commands for quick reference:

- List all topics in a namespace:

  ```sh
  danube-admin namespaces topics my-namespace
  ```

- Get the policies for a namespace:

  ```sh
  danube-admin namespaces policies my-namespace
  ```

- Create a new namespace:

  ```sh
  danube-admin namespaces create my-new-namespace
  ```

- Delete a namespace:

  ```sh
  danube-admin namespaces delete my-old-namespace
  ```

For more detailed information or help with the `danube-admin`, you can use the `--help` flag with any command.

**Example:**

```sh
danube-admin namespaces --help
```
