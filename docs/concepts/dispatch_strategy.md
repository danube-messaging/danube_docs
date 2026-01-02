# Dispatch Strategy

The dispatch strategies in Danube represent two distinct approaches to message delivery, each serving different use cases:

### Non-Reliable Dispatch Strategy

This strategy prioritizes speed and minimal resource usage by delivering messages directly from producers to subscribers without persistence. Messages flow through the broker in a "fire and forget" manner, achieving the lowest possible latency. It fits real-time metrics, live telemetry, or any workload where occasional loss is acceptable.

**Writer path (producer)**

  - The producer sends a message to the broker specifying the topic.
  - The broker validates and routes the message to the topic’s dispatcher.
  - Depending on subscription type (Exclusive/Shared/Failover), the dispatcher selects the target consumer(s).
  - The message is immediately forwarded to consumer channels. There is no on-disk persistence and no acknowledgment gating.

**Reader path (consumer)**

  - A consumer subscribes to a topic under an existing subscription (Exclusive/Shared/Failover).
  - The broker registers the consumer and attaches a live message stream to it.
  - The dispatcher pushes incoming messages directly to the consumer stream.
  - Acknowledgments are optional and do not affect delivery; if a consumer disconnects, messages in flight may be lost.

### Reliable Dispatch Strategy

This strategy ensures at-least-once delivery using a WAL + Cloud store-and-forward design. Messages are appended to a local Write-Ahead Log (WAL) and asynchronously uploaded to cloud object storage. Delivery is coordinated by the subscription engine, which tracks progress and acknowledgments per subscription.

**Writer path (producer)**

  - The producer sends a message to the broker for a reliable topic.
  - The message is appended to the local WAL (durable on disk) and becomes eligible for dispatch.
  - The dispatcher prepares the message for the subscription type (Exclusive/Shared/Failover) while the subscription engine records it as pending.
  - A background uploader asynchronously persists WAL frames to cloud object storage; this does not block producers.

**Reader path (consumer)**

  - A consumer subscribes to a reliable topic; the broker attaches a stream and initializes subscription progress.
  - The dispatcher delivers messages according to the subscription type and ordering guarantees.
  - The consumer acknowledges processed messages; the subscription engine advances progress and triggers redelivery if needed.
  - If the consumer is late or reconnects after a gap, historical data is replayed from the WAL or, if needed, from cloud storage, then seamlessly handed off to the live WAL tail.

These strategies embody Danube’s flexibility, letting you choose the right balance between performance and reliability per topic. You can run non-reliable and reliable topics side by side in the same cluster.
