# Brokers Management

Manage and view broker information in your Danube cluster.

## Overview

The `brokers` command provides visibility and control over the brokers in your Danube cluster. Use it to:

- List all brokers with their status
- Identify the leader broker
- View broker namespaces
- Check cluster balance metrics
- Trigger cluster rebalancing
- Unload topics from brokers
- Activate brokers

## Commands

### List All Brokers

Display all brokers in the cluster with their details.

```bash
danube-admin brokers list
```

**Output Formats:**

```bash
# Plain text (default) - easy to read
danube-admin brokers list

# JSON format - for scripting/automation
danube-admin brokers list --output json
```

**Example Output (Plain Text):**

```bash
┌──────────────┬─────────────────────┬──────────┬─────────────────────┬─────────────────────┬────────┐
│ Broker ID    │ Address             │ Role     │ Admin Address       │ Metrics Address     │ Status │
├──────────────┼─────────────────────┼──────────┼─────────────────────┼─────────────────────┼────────┤
│ broker-001   │ 127.0.0.1:6650      │ leader   │ 127.0.0.1:50051     │ 127.0.0.1:9090      │ active │
│ broker-002   │ 127.0.0.1:6651      │ follower │ 127.0.0.1:50052     │ 127.0.0.1:9091      │ active │
└──────────────┴─────────────────────┴──────────┴─────────────────────┴─────────────────────┴────────┘
```

**Example Output (JSON):**

```json
[
  {
    "broker_id": "broker-001",
    "broker_addr": "127.0.0.1:6650",
    "broker_role": "leader",
    "admin_addr": "127.0.0.1:50051",
    "metrics_addr": "127.0.0.1:9090",
    "broker_status": "active"
  },
  {
    "broker_id": "broker-002",
    "broker_addr": "127.0.0.1:6651",
    "broker_role": "follower",
    "admin_addr": "127.0.0.1:50052",
    "metrics_addr": "127.0.0.1:9091",
    "broker_status": "active"
  }
]
```

---

### Get Leader Broker

Identify which broker is currently the cluster leader.

```bash
danube-admin brokers leader
```

**Example Output:**

```bash
Leader: broker-001
```

**Why This Matters:**

- The leader broker coordinates cluster operations
- Useful for debugging cluster issues
- Important for understanding cluster topology

---

### List Broker Namespaces

View all namespaces managed by the cluster.

```bash
danube-admin brokers namespaces
```

**Output Formats:**

```bash
# Plain text
danube-admin brokers namespaces

# JSON format
danube-admin brokers namespaces --output json
```

**Example Output (Plain Text):**

```bash
Namespaces: ["default", "analytics", "logs"]
```

**Example Output (JSON):**

```json
["default", "analytics", "logs"]
```

---

### Unload Broker Topics

Gracefully unload topics from a broker (useful for maintenance or rebalancing).

```bash
danube-admin brokers unload <BROKER_ID> [OPTIONS]
```

**Basic Usage:**

```bash
# Unload all topics from broker-001
danube-admin brokers unload broker-001

# Dry-run to see what would be unloaded
danube-admin brokers unload broker-001 --dry-run
```

**Advanced Options:**

```bash
# Unload with custom parallelism
danube-admin brokers unload broker-001 --max-parallel 5

# Unload only specific namespaces
danube-admin brokers unload broker-001 \
  --namespace-include default \
  --namespace-include analytics

# Exclude certain namespaces
danube-admin brokers unload broker-001 \
  --namespace-exclude system

# Set custom timeout per topic (seconds)
danube-admin brokers unload broker-001 --timeout 30
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--dry-run` | Preview topics to be unloaded without making changes | `false` |
| `--max-parallel` | Number of topics to unload concurrently | `1` |
| `--namespace-include` | Only unload topics from these namespaces (repeatable) | All |
| `--namespace-exclude` | Skip topics from these namespaces (repeatable) | None |
| `--timeout` | Timeout in seconds for each topic unload | `30` |

**Example Output:**

```bash
Unload Started: true
Total Topics: 45
Succeeded: 45
Failed: 0
Pending: 0
```

**Use Cases:**

- **Broker Maintenance**: Drain topics before shutting down a broker
- **Load Rebalancing**: Move topics to other brokers
- **Rolling Upgrades**: Safely upgrade brokers one at a time

---

### Get Cluster Balance

Show cluster balance metrics and broker load distribution.

```bash
danube-admin brokers balance [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin brokers balance
```

**Output Formats:**

```bash
# Plain text (default) - formatted report
danube-admin brokers balance

# JSON format - for automation
danube-admin brokers balance --output json
```

**Example Output (Plain Text):**

```bash
╔══════════════════════════════════════════════════════════╗
║           Cluster Balance Report                       ║
╚══════════════════════════════════════════════════════════╝

Status:                    ✅ Well Balanced
Coefficient of Variation:  15.23%
Assignment Strategy:       Balanced

Interpretation Guide:
  < 20%  = ✅ Well Balanced
  20-30% = ✅ Balanced
  30-40% = ⚠️  Imbalanced
  > 40%  = ❌ Severely Imbalanced

Load Statistics:
  Mean Load:       4.50
  Max Load:        5.00
  Min Load:        4.00
  Std Deviation:   0.50
  Broker Count:    2

Broker Details:
Broker ID       Load       Topics     Status
--------------------------------------------------
broker-001      5.00       5          OK
broker-002      4.00       4          OK
```

**Example Output (JSON):**

```json
{
  "coefficient_of_variation": 0.1523,
  "mean_load": 4.5,
  "max_load": 5.0,
  "min_load": 4.0,
  "std_deviation": 0.5,
  "broker_count": 2,
  "brokers": [
    {
      "broker_id": "broker-001",
      "load": 5.0,
      "topic_count": 5,
      "is_overloaded": false,
      "is_underloaded": false
    }
  ]
}
```

**Metrics Explained:**

- **Coefficient of Variation (CV)**: Measures load distribution uniformity. Lower is better.
  - < 20% = Well Balanced (optimal)
  - 20-30% = Balanced (acceptable)
  - 30-40% = Imbalanced (consider rebalancing)
  - \> 40% = Severely Imbalanced (rebalancing recommended)

- **Load**: Topic count weighted by topic activity and resource usage

**Use Cases:**

- **Regular Monitoring**: Check cluster balance health
- **Before Rebalancing**: Determine if rebalancing is needed
- **After Broker Changes**: Verify balance after adding/removing brokers
- **Capacity Planning**: Understand load distribution patterns

---

### Trigger Cluster Rebalancing

Manually trigger cluster rebalancing to distribute topics evenly across brokers.

```bash
danube-admin brokers rebalance [OPTIONS]
```

**Basic Usage:**

```bash
# Preview moves without executing
danube-admin brokers rebalance --dry-run

# Execute rebalancing
danube-admin brokers rebalance

# Limit number of moves
danube-admin brokers rebalance --max-moves 10
```

**Options:**

| Option | Description | Default |
|--------|-------------|---------|
| `--dry-run` | Preview proposed moves without executing them | `false` |
| `--max-moves` | Maximum number of topic moves to execute | All necessary moves |

**Example Output (Dry-Run):**

```bash
🔍 Dry-Run Mode: Preview only, no changes will be applied

Proposed Moves: 3

Topic                      From            To
--------------------------------------------------
/default/events            broker-001      broker-002
/default/logs              broker-001      broker-002
/default/metrics           broker-001      broker-003

Load Before Rebalancing:
  broker-001: 8 topics (Overloaded)
  broker-002: 2 topics (Underloaded)
  broker-003: 2 topics (Underloaded)

Expected Load After Rebalancing:
  broker-001: 5 topics (OK)
  broker-002: 4 topics (OK)
  broker-003: 3 topics (OK)

💡 Run without --dry-run to execute these moves
```

**Example Output (Execution):**

```bash
✅ Rebalancing Complete!

Moves Executed: 3
Background Mode: No
Load Reason: LoadImbalance

Topic Movements:
  ✓ /default/events: broker-001 → broker-002
  ✓ /default/logs: broker-001 → broker-002
  ✓ /default/metrics: broker-001 → broker-003

💡 Run 'danube-admin brokers balance' to verify cluster state
```

**Recommended Workflow:**

```bash
# Step 1: Check current balance
danube-admin brokers balance

# Step 2: Preview moves (dry-run)
danube-admin brokers rebalance --dry-run

# Step 3: Execute rebalancing
danube-admin brokers rebalance

# Step 4: Verify results
danube-admin brokers balance
```

**When to Use:**

- **CV > 30%**: Cluster is imbalanced, rebalancing recommended
- **After Broker Changes**: Added or removed brokers from cluster
- **Before Maintenance**: Balance load before planned downtime
- **Resource Optimization**: Improve cluster efficiency

**Important Notes:**

- Topics are moved gracefully with no message loss
- Brief disruption (few seconds) during topic reassignment
- Use `--dry-run` first to preview impact
- Use `--max-moves` for gradual rebalancing

---

### Activate Broker

Mark a broker as active, allowing it to receive traffic.

```bash
danube-admin brokers activate <BROKER_ID> [OPTIONS]
```

**Basic Usage:**

```bash
danube-admin brokers activate broker-002
```

**With Audit Reason:**

```bash
danube-admin brokers activate broker-002 \
  --reason "Maintenance completed"
```

**Example Output:**

```bash
Activated: true
```

**Use Cases:**

- **After Maintenance**: Re-enable a broker after maintenance
- **After Unload**: Activate broker to start receiving topics again
- **Cluster Expansion**: Activate newly added brokers

## Common Workflows

### 1. Health Check

```bash
# Check cluster health
danube-admin brokers list
danube-admin brokers leader

# Verify all brokers are active
danube-admin brokers list | grep -c active
```

### 2. Broker Maintenance

```bash
# Step 1: Dry-run to preview unload
danube-admin brokers unload broker-001 --dry-run

# Step 2: Unload topics
danube-admin brokers unload broker-001

# Step 3: Perform maintenance (external)
# ...

# Step 4: Reactivate broker
danube-admin brokers activate broker-001 --reason "Maintenance completed"
```

### 3. Cluster Expansion

```bash
# List current brokers
danube-admin brokers list

# Add new broker (external process)
# ...

# Activate new broker
danube-admin brokers activate broker-003 --reason "New broker added"

# Verify
danube-admin brokers list
```

## Quick Reference

```bash
# List all brokers
danube-admin brokers list

# Get leader
danube-admin brokers leader

# List namespaces
danube-admin brokers namespaces

# Check cluster balance
danube-admin brokers balance

# Rebalance cluster (dry-run)
danube-admin brokers rebalance --dry-run

# Rebalance cluster (execute)
danube-admin brokers rebalance --max-moves 10

# Unload topics (dry-run)
danube-admin brokers unload <broker-id> --dry-run

# Unload topics (execute)
danube-admin brokers unload <broker-id> --max-parallel 5

# Activate broker
danube-admin brokers activate <broker-id> --reason "Ready"
```
