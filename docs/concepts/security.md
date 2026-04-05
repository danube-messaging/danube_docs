# Security

Danube provides a layered security model that protects both data in transit and access to cluster resources. Security spans three areas: **encryption** (TLS), **authentication** (verifying identity), and **authorization** (enforcing what an identity can do). Each layer can be enabled independently, but for production use all three should be active.

## Security Modes

Danube supports two security modes, controlled by the `auth.mode` setting in the broker configuration:

| Mode | Encryption | Authentication | Authorization | Use Case |
|------|-----------|---------------|---------------|----------|
| `none` | Off | Off | Off | Local development and testing |
| `tls` | TLS everywhere | JWT tokens | RBAC with default-deny | Production deployments |

When `mode: none`, all connections are plain HTTP/gRPC with no authentication. Any client can produce, consume, and manage the cluster without credentials. **This mode should never be used in production.**

When `mode: tls`, the broker enables full security:

- All client and inter-broker connections are encrypted with TLS.
- Clients must present a valid JWT token to authenticate.
- Every operation is checked against RBAC policies — if no policy grants access, the request is denied.

---

## TLS Encryption

TLS encrypts all network traffic to prevent eavesdropping and tampering. A single set of certificate files is used across all broker services:

| Connection | TLS Mode | Port |
|-----------|----------|------|
| Client gRPC (producers/consumers) | Server TLS | 6650 |
| Admin gRPC | Server TLS | 50051 |
| Inter-broker replication | Mutual TLS (mTLS) | 6650 |
| Raft consensus | Mutual TLS (mTLS) | 7650 |

### Certificate Configuration

```yaml
auth:
  mode: tls
  tls:
    cert_file: "./cert/server-cert.pem"   # Server certificate
    key_file: "./cert/server-key.pem"     # Server private key
    ca_file: "./cert/ca-cert.pem"         # CA certificate (for verifying peers)
```

- **Server TLS** (client-facing): The broker presents its certificate to clients. Clients verify it against the CA.
- **Mutual TLS** (inter-broker): Both sides present certificates and verify each other. This is how brokers authenticate to each other for replication and Raft consensus.

### Generating Certificates

Danube ships with a convenience script that generates a CA, server certificate (with proper SANs for `localhost` / `127.0.0.1`), and a client certificate for mTLS:

```bash
cd cert/
bash gen_certs.sh
```

This produces:

| File | Purpose |
|------|---------|
| `ca-cert.pem` / `ca-key.pem` | Certificate Authority (signs both server and client certs) |
| `server-cert.pem` / `server-key.pem` | Broker TLS identity — used in `auth.tls` config |
| `client-cert.pem` / `client-key.pem` | Client mTLS identity — used by inter-broker and mTLS clients |

The generated server certificate includes Subject Alternative Names for `localhost`, `127.0.0.1`, and `0.0.0.0`, making it ready for local development and testing.

For production, use certificates from your organization's PKI or a certificate manager like cert-manager (Kubernetes).

---

## Authentication

Authentication verifies *who* is making a request. When security is enabled (`mode: tls`), every gRPC call must carry valid credentials. The broker supports two authentication methods:

### JWT Tokens (Primary)

JSON Web Tokens are the primary authentication mechanism for clients and administrators. Tokens are signed with a shared secret and carry the caller's identity.

**Broker configuration:**

```yaml
auth:
  jwt:
    secret_key: "your-secret-key"    # HMAC-SHA256 signing key
    issuer: "danube-auth"            # Expected token issuer
    expiration_time: 3600            # Default TTL in seconds
```

**Token claims:**

Every Danube JWT contains:

| Claim | Description |
|-------|------------|
| `sub` | Subject — the principal name (e.g., `my-app`, `admin`) |
| `iss` | Issuer — must match the broker's `jwt.issuer` |
| `exp` | Expiration timestamp (Unix epoch) |
| `principal_type` | Either `service_account` or `user` |
| `principal_name` | Principal name (usually same as `sub`) |

**Creating tokens with `danube-admin`:**

```bash
# Create a service account token (default type, 1-year TTL)
danube-admin security tokens create \
  --subject my-app \
  --secret-key your-secret-key

# Create a user token with custom TTL
danube-admin security tokens create \
  --subject admin-user \
  --type user \
  --ttl 24h \
  --secret-key your-secret-key

# Create a token with a specific issuer
danube-admin security tokens create \
  --subject my-app \
  --ttl 365d \
  --issuer danube-auth \
  --secret-key your-secret-key
```

The command prints only the raw token to stdout, making it easy to pipe into scripts or environment variables.

**Validating a token:**

```bash
danube-admin security tokens validate \
  --token eyJhbGciOiJIUzI1NiIs... \
  --secret-key your-secret-key
```

Output includes the subject, principal type, issuer, expiration, and remaining TTL.

> **Note:** Token creation and validation are offline operations — they do not require a connection to the broker. The `secret_key` must match the broker's configured `jwt.secret_key`.

### Inter-Broker Authentication (mTLS)

Broker-to-broker communication (replication, Raft) authenticates automatically using mutual TLS. When a broker connects to another broker, the TLS handshake verifies both certificates against the shared CA.

Internal broker traffic is mapped to a `BrokerInternal` principal, which has super-admin privileges — it can perform any operation. This ensures that replication and consensus continue to work seamlessly when security is enabled.

---

## Principals

A **principal** represents an authenticated identity in the system. Every authenticated request is associated with exactly one principal. The broker recognizes four principal types:

| Principal Type | Description | How It's Created |
|---------------|-------------|-----------------|
| `service_account` | Application or service identity | JWT token with `principal_type: service_account` |
| `user` | Human operator or administrator | JWT token with `principal_type: user` |
| `broker_internal` | Inter-broker communication | Automatic via mTLS between brokers |
| `anonymous` | Unauthenticated caller | Only when `auth.mode: none` |

In production (`mode: tls`):

- **Service accounts** are the default for application clients (producers and consumers).
- **Users** are typically human administrators using `danube-admin`.
- **BrokerInternal** is assigned automatically to inter-broker connections — you don't create these manually.
- **Anonymous** is never used (requests without credentials are rejected).

---

## Authorization (RBAC)

Authorization determines *what* an authenticated principal can do. Danube uses Role-Based Access Control (RBAC) with scoped bindings and **default-deny** semantics — if no policy explicitly grants access, the request is denied.

### Key Concepts

The authorization model has three building blocks:

```
Principal  ←──  Binding  ──→  Role  ──→  Permissions
                  ↓
               Scope (cluster / namespace / topic)
```

- **Permissions** — individual actions like `Produce`, `Consume`, or `ManageTopic`
- **Roles** — named bundles of permissions (e.g., a `producer` role grants `Lookup` + `Produce`)
- **Bindings** — attach a role to a principal at a specific scope

### Permissions

Danube defines 9 permissions:

| Permission | Description | Typical Use |
|-----------|-------------|-------------|
| `Lookup` | Discover topic metadata and partitions | Required for producers and consumers |
| `Produce` | Publish messages to a topic | Producers |
| `Consume` | Subscribe and receive messages from a topic | Consumers |
| `Replicate` | Replicate data between brokers | Reserved for future cross-cluster replication |
| `ManageNamespace` | Create, delete, and configure namespaces | Namespace administrators |
| `ManageTopic` | Create, delete, and configure topics | Topic administrators |
| `ManageSchema` | Register and manage schemas | Schema administrators |
| `ManageBroker` | Broker operations (unload topics, etc.) | Cluster operators |
| `ManageCluster` | Cluster-wide operations (Raft membership, security policies) | Super administrators |

### Resources

Permissions are checked against specific resources:

| Resource | Example | Scope Level |
|----------|---------|------------|
| Cluster | (global) | `cluster` |
| Broker | A specific broker node | `cluster` |
| Namespace | `/payments` | `namespace` |
| Topic | `/payments/orders` | `topic` |
| Schema Subject | `/payments/orders` | `topic` |

### Scoped Bindings

Bindings can be attached at three scope levels:

- **Cluster** — applies to all resources in the cluster
- **Namespace** — applies to all topics within a namespace
- **Topic** — applies to a specific topic only

When evaluating a request, the authorizer checks bindings from the **most specific scope to the broadest**:

1. Topic-level bindings (if the resource is a topic)
2. Namespace-level bindings (for the topic's namespace)
3. Cluster-level bindings

If any matching binding grants the required permission, the request is **allowed**. If no binding matches, the request is **denied** (default-deny).

### Roles

Roles are named collections of permissions. You create roles to match your organizational needs.

**Creating roles:**

```bash
# A role for producers: can look up and publish to topics
danube-admin security roles create producer-role \
  --permissions Produce,Lookup

# A role for consumers: can look up, subscribe, and receive messages
danube-admin security roles create consumer-role \
  --permissions Consume,Lookup

# A namespace admin role
danube-admin security roles create ns-admin-role \
  --permissions ManageNamespace,ManageTopic,ManageSchema,Lookup

# A full cluster admin role
danube-admin security roles create cluster-admin-role \
  --permissions ManageCluster,ManageBroker,ManageNamespace,ManageTopic,ManageSchema,Lookup,Produce,Consume
```

**Listing and inspecting roles:**

```bash
# List all roles
danube-admin security roles list

# Get details of a specific role
danube-admin security roles get producer-role

# JSON output (for scripting)
danube-admin security roles list --output json
```

**Deleting roles:**

```bash
danube-admin security roles delete producer-role
```

### Bindings

Bindings connect principals to roles at a specific scope. Each binding has a unique ID.

**Creating bindings:**

```bash
# Grant producer-role to service account 'payments-writer' on the entire cluster
danube-admin security bindings create b-payments-writer \
  --principal-type service_account \
  --principal-name payments-writer \
  --roles producer-role \
  --scope cluster

# Grant consumer-role on a specific namespace
danube-admin security bindings create b-analytics-reader \
  --principal-type service_account \
  --principal-name analytics-reader \
  --roles consumer-role \
  --scope namespace \
  --resource /analytics

# Grant producer-role on a specific topic only
danube-admin security bindings create b-orders-writer \
  --principal-type service_account \
  --principal-name orders-service \
  --roles producer-role \
  --scope topic \
  --resource /payments/orders

# Grant multiple roles in one binding
danube-admin security bindings create b-platform-admin \
  --principal-type user \
  --principal-name admin-user \
  --roles cluster-admin-role,ns-admin-role \
  --scope cluster
```

**Listing and inspecting bindings:**

```bash
# List all cluster-scoped bindings
danube-admin security bindings list --scope cluster

# List bindings for a specific namespace
danube-admin security bindings list --scope namespace --resource /payments

# List bindings for a specific topic
danube-admin security bindings list --scope topic --resource /payments/orders

# Get a specific binding
danube-admin security bindings get b-payments-writer --scope cluster

# JSON output
danube-admin security bindings list --scope cluster --output json
```

**Deleting bindings:**

```bash
danube-admin security bindings delete b-payments-writer --scope cluster
danube-admin security bindings delete b-analytics-reader --scope namespace --resource /analytics
```

---

## Super-Admins

For initial cluster bootstrap and break-glass emergency access, Danube supports **super-admins** — JWT subjects that bypass RBAC entirely. Super-admins are declared in the broker configuration, not through the RBAC system.

```yaml
auth:
  super_admins:
    - "admin"
```

A super-admin can perform any operation on any resource without needing roles or bindings. This is essential for the initial setup before any roles exist.

> **Warning:** Super-admin access is powerful and bypasses all authorization checks. Use it only for initial bootstrap and emergency scenarios. Once your RBAC policies are configured, consider removing entries from `super_admins`.

---

## Securing a Cluster: Step-by-Step

### 1. Generate TLS Certificates

Generate your CA, server certificate, and key. Place them in a `cert/` directory accessible to all broker nodes:

```
cert/
  ca-cert.pem
  server-cert.pem
  server-key.pem
```

For multi-node clusters, all brokers should share the same CA certificate. Each broker can use the same or different server certificates, as long as they're signed by the shared CA.

### 2. Configure the Broker

Use the secure configuration template (`config/danube_broker_secure.yml`):

```yaml
auth:
  mode: tls

  tls:
    cert_file: "./cert/server-cert.pem"
    key_file: "./cert/server-key.pem"
    ca_file: "./cert/ca-cert.pem"

  jwt:
    secret_key: "your-secret-key"
    issuer: "danube-auth"
    expiration_time: 3600

  super_admins:
    - "admin"
```

### 3. Create a Bootstrap Admin Token

Before you can create roles and bindings, you need a super-admin token to authenticate with the admin API:

```bash
# Create a super-admin token (subject must be listed in super_admins)
export ADMIN_TOKEN=$(danube-admin security tokens create \
  --subject admin \
  --secret-key your-secret-key)
```

### 4. Start the Broker

```bash
danube-broker --config-file config/danube_broker_secure.yml
```

### 5. Create Roles

Define roles that match your access patterns:

```bash
# Producer role: can discover topics and publish messages
danube-admin security roles create producer \
  --permissions Produce,Lookup

# Consumer role: can discover topics and receive messages
danube-admin security roles create consumer \
  --permissions Consume,Lookup

# Operator role: can manage namespaces and topics
danube-admin security roles create operator \
  --permissions ManageNamespace,ManageTopic,ManageSchema,Lookup
```

### 6. Create Service Account Tokens

Create a JWT token for each application or service:

```bash
# Token for the payments service (produces to /payments/* topics)
export PAYMENTS_TOKEN=$(danube-admin security tokens create \
  --subject payments-service \
  --secret-key your-secret-key)

# Token for the analytics service (consumes from /analytics/* topics)
export ANALYTICS_TOKEN=$(danube-admin security tokens create \
  --subject analytics-service \
  --secret-key your-secret-key)
```

### 7. Create Bindings

Grant roles to service accounts at the appropriate scope:

```bash
# payments-service can produce to the /payments namespace
danube-admin security bindings create bind-payments-producer \
  --principal-type service_account \
  --principal-name payments-service \
  --roles producer \
  --scope namespace \
  --resource /payments

# analytics-service can consume from the /analytics namespace
danube-admin security bindings create bind-analytics-consumer \
  --principal-type service_account \
  --principal-name analytics-service \
  --roles consumer \
  --scope namespace \
  --resource /analytics
```

### 8. Configure Clients

Application clients authenticate by sending their JWT token as a `Bearer` token in the `authorization` gRPC metadata header. The Danube client libraries handle this automatically when configured with a token.

### 9. Verify Access

Test that your service accounts can only access what they're authorized for:

```bash
# This should succeed (payments-service has producer role on /payments)
# → produces to /payments/orders

# This should fail with PermissionDenied
# → payments-service tries to consume from /analytics/events
```

### 10. Remove Super-Admin (Optional)

Once your RBAC policies are in place, you can remove the `super_admins` entry from the broker configuration for tighter security. Keep the admin token available for emergency break-glass scenarios.

---

## Enforcement Summary

Authorization is enforced on every gRPC call across all broker services:

### Data Plane

| Operation | Required Permission | Resource |
|----------|-------------------|----------|
| Topic lookup | `Lookup` | Topic |
| Topic partitions | `Lookup` | Topic |
| Create producer | `Produce` | Topic |
| Send message | `Produce` | Topic |
| Subscribe | `Consume` | Topic |
| Receive messages | `Consume` | Cluster |
| Acknowledge | `Consume` | Cluster |

### Admin Plane

| Operation | Required Permission | Resource |
|----------|-------------------|----------|
| Create/delete namespace | `ManageNamespace` | Namespace or Cluster |
| Create/delete topic | `ManageTopic` | Topic or Namespace |
| List/inspect brokers | `ManageBroker` | Cluster |
| Raft membership changes | `ManageCluster` | Cluster |
| Schema operations | `ManageSchema` | Schema Subject or Topic |
| Role/binding management | `ManageCluster` | Cluster |

---

## Admin TLS

By default, the admin API (port 50051) runs without TLS even when `auth.mode: tls` is set. This is convenient for local management but insecure for remote access.

To enable TLS on the admin API:

```yaml
admin_tls: true
```

When enabled, the admin API uses the same certificate from `auth.tls`. This is recommended for any deployment where the admin API is accessed over untrusted networks.

---

## Quick Reference

### Configuration (danube_broker.yml)

```yaml
auth:
  mode: tls                              # none | tls
  tls:
    cert_file: "./cert/server-cert.pem"
    key_file: "./cert/server-key.pem"
    ca_file: "./cert/ca-cert.pem"
  jwt:
    secret_key: "your-secret-key"
    issuer: "danube-auth"
    expiration_time: 3600
  super_admins:
    - "admin"

admin_tls: false                         # Enable TLS on admin API
```

### CLI Commands

```bash
# ── Tokens (offline, no broker connection needed) ──
danube-admin security tokens create --subject <name> --secret-key <key> [--type user|service_account] [--ttl 8760h]
danube-admin security tokens validate --token <jwt> --secret-key <key>

# ── Roles ──
danube-admin security roles create <name> --permissions <P1,P2,...>
danube-admin security roles get <name> [--output json]
danube-admin security roles list [--output json]
danube-admin security roles delete <name>

# ── Bindings ──
danube-admin security bindings create <id> --principal-type <type> --principal-name <name> --roles <R1,R2,...> --scope <cluster|namespace|topic> [--resource <path>]
danube-admin security bindings get <id> --scope <scope> [--resource <path>] [--output json]
danube-admin security bindings list --scope <scope> [--resource <path>] [--output json]
danube-admin security bindings delete <id> --scope <scope> [--resource <path>]
```

### Valid Permissions

`Lookup`, `Produce`, `Consume`, `Replicate`, `ManageNamespace`, `ManageTopic`, `ManageSchema`, `ManageBroker`, `ManageCluster`

### Valid Principal Types

`service_account`, `user`

### Valid Scopes

`cluster`, `namespace`, `topic`
