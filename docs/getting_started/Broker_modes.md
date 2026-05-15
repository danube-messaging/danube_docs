# Broker Modes

Danube runs as a single binary (`danube-broker`) in one of three modes. The mode
determines how the broker manages metadata, handles client traffic, and
coordinates with other brokers.

## At a Glance

| | **Standalone** | **Cluster** | **Edge** |
|--|----------------|-------------|----------|
| **Use case** | Development, testing, single-server | Production, multi-broker | IoT ingestion at the edge |
| **Brokers** | 1 | 3+ (recommended) | 1 per site |
| **Config file** | None (zero-config) | `danube_broker.yml` | `edge.yaml` |
| **Raft consensus** | Single-node (auto-init) | Multi-node (auto-discovery) | Single-node (auto-init) |
| **Load balancing** | No | Yes (leader election, load manager) | No |
| **MQTT gateway** | No | No | Yes |
| **Clients** | Danube gRPC clients | Danube gRPC clients | MQTT devices + Danube gRPC |

---

## Standalone Mode

The fastest way to get started, a single broker with no config file and no
external dependencies.

```bash
danube-broker --mode standalone --data-dir ./danube-data
```

This auto-generates sensible defaults:

| Setting | Default |
|---------|---------|
| Broker address | `127.0.0.1:6650` |
| Admin address | `127.0.0.1:50051` |
| Raft address | `127.0.0.1:7650` |
| Storage mode | Local WAL |
| Authentication | None |

Override any default with CLI flags:

```bash
danube-broker --mode standalone \
  --data-dir ./danube-data \
  --broker-addr 0.0.0.0:6650 \
  --admin-addr 0.0.0.0:50051
```

!!! note "When to use standalone"
    Standalone mode is ideal for local development, CI pipelines, and
    single-server deployments where high availability is not required. It
    supports the same client APIs (producers, consumers, schemas) as cluster
    mode, just without multi-broker orchestration.

Data persists across restarts. To start fresh, remove the data directory.

---

## Cluster Mode

The default and recommended mode for production. Multiple brokers form a Raft
consensus group, share metadata, and distribute topics across the cluster.

### Requirements

- A config file (`danube_broker.yml`)
- At least 3 brokers for Raft quorum (recommended)
- Network connectivity between broker Raft ports

### Starting a Cluster

Each broker needs the config file and unique port assignments. The `--seed-nodes`
flag tells brokers how to discover each other:

```bash
# Broker 1
danube-broker --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6650 --admin-addr 0.0.0.0:50051 \
  --raft-addr 0.0.0.0:7650 --data-dir ./data1/raft \
  --seed-nodes "0.0.0.0:7650,0.0.0.0:7651,0.0.0.0:7652"

# Broker 2
danube-broker --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6651 --admin-addr 0.0.0.0:50052 \
  --raft-addr 0.0.0.0:7651 --data-dir ./data2/raft \
  --seed-nodes "0.0.0.0:7650,0.0.0.0:7651,0.0.0.0:7652"

# Broker 3
danube-broker --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6652 --admin-addr 0.0.0.0:50053 \
  --raft-addr 0.0.0.0:7652 --data-dir ./data3/raft \
  --seed-nodes "0.0.0.0:7650,0.0.0.0:7651,0.0.0.0:7652"
```

The broker with the lowest Raft node ID initializes the cluster; the others join
automatically. No manual cluster setup is needed.

### Cluster Features

Cluster mode enables capabilities that don't apply to standalone:

- **Leader election** : one broker is the Raft leader; metadata writes go through it
- **Load manager** : topics are assigned to brokers based on configurable strategies (`fair`, `balanced`, `weighted_load`)
- **Automated rebalancing** : optional proactive topic redistribution to maintain cluster balance
- **Broker scaling** : add brokers at runtime with `--join` or `danube-admin cluster add-node`

!!! tip "Deployment guides"
    For step-by-step cluster deployment instructions, see:

    - [Run Danube with Docker Compose](Danube_docker_compose.md)
    - [Run Danube on Kubernetes](Danube_kubernetes.md)

### Configuration Reference

The cluster config file (`danube_broker.yml`) controls broker ports, storage
backend, authentication, load management, and policies. See the
[Broker Configuration Reference](../reference/broker_configuration.md) and the
[sample config](https://github.com/danube-messaging/danube/blob/main/config/danube_broker.yml)
for all available options.

---

## Edge Mode

Edge mode turns the broker into a lightweight MQTT gateway that ingests data from
IoT devices and replicates it to a central Danube cluster. It is designed for
constrained environments (factory floors, remote sites) where devices
speak MQTT and need a local buffer that can survive network outages.

### How It Works

```
MQTT devices ──► Edge broker ──► Local WAL ──► Cluster broker(s)
                  (edge.yaml)     (on disk)     (gRPC replication)
```

1. Devices publish via standard MQTT (v3.1.1 / v5.0)
2. The edge broker routes messages to Danube topics via wildcard pattern matching
3. Messages are batched and written to a local WAL
4. A background replicator tail-reads the WAL and sends batches to the cluster
5. If the cluster is unreachable, messages accumulate safely in the local WAL

### Requirements

- A running Danube cluster (standalone or multi-broker) to replicate into
- An edge config file (`edge.yaml`)

### Starting an Edge Broker

```bash
danube-broker --mode edge \
  --data-dir ./edge-data \
  --edge-config edge.yaml \
  --broker-addr 0.0.0.0:6653 \
  --admin-addr 0.0.0.0:50054 \
  --raft-addr 0.0.0.0:7653
```

### Edge Configuration

The edge config file (`edge.yaml`) defines the edge identity, cluster
connection, and MQTT topic mappings:

```yaml
edge:
  edge_name: "edge1"
  cluster_url: "http://cluster-broker:6650"
  token: ""                    # optional auth token
  heartbeat_interval_ms: 30000

replicator:
  batch_size: 100
  batch_timeout_ms: 1000

mqtt:
  listener: "0.0.0.0:1883"
  # max_payload_size: 262144   # 256 KB (default)
  # max_connections: 10000     # default

  topic_mappings:
    - mqtt_pattern: "device/+/telemetry"
      danube_topic: "/edge1/telemetry"
      schema_subject: "telemetry-events"
      extract_attributes:
        device_id: "$1"

    - mqtt_pattern: "#"
      danube_topic: "/edge1/raw"

  ingestion:
    batch_size: 100
    batch_timeout_ms: 500
```

### Key Concepts

- **Namespace isolation** : all Danube topics must be under `/{edge_name}/` (e.g. `/edge1/telemetry`)
- **Topic mappings** : MQTT wildcards (`+`, `#`) map incoming messages to Danube topics; `extract_attributes` captures path segments as message attributes
- **Schema enforcement** : if `schema_subject` is set, the edge pulls the schema from the cluster registry and validates payloads locally. Invalid messages are acknowledged but silently dropped
- **WAL buffering** : messages survive edge broker restarts and network partitions; replication resumes from the last checkpoint

### Testing with an MQTT Client

Once the edge broker is running, any MQTT client can publish:

```bash
# Using mosquitto_pub
mosquitto_pub -h 127.0.0.1 -p 1883 \
  -t "device/sensor-1/telemetry" \
  -m '{"temperature": 25.5, "device_id": "sensor-1"}'
```

The message flows through the edge WAL to the cluster, where standard Danube
consumers can read it from `/edge1/telemetry`.

---

## CLI Reference

```text
danube-broker [OPTIONS]

Common options:
  --mode <mode>          cluster (default), standalone, or edge
  --broker-addr <addr>   gRPC listen address
  --admin-addr <addr>    Admin API listen address
  --raft-addr <addr>     Raft transport address
  --data-dir <path>      Base data directory
  --prom-exporter <addr> Prometheus metrics endpoint

Cluster only:
  --config-file <path>   Path to danube_broker.yml (required)
  --seed-nodes <addrs>   Comma-separated Raft peer addresses
  --join                 Join an existing cluster

Edge only:
  --edge-config <path>   Path to edge.yaml (required)
  --edge-token <token>   Auth token override (optional)
```
