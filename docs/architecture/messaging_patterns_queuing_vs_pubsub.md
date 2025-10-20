# Messaging Patterns: Queuing vs Pub/Sub (Fan‑out)

This page explains the difference between queuing (point‑to‑point) and pub/sub (fan‑out) and shows how to enable each pattern in Danube using subscriptions and dispatch modes.

- For subscription details, see `architecture/subscriptions.md`.
- For delivery semantics, see `architecture/dispatch_strategy.md`.

## Concepts at a Glance

**Queuing (Point‑to‑Point)**
  - One message is delivered to exactly one consumer in a group.
  - Suited for work distribution (orders, tasks, jobs).
  - Ordering is per consumer, and per partition when topics are partitioned.

**Pub/Sub (Fan‑out)**
  - Every subscription receives every message.
  - Suited for broadcast (notifications, analytics, multiple services reacting to events).
  - Each subscription processes the full stream independently.

## How to enable in Danube

Both patterns are built with Danube subscriptions on a topic.

**Queuing (Point‑to‑Point)**
  - Set subscription type to `Shared`.
  - Use the same subscription name across all workers (e.g., `orders-workers`).
  - Result: Each message is delivered to one consumer in the group (round‑robin, per partition).

**Pub/Sub (Fan‑out)**
  - Create distinct subscriptions per downstream consumer/team (unique subscription names), typically using `Exclusive` (or `Failover` for HA).
  - Result: Each subscription receives every message; consumers do not contend with other subscriptions.

## Dispatch modes and guarantees

**Non‑Reliable (Pub/Sub‑style, best‑effort)**
  - Lowest latency; no persistence or replay.
  - Messages are delivered only to active consumers. Disconnections may cause loss.

**Reliable (Streaming‑style, at‑least‑once)**
  - Messages are appended to the local WAL and asynchronously uploaded to cloud storage.
  - Consumers can replay historical data; acknowledgments drive redelivery.

See `architecture/persistence.md` for WAL + Cloud details.

## Partitioning behavior

With partitioned topics, both patterns apply per partition.
  - Queuing: messages are distributed round‑robin within each partition to the consumers in that subscription.
  - Pub/Sub: each subscription receives each partition’s messages independently.

## Quick examples

**Build a worker queue**
  - Topic: `/default/orders`
  - Subscription: `orders-workers` (type `Shared`)
  - Run N consumers with the same subscription name. Messages are load‑balanced.

**Broadcast to multiple services**
  - Topic: `/default/events`
  - Subscriptions: `billing` (Exclusive), `analytics` (Exclusive), `monitoring` (Failover)
  - Each subscription receives the full event stream independently.

## Notes & best practices

- Use Reliable mode for durability and replay; Non‑Reliable for minimal latency when loss is acceptable.
- Prefer `Failover` over `Exclusive` when you need quick takeover on consumer failure.
- Size partitions to match consumer parallelism for optimal throughput.
