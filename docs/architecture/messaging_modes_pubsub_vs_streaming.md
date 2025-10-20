# Pub/Sub vs Streaming: Concepts and Danube Support

This page compares Pub/Sub messaging and Streaming architectures, highlights key differences and use cases, and indicates what Danube supports. Use it as a conceptual guide aligned with Danube’s Reliable and Non‑Reliable dispatch modes.

## Pub/Sub messaging

### Purpose and Use Cases

* **Purpose**: Designed for decoupling producers and consumers, enabling asynchronous communication between different parts of a system.
* **Use Cases**: Event-driven architectures, real-time notifications, decoupled microservices, and distributed systems. Suitable when low latency is critical and some message loss is acceptable (e.g., monitoring, telemetry, ephemeral chat).

**Danube support**: Yes, via Non‑Reliable dispatch.

  - Delivery: best‑effort (at‑most‑once).
  - Persistence: none; messages are not stored.
  - Ordering: per topic/partition within a dispatcher.

### Architecture and Design

* **Components**: Producers, Subscriptions (consumer groups), and the broker.
* **Message Flow**: Producers send messages to a broker, which distributes them to active subscribers according to subscription semantics.
* **Scaling**: Scales by adding more brokers or distributing load (topics / partitions) across multiple brokers.

### Data Handling and Processing Models

#### Pub/Sub messaging Producers

* **Low Latency**: No disk persistence on the hot path; minimal overhead.
* **Ordering**: Preserved per topic/partition through the dispatcher.
* **Delivery**: Best‑effort; messages can be lost on failure.
* **Acks**: Immediate, based on in‑memory handling.
* **No Active Consumers**: Publishing is allowed; messages are dropped if no consumers are attached.

**Danube**: Matches Non‑Reliable dispatch semantics.

#### Pub/Sub messaging Consumers

* **Real-time**: Messages are delivered only to connected consumers.
* **No Replay**: No historical fetch; process as they arrive.
* **Throughput**: High, with low overhead.
* **Delivery**: Only to active subscribers; otherwise dropped.

**Danube**: Supported via Exclusive, Shared, Failover subscriptions in Non‑Reliable mode.

#### Order of Operations of Pub/Sub messaging

* **Producer Publishes Message**: The producer sends a message to the broker.
* **Broker Receives Message**: The broker processes the message.
* **Consumer Availability Check**: If consumers are available, the message is delivered to them in real-time.
* **No Consumers**: If no consumers are connected, the message is discarded.

## Streaming design considerations

### Purpose and Use Cases

* **Purpose**: Designed for processing and analyzing large volumes of data in real-time as it is generated.
* **Use Cases**: Real-time analytics, data pipelines, event sourcing, continuous data processing, and stream processing applications. Ideal when high reliability and durability are required (e.g., financial transactions, orders, audit logs).

**Danube support**: Yes, via Reliable dispatch (WAL + Cloud persistence).

  - Delivery: at‑least‑once with ack‑gating.
  - Persistence: local WAL for hot path, background uploads to object storage for durability and replay.
  - Replay: historical fetch from WAL or Cloud with seamless handoff to live tail.

### Architecture and Design

* **Components**: Producers, consumers, stream processors, and a durable log.
* **Data Flow**: Producers append to a log; consumers/processors read continuously with tracked progress.
* **Scaling**: Designed to handle high throughput and scale horizontally by partitioning data across multiple nodes in a cluster.

### Data Handling and Processing Models

### Streaming Producers

* **Durability**: Messages are appended to a persistent log. In Danube, the hot path is a local WAL with asynchronous uploads to cloud object storage, enabling replay and recovery.
* **Acknowledgements**: Acks are returned once the message is durably appended to the WAL (replication depends on deployment/storage backend). Latency is higher than non‑reliable but optimized for sub‑second dispatch.
* **Ordering**: Preserved per topic/partition.
* **Delivery Guarantees**: At‑least‑once (with redelivery on failure).
* **Publishing Without Consumers**: Allowed; messages are stored and remain available for later consumption within retention.
* **Processing**: Compatible with stream processing patterns (e.g., windowing, aggregations) layered above the durable log.

### Streaming Consumers

* **Replay & Retention**: Consumers can fetch historical data within retention. In Danube, reads come from WAL if available, otherwise from Cloud with seamless handoff to the live tail.
* **Acknowledgements**: Consumers ack processed messages; the broker tracks subscription progress and manages redelivery.
* **Fault Tolerance**: On crash/reconnect, consumers resume from the last acknowledged position.
* **Availability**: Messages remain available even with no active consumers, subject to retention policies.

### Order of Operations of Streaming

* **Producer Publishes Message**: The producer sends a message to the broker.
* **Durable Append**: The broker appends the message to the WAL (durable on disk); background tasks upload frames to cloud storage.
* **Producer Acknowledgment**: The broker acknowledges once the append is durable.
* **Delivery to Consumers**: Messages are dispatched to consumers, gated by acknowledgments for reliable delivery.
* **No Consumers**: Messages remain stored and available for later consumption within retention.

