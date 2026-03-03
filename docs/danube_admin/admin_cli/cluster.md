# Cluster Management

Manage the Raft cluster membership and view consensus state.

## Overview

The `cluster` command provides direct control over the Raft consensus layer that underpins Danube's metadata replication. Use it to:

- View Raft cluster status (leader, term, voters, learners)
- Add new brokers to the cluster as learners
- Promote learners to full voting members
- Remove nodes from the cluster

All commands connect to a broker's admin gRPC endpoint. Set the target via `--admin-addr` or the `DANUBE_ADMIN_ENDPOINT` environment variable (default `http://127.0.0.1:50051`).

## Commands

### Cluster Status

Show current Raft cluster state including leader, term, voters, and learners.

```bash
danube-admin cluster status
```

**Example Output:**

```bash
Raft Cluster Status:
  Self Node ID:  9823746501928374
  Raft Address:  127.0.0.1:7650
  Leader:        9823746501928374
  Term:          3
  Last Applied:  142
  Voters:        [9823746501928374, 1048209374659182, 7391028374651029]
```

If learners are present they are also listed:

```bash
  Learners:      [5829103746519283]
```

**Fields Explained:**

| Field | Description |
|-------|-------------|
| **Self Node ID** | The node ID of the broker you connected to |
| **Raft Address** | The Raft transport address (`host:raft_port`) |
| **Leader** | Node ID of the current Raft leader (`none` if no leader elected) |
| **Term** | Current Raft term (monotonically increasing election counter) |
| **Last Applied** | Index of the last log entry applied to the state machine |
| **Voters** | Node IDs that participate in consensus (quorum members) |
| **Learners** | Node IDs replicating the log but not yet voting |

---

### Add Node

Add a new broker to the cluster as a Raft learner (non-voting). The CLI auto-discovers the new node's `node_id` and `raft_addr` by connecting to its admin endpoint first, then tells the leader to add it.

```bash
danube-admin cluster add-node --node-addr <ADMIN_ENDPOINT>
```

**Arguments:**

| Option | Required | Description |
|--------|----------|-------------|
| `--node-addr` | Yes | Admin gRPC endpoint of the new broker (e.g., `http://new-broker:50054`) |

**Example:**

```bash
danube-admin cluster add-node --node-addr http://new-broker:50054
```

**Example Output:**

```bash
Discovering node at http://new-broker:50054...
  Discovered: node_id=5829103746519283, raft_addr=new-broker:7653
Adding node 5829103746519283 as learner via http://127.0.0.1:50051...
Success: Node 5829103746519283 added as learner
```

**How It Works:**

1. Connects to the new node's admin endpoint (`--node-addr`)
2. Calls `cluster status` on the new node to discover its `node_id` and `raft_addr`
3. Connects to the current leader (via `DANUBE_ADMIN_ENDPOINT`)
4. Requests the leader to add the node as a Raft learner

**Important Notes:**

- The new broker must already be running with its Raft port accessible
- Learners replicate the Raft log but do **not** participate in votes
- After adding, use `promote-node` to grant voting rights
- See [Scaling the Cluster](../../concepts/scaling_cluster.md) for the full add → promote → activate workflow

---

### Promote Node

Promote a Raft learner to a full voting member (voter).

```bash
danube-admin cluster promote-node --node-id <NODE_ID>
```

**Arguments:**

| Option | Required | Description |
|--------|----------|-------------|
| `--node-id` | Yes | Node ID of the learner to promote |

**Example:**

```bash
danube-admin cluster promote-node --node-id 5829103746519283
```

**Example Output:**

```bash
Promoting node 5829103746519283 to voter...
Success: Node 5829103746519283 promoted to voter
```

**Important Notes:**

- The node must already be a learner (added via `add-node`)
- After promotion the node participates in Raft consensus votes and counts toward quorum
- For production clusters, always activate the broker after promoting so it can receive topic assignments:

```bash
danube-admin brokers activate <BROKER_ID>
```

---

### Remove Node

Remove a node from the Raft cluster entirely.

```bash
danube-admin cluster remove-node --node-id <NODE_ID>
```

**Arguments:**

| Option | Required | Description |
|--------|----------|-------------|
| `--node-id` | Yes | Node ID to remove from the cluster |

**Example:**

```bash
danube-admin cluster remove-node --node-id 5829103746519283
```

**Example Output:**

```bash
Removing node 5829103746519283 from cluster...
Success: Node 5829103746519283 removed
```

**Important Notes:**

- Unload topics from the broker **before** removing it from the cluster:

```bash
danube-admin brokers unload <BROKER_ID>
```

- Removing a voter reduces the quorum size — ensure you maintain a majority (e.g., don't drop from 3 voters to 1)
- The removed node's data directory can be cleaned up after removal

## Common Workflows

### 1. Scale Up (Add a Broker)

```bash
# Step 1: Check current cluster state
danube-admin cluster status

# Step 2: Add the new broker as a learner
danube-admin cluster add-node --node-addr http://new-broker:50054

# Step 3: Verify it appears as a learner
danube-admin cluster status

# Step 4: Promote to voter
danube-admin cluster promote-node --node-id <NODE_ID>

# Step 5: Activate the broker so it receives topics
danube-admin brokers activate <BROKER_ID>

# Step 6: Verify cluster balance
danube-admin brokers balance
```

### 2. Scale Down (Remove a Broker)

```bash
# Step 1: Unload topics from the broker
danube-admin brokers unload <BROKER_ID>

# Step 2: Remove from the Raft cluster
danube-admin cluster remove-node --node-id <NODE_ID>

# Step 3: Verify cluster state
danube-admin cluster status

# Step 4: Stop the broker process (external)
```

### 3. Health Check

```bash
# Quick cluster health overview
danube-admin cluster status
danube-admin brokers list
danube-admin brokers balance
```

## Quick Reference

```bash
# Raft cluster status
danube-admin cluster status

# Add a new broker (learner)
danube-admin cluster add-node --node-addr http://new-broker:50054

# Promote learner to voter
danube-admin cluster promote-node --node-id <node-id>

# Remove node from cluster
danube-admin cluster remove-node --node-id <node-id>
```
