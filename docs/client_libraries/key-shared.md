# Key-Shared Subscriptions Guide

Key-Shared subscriptions route messages to consumers based on their **routing key**. All messages with the same key are guaranteed to be delivered to the same consumer, in order. This combines the parallelism of Shared subscriptions with per-key ordering guarantees.

For architecture details, see [Key-Shared Dispatch Architecture](../architecture/key_shared_architecture.md).

## When to Use Key-Shared

Key-Shared is the right choice when you need:

- **Per-key ordering** with **multi-consumer parallelism** : messages for the same key are processed sequentially, but different keys are processed in parallel
- **Key-affinity processing** : all events for a given entity (order, user, device) go to the same consumer
- **Selective routing** : specific consumers handle specific subsets of keys via filtering

### Comparison with Other Subscription Types

| Requirement | Subscription Type |
|-------------|------------------|
| Total ordering, single consumer | **Exclusive** |
| Maximum throughput, no ordering | **Shared** |
| Ordering + high availability | **Failover** |
| Per-key ordering + parallelism | **Key-Shared** |

---

## Producer: Sending with Routing Keys

Tag each message with a routing key using `send_with_key()`. Messages sent with the same key will always be delivered to the same consumer.

=== "Rust"

    ```rust
    use anyhow::Result;
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<()> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut producer = client
            .new_producer()
            .with_topic("/default/orders")
            .with_name("orders_producer")
            .with_reliable_dispatch()
            .build()?;

        producer.create().await?;

        // Send messages with routing keys
        // All "payment" messages go to the same consumer
        producer
            .send_with_key(
                b"Payment received for order #1001".to_vec(),
                None,        // no attributes
                "payment",   // routing key
            )
            .await?;

        // "shipping" messages go to (potentially) a different consumer
        producer
            .send_with_key(
                b"Order #1001 shipped".to_vec(),
                None,
                "shipping",
            )
            .await?;

        // Same key = same consumer, guaranteed
        producer
            .send_with_key(
                b"Payment received for order #1002".to_vec(),
                None,
                "payment",
            )
            .await?;

        Ok(())
    }
    ```

=== "Go"

    !!! note "Coming soon"
        Key-Shared support for the Go client is under development.

=== "Python"

    !!! note "Coming soon"
        Key-Shared support for the Python client is under development.

=== "Java"

    !!! note "Coming soon"
        Key-Shared support for the Java client is under development.

**Key points:**

- `send_with_key(payload, attributes, routing_key)` tags the message with a routing key
- Messages with the same routing key always go to the same consumer
- If you use `send()` instead of `send_with_key()`, the producer name is used as the fallback routing key
- Routing keys can be any string: order IDs, user IDs, device IDs, event types, etc.

---

## Consumer: Automatic Key Distribution

The simplest Key-Shared consumer uses automatic key distribution via consistent hashing. The broker assigns keys to consumers automatically, no configuration needed.

=== "Rust"

    ```rust
    use anyhow::Result;
    use danube_client::{DanubeClient, SubType};

    #[tokio::main]
    async fn main() -> Result<()> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut consumer = client
            .new_consumer()
            .with_topic("/default/orders")
            .with_consumer_name("worker_1")
            .with_subscription("orders_sub")
            .with_subscription_type(SubType::KeyShared)
            .build()?;

        consumer.subscribe().await?;
        println!("✅ Consumer subscribed (Key-Shared)");

        let mut stream = consumer.receive().await?;

        while let Some(message) = stream.recv().await {
            let payload = String::from_utf8_lossy(&message.payload);
            let key = message.routing_key.as_deref().unwrap_or("<none>");

            println!("📥 key={:<10} | '{}'", key, payload);

            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

=== "Go"

    !!! note "Coming soon"
        Key-Shared support for the Go client is under development.

=== "Python"

    !!! note "Coming soon"
        Key-Shared support for the Python client is under development.

=== "Java"

    !!! note "Coming soon"
        Key-Shared support for the Java client is under development.

### Running Multiple Consumers

Start multiple consumers with the same subscription to see key distribution:

```bash
# Terminal 1
cargo run --example key_shared_consumer -- consumer_1

# Terminal 2
cargo run --example key_shared_consumer -- consumer_2

# Terminal 3 — start the producer
cargo run --example key_shared_producer
```

**Expected output:**

Each consumer receives a distinct subset of routing keys. For example:

```
# consumer_1 output
📥 key=shipping   | 'Order #1001 shipped via express'
📥 key=return     | 'Return request for order #998'
📥 key=shipping   | 'Order #1002 shipped via standard'

# consumer_2 output
📥 key=payment    | 'Payment received for order #1001'
📥 key=payment    | 'Payment received for order #1002'
📥 key=invoice    | 'Invoice generated for order #1001'
```

The exact key-to-consumer assignment depends on consistent hashing but is deterministic — re-running produces the same distribution.

---

## Consumer: Key Filtering

Key filtering gives consumers explicit control over which keys they handle. Instead of relying on automatic consistent hashing, each consumer declares glob patterns and only receives messages whose routing key matches.

=== "Rust"

    ```rust
    use anyhow::Result;
    use danube_client::{DanubeClient, SubType};

    #[tokio::main]
    async fn main() -> Result<()> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        // This consumer only receives "payment" and "invoice" messages
        let mut consumer = client
            .new_consumer()
            .with_topic("/default/orders")
            .with_consumer_name("payments_worker")
            .with_subscription("orders_filtered")
            .with_subscription_type(SubType::KeyShared)
            .with_key_filter("payment")    // exact match
            .with_key_filter("invoice")    // exact match
            .build()?;

        consumer.subscribe().await?;
        println!("✅ Payments consumer subscribed");

        let mut stream = consumer.receive().await?;

        while let Some(message) = stream.recv().await {
            let payload = String::from_utf8_lossy(&message.payload);
            let key = message.routing_key.as_deref().unwrap_or("<none>");

            println!("📥 key={:<10} | '{}'", key, payload);

            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

=== "Go"

    !!! note "Coming soon"
        Key-Shared support for the Go client is under development.

=== "Python"

    !!! note "Coming soon"
        Key-Shared support for the Python client is under development.

=== "Java"

    !!! note "Coming soon"
        Key-Shared support for the Java client is under development.

### Filter Patterns

Filters use glob syntax:

| Pattern | Matches |
|---------|---------|
| `"payment"` | Exact match only |
| `"ship*"` | `"shipping"`, `"shipment"`, etc. |
| `"eu-west-?"` | `"eu-west-1"`, `"eu-west-2"`, etc. |
| `"*"` | Everything (same as no filter) |

### Multiple Filters

Use `with_key_filters()` to set multiple filters at once:

```rust
let filters = vec![
    "payment".to_string(),
    "invoice".to_string(),
];

let mut consumer = client
    .new_consumer()
    .with_topic("/default/orders")
    .with_consumer_name("payments_worker")
    .with_subscription("orders_sub")
    .with_subscription_type(SubType::KeyShared)
    .with_key_filters(filters)
    .build()?;
```

Or chain individual `with_key_filter()` calls:

```rust
let mut consumer = client
    .new_consumer()
    .with_topic("/default/orders")
    .with_consumer_name("payments_worker")
    .with_subscription("orders_sub")
    .with_subscription_type(SubType::KeyShared)
    .with_key_filter("payment")
    .with_key_filter("invoice")
    .build()?;
```

### Running Filtered Consumers

```bash
# Terminal 1 — payments consumer (filters: "payment", "invoice")
cargo run --example key_shared_filtered_consumer -- payments

# Terminal 2 — logistics consumer (filters: "ship*", "return")
cargo run --example key_shared_filtered_consumer -- logistics

# Terminal 3 — start the producer
cargo run --example key_shared_producer
```

**Expected output:**

```
# payments consumer
📥 key=payment    | 'Payment received for order #1001'
📥 key=payment    | 'Payment received for order #1002'
📥 key=invoice    | 'Invoice generated for order #1001'

# logistics consumer
📥 key=shipping   | 'Order #1001 shipped via express'
📥 key=return     | 'Return request for order #998'
📥 key=shipping   | 'Order #1002 shipped via standard'
```

---

## Design Behavior

### Per-Key Ordering

At most **one message per routing key** is in-flight at any time. If a second message arrives for a key that already has an unacknowledged message, the second message is queued until the first is acked. This guarantees per-key ordering even with multiple keys being processed in parallel.

### Consumer Elasticity

When consumers join or leave, key-to-consumer mappings are rebalanced using consistent hashing. Only approximately **1/N** of existing key assignments are remapped (where N is the number of consumers). The rest remain stable.

### Unmatched Keys

If a message's routing key matches no consumer's filters, the message is skipped by this subscription. The offset is treated as implicitly acked and the cursor advances past it. The message still remains in the topic and is available for delivery to other subscriptions.

### Acknowledgment and Retry

Key-Shared supports the same ack/nack semantics as other reliable subscription types:

- **Ack**: confirms successful processing, frees the key lock, advances the cursor
- **Nack**: rejects the message with an optional delay and reason, triggers redelivery according to the failure policy

For failure policy configuration, see [Dispatch Strategy - Failure Handling](../concepts/dispatch_strategy.md).

---

## Use Cases

### Order Processing

Route all events for the same order to the same consumer for consistent state management:

```rust
// Producer
producer.send_with_key(data, None, &format!("order-{}", order_id)).await?;

// Consumer — processes all events for its assigned order IDs
```

### Per-User Event Streams

Group user activity by user ID across a pool of consumers:

```rust
producer.send_with_key(data, None, &format!("user-{}", user_id)).await?;
```

### Multi-Tenant Workloads

Dedicate consumers to specific tenants using key filtering:

```rust
// Tenant A consumer
consumer.with_key_filter("tenant-a-*").build()?;

// Tenant B consumer
consumer.with_key_filter("tenant-b-*").build()?;
```

### IoT Device Routing

Route device telemetry by device ID for per-device state tracking:

```rust
producer.send_with_key(data, None, &format!("device-{}", device_id)).await?;
```

---

## Full Examples

For complete runnable examples, see:

- Rust: [key_shared_producer.rs](https://github.com/danube-messaging/danube/tree/main/danube-client/examples/key_shared_producer.rs), [key_shared_consumer.rs](https://github.com/danube-messaging/danube/tree/main/danube-client/examples/key_shared_consumer.rs), [key_shared_filtered_consumer.rs](https://github.com/danube-messaging/danube/tree/main/danube-client/examples/key_shared_filtered_consumer.rs)
