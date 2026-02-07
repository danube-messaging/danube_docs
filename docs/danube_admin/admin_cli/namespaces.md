# Namespaces Management

Organize your topics with namespaces in Danube.

## Overview

Namespaces provide logical isolation for topics in your Danube cluster. Use namespaces to:

- Organize topics by application, team, or environment
- Apply policies at the namespace level
- Control access and resource allocation
- Separate production, staging, and development workloads

## Commands

### List Topics in a Namespace

View all topics within a specific namespace.

```bash
danube-admin-cli namespaces topics <NAMESPACE>
```

**Basic Usage:**

```bash
danube-admin-cli namespaces topics default
```

**Output Formats:**

```bash
# Plain text (default)
danube-admin-cli namespaces topics default

# JSON format - for automation
danube-admin-cli namespaces topics default --output json
```

**Example Output (Plain Text):**

```
Topics in namespace 'default':
  /default/user-events
  /default/payment-logs
  /default/analytics
```

**Example Output (JSON):**

```json
[
  "/default/user-events",
  "/default/payment-logs",
  "/default/analytics"
]
```

---

### View Namespace Policies

Get the policies configured for a namespace.

```bash
danube-admin-cli namespaces policies <NAMESPACE>
```

**Basic Usage:**

```bash
danube-admin-cli namespaces policies default
```

**Output Formats:**

```bash
# Plain text (default) - pretty printed
danube-admin-cli namespaces policies default

# JSON format
danube-admin-cli namespaces policies default --output json
```

**Example Output (Plain Text):**

```bash
Policies for namespace 'default':
{
  "max_topics_per_namespace": 1000,
  "max_producers_per_topic": 100,
  "max_consumers_per_topic": 100,
  "message_ttl_seconds": 604800,
  "retention_policy": "time_based"
}
```

**Example Output (JSON):**

```json
{
  "max_topics_per_namespace": 1000,
  "max_producers_per_topic": 100,
  "max_consumers_per_topic": 100,
  "message_ttl_seconds": 604800,
  "retention_policy": "time_based"
}
```

**Common Policies:**

| Policy | Description | Typical Values |
|--------|-------------|----------------|
| `max_topics_per_namespace` | Maximum number of topics | `100` - `10000` |
| `max_producers_per_topic` | Maximum producers per topic | `10` - `1000` |
| `max_consumers_per_topic` | Maximum consumers per topic | `10` - `1000` |
| `message_ttl_seconds` | Message time-to-live | `3600` (1h) - `604800` (7d) |
| `retention_policy` | How messages are retained | `time_based`, `size_based` |

---

### Create a Namespace

Create a new namespace in the cluster.

```bash
danube-admin-cli namespaces create <NAMESPACE>
```

**Basic Usage:**

```bash
# Create namespace
danube-admin-cli namespaces create production
```

**Example Output:**

```bash
✅ Namespace created: production
```

**Naming Guidelines:**

- Use lowercase letters and hyphens
- Keep names descriptive: `production`, `staging`, `dev`
- Avoid special characters
- Use consistent naming: `team-app-env` pattern

**Examples:**

```bash
# By environment
danube-admin-cli namespaces create production
danube-admin-cli namespaces create staging
danube-admin-cli namespaces create development

# By team
danube-admin-cli namespaces create analytics-team
danube-admin-cli namespaces create platform-team

# By application
danube-admin-cli namespaces create payment-service
danube-admin-cli namespaces create user-service
```

---

### Delete a Namespace

Remove a namespace from the cluster.

```bash
danube-admin-cli namespaces delete <NAMESPACE>
```

**Basic Usage:**

```bash
danube-admin-cli namespaces delete old-namespace
```

**Example Output:**

```
✅ Namespace deleted: old-namespace
```

**⚠️ Important Warnings:**

1. **All Topics Deleted**: Deleting a namespace removes ALL topics within it
2. **No Confirmation**: This operation is immediate and irreversible
3. **Active Connections**: Connected producers/consumers will be disconnected
4. **Data Loss**: All messages in the namespace are permanently deleted

**Safety Checklist:**

```bash
# 1. List topics before deletion
danube-admin-cli namespaces topics my-namespace

# 2. Verify no critical topics
danube-admin-cli namespaces topics my-namespace --output json | grep -i critical

# 3. Check policies to understand impact
danube-admin-cli namespaces policies my-namespace

# 4. Only then delete
danube-admin-cli namespaces delete my-namespace
```

## Common Workflows

### 1. Namespace Setup for New Application

```bash
# Create namespace
danube-admin-cli namespaces create payment-service

# Verify creation
danube-admin-cli brokers namespaces | grep payment-service

# Check default policies
danube-admin-cli namespaces policies payment-service

# Create topics in namespace
danube-admin-cli topics create /payment-service/transactions
danube-admin-cli topics create /payment-service/refunds
danube-admin-cli topics create /payment-service/notifications
```

### 2. Multi-Environment Setup

```bash
# Create environments
danube-admin-cli namespaces create production
danube-admin-cli namespaces create staging
danube-admin-cli namespaces create development

# List all namespaces
danube-admin-cli brokers namespaces

# Create same topics in each environment
for env in production staging development; do
  danube-admin-cli topics create /$env/user-events
  danube-admin-cli topics create /$env/order-events
done
```
