# Scaling a Danube Cluster

This guide explains how Danube brokers join and leave a cluster, and how to
scale the cluster up or down, on bare metal, Docker Compose, or Kubernetes.

## Node Lifecycle

Every broker embeds a Raft consensus node. On startup, the broker inspects its
local state and the cluster to decide how to proceed. This decision is captured
by the `BootstrapResult` enum:

| Scenario | How it's detected | What happens |
|---|---|---|
| **Single-node, first boot** | No `node_id` file, empty `seed_nodes` | Auto-initializes a 1-voter cluster |
| **Multi-node, first boot** | No `node_id` file, seed peers have no leader yet | Peers discover each other; the lowest `node_id` bootstraps the cluster, others become followers |
| **New node, existing cluster** | No `node_id` file, a seed peer reports `has_leader = true` | Enters join mode automatically — starts in "drained" state, must be added via admin CLI |
| **Restart** | `node_id` file exists on disk | Waits for Raft leader election then resumes; membership is already persisted |

The `has_leader` flag is served by the `GetNodeInfo` gRPC RPC during seed-node
discovery. This lets a fresh broker distinguish "cluster doesn't exist yet"
from "cluster is already running" without any external configuration or init
scripts.

### Auto-Detection vs. Manual Membership

Auto-detection determines **what the node should do**, not whether it joins the
cluster. When a fresh node sees `has_leader = true`, it knows not to bootstrap
a new cluster and instead enters a waiting state (equivalent to `--join`). This
prevents accidental split-brain scenarios where a new node could form a
second, independent cluster.

However, actually adding the node to the Raft voter set still requires explicit
admin commands (`add-node` → `promote-node` → `activate`). This is intentional:

- **Safety** — arbitrary nodes should not auto-join a Raft quorum. A
  misconfigured or rogue broker could disrupt consensus if membership were
  fully automatic.
- **Staged rollout** — the learner → voter → active progression lets operators
  verify each step. A learner doesn't affect quorum, so it can be removed
  safely if something is wrong.
- **Operational control** — the operator decides when a new broker starts
  receiving traffic, not the infrastructure.

This design applies equally to bare metal, Docker Compose, and Kubernetes. The
auto-detection removes the need for `--join` flags, shell wrappers, or
init-containers, but the membership decision remains an explicit admin action.

### Broker States

Once a broker joins a cluster, it transitions through these states:

```
drained ──► active ──► drained (unload) ──► removed
```

- **Drained** — the broker is a Raft member but the Load Manager will not
  assign topics to it. New nodes start here after joining.
- **Active** — eligible for topic assignments. Activated explicitly via
  `danube-admin brokers activate`.
- **Removed** — the node has been removed from Raft membership and can be
  safely stopped.

---

## Prerequisites

- **`danube-admin`** CLI installed ([releases](https://github.com/danube-messaging/danube/releases))
- **`DANUBE_ADMIN_ENDPOINT`** pointing to any active broker's admin port

```bash
export DANUBE_ADMIN_ENDPOINT=http://broker-1:50051
```

## Cluster Overview Commands

Before any scaling operation, understand the current state:

```bash
# Raft cluster membership (voters, learners, leader, term)
danube-admin cluster status

# Active brokers with status, addresses, and roles
danube-admin brokers list

# Load distribution and balance metrics
danube-admin brokers balance
```

`cluster status` shows the **Raft layer** (consensus membership), while
`brokers list` shows the **broker layer** (application-level registration). A
node can be a Raft voter but not yet an active broker — this distinction is
central to the scaling workflow.

---

## Scaling Up (Bare Metal / Docker Compose)

Adding a new node is a **4-step process**: start → add → promote → activate.

### Step 1: Start the New Broker in Join Mode

The new broker uses `--join` to skip cluster bootstrap. It does **not** need to
be listed in `seed_nodes` — it will be added dynamically.

```bash
danube-broker --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6653 --admin-addr 0.0.0.0:50054 \
  --raft-addr 0.0.0.0:7653 --data-dir ./danube-data/raft-4 \
  --prom-exporter 0.0.0.0:9043 \
  --join
```

At this point the broker:

- Starts its **admin gRPC server** (port 50054) — so it can be discovered
- **Waits** for a Raft leader to appear — it is not yet part of the cluster
- Does **not** start the client-facing broker gRPC server yet

### Step 2: Add as Learner (Non-Voting)

From any existing broker's admin endpoint, add the new node:

```bash
danube-admin cluster add-node --node-addr http://broker-4:50054
```

The node is now a **learner** — it receives Raft log replication but does not
vote. This is safe: it doesn't affect quorum, and if something goes wrong you
can remove it without impact.

Verify:

```bash
$ danube-admin cluster status
  Voters:   [8101916068459076819, 3725104926781042315, 6192847503618294107]
  Learners: [4918273605482917630]
```

### Step 3: Promote to Voter

Once the learner has caught up with the log (usually within seconds):

```bash
danube-admin cluster promote-node --node-id 4918273605482917630
```

The cluster is now a 4-voter Raft group (quorum = 3). The broker registers
itself in **drained** state — the Load Manager won't assign topics to it yet.

### Step 4: Activate the Broker

Make the broker eligible for topic assignments:

```bash
danube-admin brokers activate --broker-id 4918273605482917630 --reason "Scale-up"
```

### Step 5 (Optional): Rebalance Existing Topics

Activation only affects **new** topic assignments. Existing topics stay where
they are. To redistribute:

```bash
# Preview what would happen
danube-admin brokers rebalance --dry-run

# Execute rebalancing
danube-admin brokers rebalance
```

---

## Scaling Down (Bare Metal / Docker Compose)

Removing a node is the reverse: unload → remove → stop. The key principle is:
**always remove the node from Raft membership before stopping the process**,
otherwise you may lose quorum.

### Understanding Quorum

| Cluster Size | Quorum | Can Lose |
|---|---|---|
| 3 | 2 | 1 node |
| 4 | 3 | 1 node |
| 5 | 3 | 2 nodes |

If you kill a node without removing it from Raft, the cluster still needs its
vote for quorum. **Always remove before stopping.**

### Step 1: Unload All Topics

Migrate all hosted topics to other brokers:

```bash
# Preview
danube-admin brokers unload --broker-id 6192847503618294107 --dry-run

# Execute
danube-admin brokers unload --broker-id 6192847503618294107
```

### Step 2: Remove from Raft Membership

```bash
danube-admin cluster remove-node --node-id 6192847503618294107
```

### Step 3: Stop the Broker Process

Now it's safe to stop:

```bash
kill $(cat broker-3.pid)
# Or: systemctl stop danube-broker@3
```

### Bringing a Broker Back After Maintenance

After maintenance, **clean the old data directory** (the node was removed from
Raft, so its persisted state is stale), then rejoin using the same `--join`
workflow as scale-up:

```bash
rm -rf ./danube-data/raft-3

danube-broker --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6652 --admin-addr 0.0.0.0:50053 \
  --raft-addr 0.0.0.0:7652 --data-dir ./danube-data/raft-3 \
  --prom-exporter 0.0.0.0:9042 \
  --join

# Then: add-node → promote → activate → rebalance
```

---

## Scaling on Kubernetes (Helm)

When using the [danube-core Helm chart](https://github.com/danube-messaging/danube_helm),
brokers run as a StatefulSet with `podManagementPolicy: Parallel`. All pods
start simultaneously and discover each other via headless DNS seed nodes.

### Initial Cluster Formation

On first install, the Helm chart passes `--seed-nodes` with all pod FQDNs.
The brokers auto-bootstrap: the lowest `node_id` initializes the cluster and
the others join as followers. No manual intervention is needed.

### Scaling Up

1. Increase `broker.replicaCount` in your values:

    ```bash
    helm upgrade danube-core ./charts/danube-core -n danube \
      --set broker.replicaCount=5
    ```

2. The new pods start with `--seed-nodes` listing all 5 pod addresses. Because
   the existing pods already have a Raft leader, the new pods detect
   `has_leader = true` during discovery and enter **join mode** automatically.

3. Use the admin CLI to add, promote, and activate the new brokers:

    ```bash
    # Port-forward to any existing broker
    kubectl port-forward danube-core-broker-0 50051:50051 -n danube

    # For each new pod (e.g., broker-3, broker-4):
    danube-admin cluster add-node \
      --node-addr http://danube-core-broker-3.danube-core-broker-headless.danube.svc.cluster.local:50051
    danube-admin cluster promote-node --node-id <ID>
    danube-admin brokers activate --broker-id <ID>
    ```

4. Optionally rebalance:

    ```bash
    danube-admin brokers rebalance
    ```

### Scaling Down

1. Unload topics and remove the target broker from Raft (same commands as
   bare metal, via port-forward or from inside a pod).

2. Reduce `broker.replicaCount`:

    ```bash
    helm upgrade danube-core ./charts/danube-core -n danube \
      --set broker.replicaCount=3
    ```

3. Delete orphaned PVCs if needed:

    ```bash
    kubectl delete pvc data-danube-core-broker-3 -n danube
    kubectl delete pvc data-danube-core-broker-4 -n danube
    ```

### Pod Restarts and Rolling Upgrades

Pod restarts (crashes, rolling upgrades, node evictions) are handled
automatically. Because each pod's `node_id` is persisted in the PVC, the
broker detects `BootstrapResult::Restart` on startup — it waits for Raft
leader election and resumes with its existing membership. No admin
intervention is required.

---

## Quick Reference: Node Lifecycle

```
                    ┌───────────────────────────────────────┐
                    │         SCALING UP                     │
                    │                                       │
  danube-broker     │  cluster       cluster      brokers   │
    --join ─────────┼─► add-node ──► promote ──► activate ──┼──► Active
                    │   (learner)    (voter)      (active)  │
                    └───────────────────────────────────────┘

                    ┌───────────────────────────────────────┐
                    │         SCALING DOWN                   │
                    │                                       │
  Active ───────────┼─► brokers   ─► cluster     ─► kill   │
                    │   unload      remove-node    process  │
                    │   (drain)     (leave Raft)            │
                    └───────────────────────────────────────┘
```

## Command Reference

| Command | Purpose |
|---|---|
| `cluster status` | Show Raft membership (voters, learners, leader, term) |
| `cluster add-node --node-addr URL` | Add a new broker as a non-voting learner |
| `cluster promote-node --node-id ID` | Promote learner to voting member |
| `cluster remove-node --node-id ID` | Remove a node from Raft cluster |
| `brokers list` | List all registered brokers with status |
| `brokers activate --broker-id ID` | Set broker state to active (eligible for topics) |
| `brokers unload --broker-id ID` | Migrate all topics off a broker |
| `brokers unload --broker-id ID --dry-run` | Preview unload without executing |
| `brokers balance` | Show load distribution and CV metric |
| `brokers rebalance` | Trigger manual topic redistribution |
| `brokers rebalance --dry-run` | Preview rebalance moves |
