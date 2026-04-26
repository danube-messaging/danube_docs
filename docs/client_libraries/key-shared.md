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
    let mut producer = client
        .new_producer()
        .with_topic("/default/orders")
        .with_name("orders_producer")
        .with_reliable_dispatch()
        .build()?;

    producer.create().await?;

    // All "payment" messages go to the same consumer
    producer.send_with_key(b"Payment for #1001".to_vec(), None, "payment").await?;

    // "shipping" messages go to (potentially) a different consumer
    producer.send_with_key(b"Order #1001 shipped".to_vec(), None, "shipping").await?;

    // Same key = same consumer, guaranteed
    producer.send_with_key(b"Payment for #1002".to_vec(), None, "payment").await?;
    ```

=== "Go"

    ```go
    producer, err := client.NewProducer().
        WithName("orders_producer").
        WithTopic("/default/orders").
        WithDispatchStrategy(danube.NewReliableDispatchStrategy()).
        Build()
    producer.Create(ctx)

    // All "payment" messages go to the same consumer
    producer.SendWithKey(ctx, []byte("Payment for #1001"), nil, "payment")

    // "shipping" messages go to (potentially) a different consumer
    producer.SendWithKey(ctx, []byte("Order #1001 shipped"), nil, "shipping")

    // Same key = same consumer, guaranteed
    producer.SendWithKey(ctx, []byte("Payment for #1002"), nil, "payment")
    ```

=== "Python"

    ```python
    producer = (
        client.new_producer()
        .with_topic("/default/orders")
        .with_name("orders_producer")
        .with_dispatch_strategy(DispatchStrategy.RELIABLE)
        .build()
    )
    await producer.create()

    # All "payment" messages go to the same consumer
    await producer.send_with_key(b"Payment for #1001", None, "payment")

    # "shipping" messages go to (potentially) a different consumer
    await producer.send_with_key(b"Order #1001 shipped", None, "shipping")

    # Same key = same consumer, guaranteed
    await producer.send_with_key(b"Payment for #1002", None, "payment")
    ```

=== "Java"

    ```java
    Producer producer = client.newProducer()
            .withTopic("/default/orders")
            .withName("orders_producer")
            .withDispatchStrategy(DispatchStrategy.RELIABLE)
            .build();
    producer.create();

    // All "payment" messages go to the same consumer
    producer.sendWithKey("Payment for #1001".getBytes(), Map.of(), "payment");

    // "shipping" messages go to (potentially) a different consumer
    producer.sendWithKey("Order #1001 shipped".getBytes(), Map.of(), "shipping");

    // Same key = same consumer, guaranteed
    producer.sendWithKey("Payment for #1002".getBytes(), Map.of(), "payment");
    ```

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
    let mut consumer = client
        .new_consumer()
        .with_topic("/default/orders")
        .with_consumer_name("worker_1")
        .with_subscription("orders_sub")
        .with_subscription_type(SubType::KeyShared)
        .build()?;

    consumer.subscribe().await?;
    let mut stream = consumer.receive().await?;

    while let Some(message) = stream.recv().await {
        let key = message.routing_key.as_deref().unwrap_or("<none>");
        let payload = String::from_utf8_lossy(&message.payload);
        println!("📥 key={:<10} | '{}'", key, payload);
        consumer.ack(&message).await?;
    }
    ```

=== "Go"

    ```go
    consumer, err := client.NewConsumer().
        WithConsumerName("worker_1").
        WithTopic("/default/orders").
        WithSubscription("orders_sub").
        WithSubscriptionType(danube.KeyShared).
        Build()

    consumer.Subscribe(ctx)
    stream, _ := consumer.Receive(ctx)

    for msg := range stream {
        fmt.Printf("📥 key=%s payload=%s\n",
            msg.GetRoutingKey(), string(msg.GetPayload()))
        consumer.Ack(ctx, msg)
    }
    ```

=== "Python"

    ```python
    consumer = (
        client.new_consumer()
        .with_topic("/default/orders")
        .with_consumer_name("worker_1")
        .with_subscription("orders_sub")
        .with_subscription_type(SubType.KEY_SHARED)
        .build()
    )

    await consumer.subscribe()
    queue = await consumer.receive()

    while True:
        message = await queue.get()
        key = message.routing_key if message.HasField("routing_key") else ""
        print(f"📥 key={key} payload={message.payload.decode()}")
        await consumer.ack(message)
    ```

=== "Java"

    ```java
    Consumer consumer = client.newConsumer()
            .withTopic("/default/orders")
            .withConsumerName("worker_1")
            .withSubscription("orders_sub")
            .withSubscriptionType(SubType.KEY_SHARED)
            .build();

    consumer.subscribe();

    consumer.receive().subscribe(new Flow.Subscriber<>() {
        @Override public void onSubscribe(Flow.Subscription s) { s.request(Long.MAX_VALUE); }

        @Override
        public void onNext(StreamMessage msg) {
            String key = msg.routingKey() != null ? msg.routingKey() : "";
            System.out.printf("📥 key=%s payload=%s%n", key, new String(msg.payload()));
            consumer.ack(msg);
        }

        @Override public void onError(Throwable t) {}
        @Override public void onComplete() {}
    });
    ```

### Running Multiple Consumers

Start multiple consumers with the same subscription to see key distribution:

=== "Rust"

    ```bash
    # Terminal 1 & 2 — consumers
    cargo run --example key_shared_consumer

    # Terminal 3 — producer
    cargo run --example key_shared_producer
    ```

=== "Go"

    ```bash
    # Terminal 1 & 2 — consumers
    go run examples/key_shared/consumer/consumer.go

    # Terminal 3 — producer
    go run examples/key_shared/producer/producer.go
    ```

=== "Python"

    ```bash
    # Terminal 1 & 2 — consumers
    python examples/key_shared_consumer.py

    # Terminal 3 — producer
    python examples/key_shared_producer.py
    ```

=== "Java"

    ```bash
    # Terminal 1 & 2 — consumers
    java examples/KeySharedConsumer.java

    # Terminal 3 — producer
    java examples/KeySharedProducer.java
    ```

**Expected output:**

Each consumer receives a distinct subset of routing keys:

```
# consumer_1 output
📥 key=shipping   | 'Order #1001 shipped via express'
📥 key=shipping   | 'Order #1002 shipped via standard'

# consumer_2 output
📥 key=payment    | 'Payment received for order #1001'
📥 key=payment    | 'Payment received for order #1002'
📥 key=invoice    | 'Invoice generated for order #1001'
```

The exact key-to-consumer assignment depends on consistent hashing but is deterministic — re-running produces the same distribution.

---

## Consumer: Key Filtering

Key filtering gives consumers explicit control over which keys they handle. Each consumer declares glob patterns and only receives messages whose routing key matches.

=== "Rust"

    ```rust
    // This consumer only receives "payment" and "invoice" messages
    let mut consumer = client
        .new_consumer()
        .with_topic("/default/orders")
        .with_consumer_name("payments_worker")
        .with_subscription("orders_filtered")
        .with_subscription_type(SubType::KeyShared)
        .with_key_filter("payment")
        .with_key_filter("invoice")
        .build()?;

    consumer.subscribe().await?;
    let mut stream = consumer.receive().await?;

    while let Some(message) = stream.recv().await {
        let key = message.routing_key.as_deref().unwrap_or("<none>");
        println!("📥 key={:<10} | '{}'", key, String::from_utf8_lossy(&message.payload));
        consumer.ack(&message).await?;
    }
    ```

=== "Go"

    ```go
    // This consumer only receives "payment" and "invoice" messages
    consumer, err := client.NewConsumer().
        WithConsumerName("payments_worker").
        WithTopic("/default/orders").
        WithSubscription("orders_filtered").
        WithSubscriptionType(danube.KeyShared).
        WithKeyFilter("payment").
        WithKeyFilter("invoice").
        Build()

    consumer.Subscribe(ctx)
    stream, _ := consumer.Receive(ctx)

    for msg := range stream {
        fmt.Printf("📥 key=%s payload=%s\n",
            msg.GetRoutingKey(), string(msg.GetPayload()))
        consumer.Ack(ctx, msg)
    }
    ```

=== "Python"

    ```python
    # This consumer only receives "payment" and "invoice" messages
    consumer = (
        client.new_consumer()
        .with_topic("/default/orders")
        .with_consumer_name("payments_worker")
        .with_subscription("orders_filtered")
        .with_subscription_type(SubType.KEY_SHARED)
        .with_key_filter("payment")
        .with_key_filter("invoice")
        .build()
    )

    await consumer.subscribe()
    queue = await consumer.receive()

    while True:
        message = await queue.get()
        key = message.routing_key if message.HasField("routing_key") else ""
        print(f"📥 key={key} payload={message.payload.decode()}")
        await consumer.ack(message)
    ```

=== "Java"

    ```java
    // This consumer only receives "payment" and "invoice" messages
    Consumer consumer = client.newConsumer()
            .withTopic("/default/orders")
            .withConsumerName("payments_worker")
            .withSubscription("orders_filtered")
            .withSubscriptionType(SubType.KEY_SHARED)
            .withKeyFilter("payment")
            .withKeyFilter("invoice")
            .build();

    consumer.subscribe();

    consumer.receive().subscribe(new Flow.Subscriber<>() {
        @Override public void onSubscribe(Flow.Subscription s) { s.request(Long.MAX_VALUE); }

        @Override
        public void onNext(StreamMessage msg) {
            String key = msg.routingKey() != null ? msg.routingKey() : "";
            System.out.printf("📥 key=%s payload=%s%n", key, new String(msg.payload()));
            consumer.ack(msg);
        }

        @Override public void onError(Throwable t) {}
        @Override public void onComplete() {}
    });
    ```

### Filter Patterns

Filters use glob syntax:

| Pattern | Matches |
|---------|---------|
| `"payment"` | Exact match only |
| `"ship*"` | `"shipping"`, `"shipment"`, etc. |
| `"eu-west-?"` | `"eu-west-1"`, `"eu-west-2"`, etc. |
| `"*"` | Everything (same as no filter) |

### Multiple Filters

You can set multiple filters at once or chain individual calls:

=== "Rust"

    ```rust
    // Chain individual filters
    .with_key_filter("payment")
    .with_key_filter("invoice")

    // Or set multiple at once
    .with_key_filters(vec!["payment".into(), "invoice".into()])
    ```

=== "Go"

    ```go
    // Chain individual filters
    .WithKeyFilter("payment").
    .WithKeyFilter("invoice")

    // Or set multiple at once
    .WithKeyFilters([]string{"payment", "invoice"})
    ```

=== "Python"

    ```python
    # Chain individual filters
    .with_key_filter("payment")
    .with_key_filter("invoice")

    # Or set multiple at once
    .with_key_filters(["payment", "invoice"])
    ```

=== "Java"

    ```java
    // Chain individual filters
    .withKeyFilter("payment")
    .withKeyFilter("invoice")

    // Or set multiple at once
    .withKeyFilters(List.of("payment", "invoice"))
    ```

### Expected Filtered Output

```
# payments consumer (filters: "payment", "invoice")
📥 key=payment    | 'Payment received for order #1001'
📥 key=payment    | 'Payment received for order #1002'
📥 key=invoice    | 'Invoice generated for order #1001'

# logistics consumer (filters: "ship*", "return")
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

Route all events for the same order to the same consumer:

=== "Rust"

    ```rust
    producer.send_with_key(data, None, &format!("order-{}", order_id)).await?;
    ```

=== "Go"

    ```go
    producer.SendWithKey(ctx, data, nil, fmt.Sprintf("order-%s", orderID))
    ```

=== "Python"

    ```python
    await producer.send_with_key(data, None, f"order-{order_id}")
    ```

=== "Java"

    ```java
    producer.sendWithKey(data, Map.of(), "order-" + orderId);
    ```

### Multi-Tenant Workloads

Dedicate consumers to specific tenants using key filtering:

=== "Rust"

    ```rust
    .with_key_filter("tenant-a-*")  // Tenant A consumer
    .with_key_filter("tenant-b-*")  // Tenant B consumer
    ```

=== "Go"

    ```go
    .WithKeyFilter("tenant-a-*")  // Tenant A consumer
    .WithKeyFilter("tenant-b-*")  // Tenant B consumer
    ```

=== "Python"

    ```python
    .with_key_filter("tenant-a-*")  # Tenant A consumer
    .with_key_filter("tenant-b-*")  # Tenant B consumer
    ```

=== "Java"

    ```java
    .withKeyFilter("tenant-a-*")  // Tenant A consumer
    .withKeyFilter("tenant-b-*")  // Tenant B consumer
    ```

### IoT Device Routing

Route device telemetry by device ID for per-device state tracking:

=== "Rust"

    ```rust
    producer.send_with_key(data, None, &format!("device-{}", device_id)).await?;
    ```

=== "Go"

    ```go
    producer.SendWithKey(ctx, data, nil, fmt.Sprintf("device-%s", deviceID))
    ```

=== "Python"

    ```python
    await producer.send_with_key(data, None, f"device-{device_id}")
    ```

=== "Java"

    ```java
    producer.sendWithKey(data, Map.of(), "device-" + deviceId);
    ```

---

## Full Examples

For complete runnable examples, see the client repositories:

- [Rust examples](https://github.com/danube-messaging/danube/tree/main/danube-client/examples)
- [Go examples](https://github.com/danube-messaging/danube-go/tree/main/examples/key_shared)
- [Python examples](https://github.com/danube-messaging/danube-py/tree/main/examples)
- [Java examples](https://github.com/danube-messaging/danube-java/tree/main/examples)
