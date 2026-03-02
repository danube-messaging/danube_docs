# Run Danube with Docker Compose

Danube provides several Docker Compose setups for different use cases. All setups
use embedded Raft consensus for metadata — no external dependencies like etcd
are required.

## Available Setups

| Setup | Services | Best for |
|-------|----------|----------|
| **[Quickstart](#quickstart)** | 3 Brokers, CLI, Prometheus | Quick testing, learning Danube |
| **[With UI](#with-ui)** | Quickstart + Admin Server + Web UI | Visual exploration |
| **[With Cloud Storage](#with-cloud-storage)** | Quickstart + MinIO + MC | Cloud storage / durability testing |
| **[Local Development](#local-development)** | All above (builds from source) | Development, testing local changes |

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- At least 2GB RAM (4GB for cloud-storage setup)

## Quickstart

Minimal setup with filesystem backend — 3 brokers forming a Raft cluster, a
CLI container, and Prometheus for metrics.

### Step 1: Get the files

```bash
git clone https://github.com/danube-messaging/danube.git
cd danube/docker
```

### Step 2: Start the cluster

```bash
cd quickstart/
docker-compose up -d
```

### Step 3: Verify

```bash
docker-compose ps
```

Expected output:

```text
NAME               IMAGE                                          STATUS          PORTS
danube-broker1     ghcr.io/danube-messaging/danube-broker:latest   Up (healthy)    6650, 50051, 9040
danube-broker2     ghcr.io/danube-messaging/danube-broker:latest   Up (healthy)    6651, 50052, 9041
danube-broker3     ghcr.io/danube-messaging/danube-broker:latest   Up (healthy)    6652, 50053, 9042
danube-cli         ghcr.io/danube-messaging/danube-cli:latest      Up
danube-prometheus  prom/prometheus:latest                          Up              9090
```

Brokers form a Raft cluster automatically using `--seed-nodes`. The broker with
the lowest node ID initializes the cluster; the others join.

### Service Endpoints

| Service | Endpoint | Purpose |
|---------|----------|---------|
| Broker 1 | `localhost:6650` | gRPC messaging |
| Broker 2 | `localhost:6651` | gRPC messaging |
| Broker 3 | `localhost:6652` | gRPC messaging |
| Admin API 1 | `localhost:50051` | Broker administration |
| Admin API 2 | `localhost:50052` | Broker administration |
| Admin API 3 | `localhost:50053` | Broker administration |
| Prometheus | `localhost:9090` | Metrics UI |

## Testing with Danube CLI

The compose setup includes a `danube-cli` container with both `danube-cli` and
`danube-admin` pre-installed. No local installation required.

### Produce and consume messages

```bash
# Produce messages
docker exec danube-cli danube-cli produce \
  --service-addr http://broker1:6650 \
  --topic "/default/test" \
  --count 100 \
  --message "Hello Danube!"

# Consume messages
docker exec -it danube-cli danube-cli consume \
  --service-addr http://broker1:6650 \
  --topic "/default/test" \
  --subscription "my-sub"
```

### JSON schema messages

```bash
# Produce with schema
docker exec danube-cli danube-cli produce \
  --service-addr http://broker1:6650 \
  --topic "/default/json-topic" \
  --count 100 \
  --schema json \
  --json-schema '{"type":"object","properties":{"message":{"type":"string"},"timestamp":{"type":"number"}}}' \
  --message '{"message":"Hello JSON","timestamp":1640995200}'

# Consume
docker exec -it danube-cli danube-cli consume \
  --service-addr http://broker1:6650 \
  --topic "/default/json-topic" \
  --subscription "json-sub"
```

### Admin CLI operations

```bash
# List active brokers
docker exec danube-cli danube-admin brokers list

# List namespaces
docker exec danube-cli danube-admin brokers namespaces

# List topics
docker exec danube-cli danube-admin topics list default

# Cluster status (Raft membership)
docker exec danube-cli danube-admin cluster status
```

## With UI

Adds a web dashboard and admin server for visual cluster exploration.

```bash
cd with-ui/
docker-compose up -d
```

- **Web UI**: [http://localhost:8081](http://localhost:8081)
- **Admin API**: [http://localhost:8080](http://localhost:8080)

## With Cloud Storage

Adds MinIO (S3-compatible) for testing reliable / persistent messaging.

```bash
cd with-cloud-storage/
docker-compose up -d
```

- **MinIO Console**: [http://localhost:9001](http://localhost:9001) (login: `minioadmin` / `minioadmin123`)
- Buckets: `danube-messages`, `danube-wal`

**Test reliable delivery:**

```bash
docker exec danube-cli danube-cli produce \
  --service-addr http://broker1:6650 \
  --topic "/default/persistent" \
  --count 1000 \
  --message "Persistent message" \
  --reliable
```

## Local Development

Builds all services from source — for contributors working on broker or admin code.

```bash
cd local-development/
docker-compose up -d --build
```

Rebuild after code changes:

```bash
docker-compose build broker1 broker2 broker3
docker-compose up -d --no-deps broker1 broker2 broker3
```

## Monitoring

```bash
# Broker metrics
curl http://localhost:9040/metrics

# Prometheus UI
open http://localhost:9090
```

## Troubleshooting

- **Port conflicts**: Ensure ports 6650-6652, 50051-50053, 9040-9042, 9090 are free
- **Memory issues**: Increase Docker memory if containers fail to start
- **Logs**: `docker-compose logs -f broker1`

### Reset environment

```bash
docker-compose down -v
docker volume prune -f
docker-compose up -d
```

## Production Considerations

1. **Storage**: Replace MinIO with AWS S3, GCS, or Azure Blob Storage
2. **Security**: Enable TLS/mTLS in broker configuration (`auth: tls`)
3. **Resources**: Configure CPU/memory limits
4. **Monitoring**: Add Grafana dashboards for Prometheus
5. **Backup**: Back up Raft data directories for state recovery
6. **Orchestration**: Use [Kubernetes with Helm](Danube_kubernetes.md) for production scaling

### AWS S3 Migration

Update `danube_broker.yml`:

```yaml
wal_cloud:
  cloud:
    backend: "s3"
    root: "s3://your-production-bucket/danube-cluster"
    region: "us-west-2"
    # Remove endpoint for AWS S3
    # endpoint: "http://minio:9000"
    # Use IAM roles or environment variables for credentials
```
