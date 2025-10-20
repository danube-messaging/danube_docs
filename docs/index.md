# Welcome to Danube Messaging

[Danube](https://github.com/danube-messaging/danube) is a lightweight, cloud‑native messaging platform built in Rust. It delivers sub‑second dispatch with cloud economics by combining a Write‑Ahead Log (WAL) with object storage, so you get low‑latency pub/sub and durable streaming—on one broker.

Danube lets multiple **producers** publish to **topics**, and multiple **consumers** receive messages via named **subscriptions**. Choose Non‑Reliable (best‑effort pub/sub) or Reliable (at‑least‑once streaming) per topic to match your workload.

For design details, see the [Architecture](architecture/architecture.md).

## Try Danube in minutes

**Docker Compose Quickstart**: Use the provided HA setup with MinIO and ETCD.

  - Guide: https://github.com/danube-messaging/danube/tree/main/docker
  - Or see our docs page: [Docker Compose setup](../../docker/README.md)

## Core capabilities

**[Topics](architecture/topics.md)**

  - Non‑partitioned: served by a single broker.
  - Partitioned: split across brokers for scale and HA.

**[Dispatch strategies](architecture/dispatch_strategy.md)**

  - Non‑Reliable: in‑memory, best‑effort delivery, lowest latency.
  - Reliable: WAL + Cloud persistence with acknowledgments and replay.

**[Subscriptions](architecture/subscriptions.md)**

  - Exclusive, Shared, Failover patterns for queueing and fan‑out.

**[Message schemas](architecture/messages.md)**
  
  - Bytes, String, Int64, JSON.

**Concept guides**

  - [Messaging Modes (Pub/Sub vs Streaming)](architecture/messaging_modes_pubsub_vs_streaming.md)
  - [Messaging Patterns (Queuing vs Pub/Sub)](architecture/messaging_patterns_queuing_vs_pubsub.md)

## Crates in the workspace

Repository: https://github.com/danube-messaging/danube

- [danube-broker](https://github.com/danube-messaging/danube/tree/main/danube-broker) – The broker service (topics, producers, consumers, subscriptions).
- [danube-core](https://github.com/danube-messaging/danube/tree/main/danube-core) – Core types, protocol, and shared logic.
- [danube-metadata-store](https://github.com/danube-messaging/danube/tree/main/danube-metadata-store) – Metadata storage and cluster coordination.
- [danube-persistent-storage](https://github.com/danube-messaging/danube/tree/main/danube-persistent-storage) – WAL and cloud persistence backends.

CLIs and client libraries:

- [danube-client](https://github.com/danube-messaging/danube/tree/main/danube-client) – Async Rust client library.
- [danube-cli](https://github.com/danube-messaging/danube/tree/main/danube-cli) – Publish/consume client CLI.
- [danube-admin-cli](https://github.com/danube-messaging/danube/tree/main/danube-admin-cli) – Admin CLI for cluster management.

## Client libraries

- [danube-client (Rust)](https://crates.io/crates/danube-client)
- [danube-go (Go)](https://pkg.go.dev/github.com/danube-messaging/danube-go)

Contributions for other languages (Python, Java, etc.) are welcome.
