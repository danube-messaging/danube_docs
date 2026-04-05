# Security Management

Manage authentication tokens, authorization roles, and access bindings.

## Overview

The `security` command provides tools for managing Danube's security layer. It covers three areas:

- **Tokens** — Create and validate JWT tokens for client and admin authentication (offline, no broker connection needed)
- **Roles** — Define named sets of permissions (e.g., `producer`, `consumer`, `admin`)
- **Bindings** — Attach roles to principals at cluster, namespace, or topic scope

Roles and bindings commands connect to a broker's admin gRPC endpoint. Set the target via `--admin-addr` or the `DANUBE_ADMIN_ENDPOINT` environment variable (default `http://127.0.0.1:50051`).

Token commands are **offline** — they perform local cryptographic operations and do not require a running broker.

> **Note:** Managing roles and bindings requires `ManageCluster` permission. See [Security Concepts](../../concepts/security.md) for the full RBAC model.

## Tokens

Create and validate JWT tokens locally. These are offline operations — no broker connection is needed.

### Create Token

Create a signed JWT token for client or admin authentication.

```bash
danube-admin security tokens create \
  --subject <NAME> \
  --secret-key <KEY>
```

**Arguments:**

| Option | Required | Default | Description |
|--------|----------|---------|-------------|
| `--subject` | Yes | — | Principal name (e.g., `my-app`, `admin-user`) |
| `--secret-key` | Yes | `$DANUBE_JWT_SECRET_KEY` | HMAC-SHA256 signing key (must match broker's `jwt.secret_key`) |
| `--type` | No | `service_account` | Principal type: `service_account` or `user` |
| `--ttl` | No | `8760h` (1 year) | Token time-to-live (e.g., `24h`, `365d`, `3600`) |
| `--issuer` | No | `danube-auth` | Token issuer (should match broker's `jwt.issuer`) |

**TTL Formats:**

| Format | Example | Meaning |
|--------|---------|---------|
| `Nh` | `24h` | N hours |
| `Nd` | `365d` | N days |
| `N` | `3600` | N seconds |

**Examples:**

```bash
# Service account token with default TTL (1 year)
danube-admin security tokens create \
  --subject my-app \
  --secret-key your-secret-key

# User token with 24-hour TTL
danube-admin security tokens create \
  --subject admin-user \
  --type user \
  --ttl 24h \
  --secret-key your-secret-key

# Long-lived token (1 year)
danube-admin security tokens create \
  --subject my-app \
  --ttl 365d \
  --secret-key your-secret-key

# Using environment variable for the secret key
export DANUBE_JWT_SECRET_KEY=your-secret-key
danube-admin security tokens create --subject my-app
```

**Output:**

The command prints only the raw JWT token to stdout, making it easy to capture in scripts:

```bash
export MY_TOKEN=$(danube-admin security tokens create \
  --subject my-app \
  --secret-key your-secret-key)
```

---

### Validate Token

Validate a JWT token and display its claims.

```bash
danube-admin security tokens validate \
  --token <JWT> \
  --secret-key <KEY>
```

**Arguments:**

| Option | Required | Default | Description |
|--------|----------|---------|-------------|
| `--token` | Yes | — | JWT token string to validate |
| `--secret-key` | Yes | `$DANUBE_JWT_SECRET_KEY` | HMAC-SHA256 signing key |

**Example:**

```bash
danube-admin security tokens validate \
  --token eyJhbGciOiJIUzI1NiIs... \
  --secret-key your-secret-key
```

**Example Output:**

```
Token is valid.
  Subject:        my-app
  Principal type:  service_account
  Principal name:  my-app
  Issuer:          danube-auth
  Expires:         2027-04-05 10:30:00 UTC
  Remaining:       364d 23h
```

If the token is invalid or expired, the command prints an error and exits with code 1.

---

## Roles

Manage authorization roles. Each role defines a named set of permissions.

### Valid Permissions

| Permission | Description |
|-----------|-------------|
| `Lookup` | Discover topic metadata and partitions |
| `Produce` | Publish messages to a topic |
| `Consume` | Subscribe and receive messages |
| `Replicate` | Replicate data between brokers |
| `ManageNamespace` | Create, delete, and configure namespaces |
| `ManageTopic` | Create, delete, and configure topics |
| `ManageSchema` | Register and manage schemas |
| `ManageBroker` | Broker operations (unload topics, etc.) |
| `ManageCluster` | Cluster-wide operations (Raft, security policies) |

### Create Role

Create a new role with the specified permissions.

```bash
danube-admin security roles create <NAME> --permissions <P1,P2,...>
```

**Arguments:**

| Argument/Option | Required | Description |
|----------------|----------|-------------|
| `<NAME>` | Yes | Role name (unique identifier) |
| `--permissions` | Yes | Comma-separated list of permissions |

**Examples:**

```bash
# Producer role
danube-admin security roles create producer-role \
  --permissions Produce,Lookup

# Consumer role
danube-admin security roles create consumer-role \
  --permissions Consume,Lookup

# Namespace admin role
danube-admin security roles create ns-admin-role \
  --permissions ManageNamespace,ManageTopic,ManageSchema,Lookup

# Full cluster admin role
danube-admin security roles create cluster-admin-role \
  --permissions ManageCluster,ManageBroker,ManageNamespace,ManageTopic,ManageSchema,Lookup,Produce,Consume
```

**Example Output:**

```
Role 'producer-role' created: true
```

---

### Get Role

Get details of a specific role.

```bash
danube-admin security roles get <NAME> [--output json]
```

**Arguments:**

| Argument/Option | Required | Description |
|----------------|----------|-------------|
| `<NAME>` | Yes | Role name |
| `--output` | No | Output format (`json`) |

**Example:**

```bash
danube-admin security roles get producer-role
```

**Example Output:**

```
Role: producer-role
  Permissions: Produce, Lookup
```

**JSON Output:**

```bash
danube-admin security roles get producer-role --output json
```

```json
{
  "name": "producer-role",
  "permissions": ["Produce", "Lookup"],
  "system": false
}
```

---

### List Roles

List all defined roles.

```bash
danube-admin security roles list [--output json]
```

**Example:**

```bash
danube-admin security roles list
```

**Example Output:**

```
Role: producer-role
  Permissions: Produce, Lookup

Role: consumer-role
  Permissions: Consume, Lookup

Role: cluster-admin-role
  Permissions: ManageCluster, ManageBroker, ManageNamespace, ManageTopic, ManageSchema, Lookup, Produce, Consume
```

If no roles exist:

```
No roles found.
```

---

### Delete Role

Delete a role by name. System roles cannot be deleted.

```bash
danube-admin security roles delete <NAME>
```

**Example:**

```bash
danube-admin security roles delete producer-role
```

**Example Output:**

```
Role 'producer-role' deleted: true
```

---

## Bindings

Manage authorization bindings. A binding attaches one or more roles to a principal at a specific scope.

### Scopes

| Scope | Resource | Description |
|-------|----------|-------------|
| `cluster` | (none) | Applies to all resources in the cluster |
| `namespace` | Namespace path (e.g., `/payments`) | Applies to all topics within the namespace |
| `topic` | Topic path (e.g., `/payments/orders`) | Applies to a specific topic only |

### Create Binding

Create a new binding that grants roles to a principal at a specified scope.

```bash
danube-admin security bindings create <ID> \
  --principal-type <TYPE> \
  --principal-name <NAME> \
  --roles <R1,R2,...> \
  --scope <SCOPE> \
  [--resource <PATH>]
```

**Arguments:**

| Argument/Option | Required | Default | Description |
|----------------|----------|---------|-------------|
| `<ID>` | Yes | — | Unique binding identifier |
| `--principal-type` | Yes | — | Principal type: `service_account` or `user` |
| `--principal-name` | Yes | — | Principal name (must match JWT subject) |
| `--roles` | Yes | — | Comma-separated list of role names to grant |
| `--scope` | Yes | — | Scope: `cluster`, `namespace`, or `topic` |
| `--resource` | No | `""` | Resource path (empty for cluster scope) |

**Examples:**

```bash
# Cluster-wide binding
danube-admin security bindings create b-platform-ops \
  --principal-type user \
  --principal-name ops-team \
  --roles cluster-admin-role \
  --scope cluster

# Namespace-scoped binding
danube-admin security bindings create b-payments-producer \
  --principal-type service_account \
  --principal-name payments-service \
  --roles producer-role \
  --scope namespace \
  --resource /payments

# Topic-scoped binding
danube-admin security bindings create b-orders-consumer \
  --principal-type service_account \
  --principal-name orders-processor \
  --roles consumer-role \
  --scope topic \
  --resource /payments/orders

# Multiple roles in one binding
danube-admin security bindings create b-devops \
  --principal-type user \
  --principal-name devops-user \
  --roles producer-role,consumer-role,ns-admin-role \
  --scope namespace \
  --resource /staging
```

**Example Output:**

```
Binding 'b-payments-producer' created: true
```

---

### Get Binding

Get details of a specific binding. You must specify the scope (and resource for namespace/topic scopes).

```bash
danube-admin security bindings get <ID> \
  --scope <SCOPE> \
  [--resource <PATH>] \
  [--output json]
```

**Arguments:**

| Argument/Option | Required | Default | Description |
|----------------|----------|---------|-------------|
| `<ID>` | Yes | — | Binding identifier |
| `--scope` | Yes | — | Scope where the binding was created |
| `--resource` | No | `""` | Resource path (for namespace/topic scopes) |
| `--output` | No | — | Output format (`json`) |

**Example:**

```bash
danube-admin security bindings get b-payments-producer \
  --scope namespace \
  --resource /payments
```

**Example Output:**

```
Binding: b-payments-producer
  Principal: service_account:payments-service
  Roles: producer-role
  Scope: namespace (/payments)
```

**JSON Output:**

```bash
danube-admin security bindings get b-payments-producer \
  --scope namespace --resource /payments --output json
```

```json
{
  "id": "b-payments-producer",
  "principal_type": "service_account",
  "principal_name": "payments-service",
  "role_names": ["producer-role"],
  "scope": "namespace",
  "resource_name": "/payments"
}
```

---

### List Bindings

List all bindings at a specific scope.

```bash
danube-admin security bindings list \
  --scope <SCOPE> \
  [--resource <PATH>] \
  [--output json]
```

**Arguments:**

| Argument/Option | Required | Default | Description |
|----------------|----------|---------|-------------|
| `--scope` | Yes | — | Scope to list: `cluster`, `namespace`, or `topic` |
| `--resource` | No | `""` | Resource path (for namespace/topic scopes) |
| `--output` | No | — | Output format (`json`) |

**Examples:**

```bash
# List all cluster-scoped bindings
danube-admin security bindings list --scope cluster

# List bindings for a specific namespace
danube-admin security bindings list --scope namespace --resource /payments

# List bindings for a specific topic
danube-admin security bindings list --scope topic --resource /payments/orders

# JSON output
danube-admin security bindings list --scope cluster --output json
```

**Example Output:**

```
Binding: b-platform-ops
  Principal: user:ops-team
  Roles: cluster-admin-role
  Scope: cluster

Binding: b-payments-producer
  Principal: service_account:payments-service
  Roles: producer-role
  Scope: namespace (/payments)
```

If no bindings exist at the specified scope:

```
No bindings found.
```

---

### Delete Binding

Delete a binding by ID. You must specify the scope (and resource for namespace/topic scopes).

```bash
danube-admin security bindings delete <ID> \
  --scope <SCOPE> \
  [--resource <PATH>]
```

**Arguments:**

| Argument/Option | Required | Default | Description |
|----------------|----------|---------|-------------|
| `<ID>` | Yes | — | Binding identifier |
| `--scope` | Yes | — | Scope where the binding was created |
| `--resource` | No | `""` | Resource path (for namespace/topic scopes) |

**Examples:**

```bash
# Delete a cluster-scoped binding
danube-admin security bindings delete b-platform-ops --scope cluster

# Delete a namespace-scoped binding
danube-admin security bindings delete b-payments-producer \
  --scope namespace --resource /payments

# Delete a topic-scoped binding
danube-admin security bindings delete b-orders-consumer \
  --scope topic --resource /payments/orders
```

**Example Output:**

```
Binding 'b-payments-producer' deleted: true
```

---

## Common Workflows

### 1. Bootstrap a Secured Cluster

```bash
# Create a super-admin token (subject must be in broker's super_admins list)
export ADMIN_TOKEN=$(danube-admin security tokens create \
  --subject admin --secret-key your-secret-key)

# Create standard roles
danube-admin security roles create producer --permissions Produce,Lookup
danube-admin security roles create consumer --permissions Consume,Lookup
danube-admin security roles create operator --permissions ManageNamespace,ManageTopic,ManageSchema,Lookup

# Create service account tokens
export APP_TOKEN=$(danube-admin security tokens create \
  --subject my-app --secret-key your-secret-key)

# Bind the service account to a role
danube-admin security bindings create bind-my-app \
  --principal-type service_account --principal-name my-app \
  --roles producer --scope namespace --resource /default
```

### 2. Grant Topic-Level Access

```bash
# Create a token for the service
danube-admin security tokens create \
  --subject order-processor --secret-key your-secret-key

# Grant consume access on a specific topic only
danube-admin security bindings create bind-order-processor \
  --principal-type service_account --principal-name order-processor \
  --roles consumer --scope topic --resource /payments/orders
```

### 3. Audit Current Policies

```bash
# List all roles
danube-admin security roles list

# List all cluster-wide bindings
danube-admin security bindings list --scope cluster

# List bindings for a namespace
danube-admin security bindings list --scope namespace --resource /payments

# Inspect a specific role
danube-admin security roles get operator --output json
```

### 4. Revoke Access

```bash
# Delete the binding (principal loses access immediately)
danube-admin security bindings delete bind-my-app \
  --scope namespace --resource /default

# Optionally delete the role if no longer needed
danube-admin security roles delete producer
```

## Quick Reference

```bash
# ── Tokens (offline) ──
danube-admin security tokens create --subject <name> --secret-key <key> [--type service_account] [--ttl 8760h] [--issuer danube-auth]
danube-admin security tokens validate --token <jwt> --secret-key <key>

# ── Roles ──
danube-admin security roles create <name> --permissions <P1,P2,...>
danube-admin security roles get <name> [--output json]
danube-admin security roles list [--output json]
danube-admin security roles delete <name>

# ── Bindings ──
danube-admin security bindings create <id> --principal-type <type> --principal-name <name> --roles <R1,R2,...> --scope <scope> [--resource <path>]
danube-admin security bindings get <id> --scope <scope> [--resource <path>] [--output json]
danube-admin security bindings list --scope <scope> [--resource <path>] [--output json]
danube-admin security bindings delete <id> --scope <scope> [--resource <path>]
```
