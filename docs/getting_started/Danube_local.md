# Run Danube Broker on your local machine

Danube brokers use an embedded Raft consensus layer for metadata — no external
dependencies (like etcd) are required. A single broker auto-initializes as a
single-node Raft cluster on first boot.

For an overview of all deployment modes (standalone, cluster, edge), see
[Broker Modes](Broker_modes.md).

## Download the Danube Broker

Download the latest binary from the [releases](https://github.com/danube-messaging/danube/releases) page.

## Option 1: Single-Node Quick Start

The fastest way to get a broker running locally — no config file needed:

```bash
./danube-broker --single-node --data-dir ./danube-data
```

This auto-generates sensible defaults:

| Setting         | Value                  |
|-----------------|------------------------|
| Broker address  | `127.0.0.1:6650`      |
| Admin address   | `127.0.0.1:50051`     |
| Raft address    | `127.0.0.1:7650`      |
| Raft data       | `./danube-data/raft`   |
| WAL storage     | `./danube-data/wal`    |
| Authentication  | None                   |
| Storage mode    | Local                  |

Data is persisted across restarts. To start fresh, remove the data directory and re-run.

Skip to [Use Danube CLI to Publish and Consume Messages](#use-danube-cli-to-publish-and-consume-messages) to test it.

## Option 2: Config File

Create a local config file using the [sample config](https://github.com/danube-messaging/danube/blob/main/config/danube_broker.yml) as a reference:

```bash
curl -O https://raw.githubusercontent.com/danube-messaging/danube/main/config/danube_broker.yml
```

### Run the Broker with Config File

```bash
./danube-broker-linux \
  --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6650 \
  --admin-addr 0.0.0.0:50051 \
  --raft-addr 0.0.0.0:7650 \
  --data-dir ./danube-data/raft > broker.log 2>&1 &
```

Check the logs:

```bash
tail -n 100 -f broker.log
```

You should see the Raft node initialize and the broker start:

```text
INFO danube_broker: Raft metadata store initialized node_id=... raft_addr=0.0.0.0:7650
INFO danube_broker: Start the Danube Service
INFO danube_broker::danube_service: Setting up the cluster MY_CLUSTER
INFO danube_broker::danube_service::broker_register: Broker ... registered in the cluster
INFO danube_broker::broker_server: Server is listening on address: 0.0.0.0:6650
INFO danube_broker::admin: Admin is listening on address: 0.0.0.0:50051
```

## Use Danube CLI to Publish and Consume Messages

Download the latest Danube CLI binary from the [releases](https://github.com/danube-messaging/danube/releases) page and run it:

```bash
./danube-cli-linux produce -s http://127.0.0.1:6650 -t /default/demo_topic -c 1000 -m "Hello, Danube!"
```

```bash
Message sent successfully with ID: 9
Message sent successfully with ID: 10
Message sent successfully with ID: 11
Message sent successfully with ID: 12
```

Open a new terminal and run the below command to consume the messages:

```bash
./danube-cli-linux consume -s http://127.0.0.1:6650 -t /default/demo_topic -m my_subscription
```

```bash
Received bytes message: 9, with payload: Hello, Danube!
Received bytes message: 10, with payload: Hello, Danube!
Received bytes message: 11, with payload: Hello, Danube!
Received bytes message: 12, with payload: Hello, Danube!
```

## Validate

Ensure the broker is running and listening on the expected port:

```bash
ss -tuln | grep 6650
```

Check cluster state with the admin CLI:

```bash
danube-admin brokers list
```

For debugging, check `broker.log`.

## Running a Multi-Broker Cluster

To run multiple brokers locally, give each broker a unique port set and pass
`--seed-nodes` so they discover each other:

```bash
# Broker 1
./danube-broker-linux --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6650 --admin-addr 0.0.0.0:50051 \
  --raft-addr 0.0.0.0:7650 --data-dir ./data1/raft \
  --seed-nodes "0.0.0.0:7650,0.0.0.0:7651" &

# Broker 2
./danube-broker-linux --config-file danube_broker.yml \
  --broker-addr 0.0.0.0:6651 --admin-addr 0.0.0.0:50052 \
  --raft-addr 0.0.0.0:7651 --data-dir ./data2/raft \
  --seed-nodes "0.0.0.0:7650,0.0.0.0:7651" &
```

The broker with the lowest Raft node ID initializes the cluster; the other
joins automatically.

## Cleanup

Stop the broker:

```bash
pkill danube-broker
```

Remove Raft data (for a fresh start):

```bash
rm -rf ./danube-data
```

Verify cleanup:

```bash
ps aux | grep danube-broker
```
