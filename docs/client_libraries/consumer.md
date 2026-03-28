# Consumer Guide

Consumers receive messages from Danube topics via subscriptions. This guide covers both basic and advanced consumer capabilities.

## Creating a Consumer

### Simple Consumer

The minimal setup to receive messages:

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SubType};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut consumer = client
            .new_consumer()
            .with_topic("/default/my-topic")
            .with_consumer_name("my-consumer")
            .with_subscription("my-subscription")
            .with_subscription_type(SubType::Exclusive)
            .build()?;

        consumer.subscribe().await?;
        println!("✅ Consumer subscribed");

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "context"
        "fmt"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("Failed to create client: %v", err)
        }

        ctx := context.Background()

        consumer, err := client.NewConsumer().
            WithConsumerName("my-consumer").
            WithTopic("/default/my-topic").
            WithSubscription("my-subscription").
            WithSubscriptionType(danube.Exclusive).
            Build()
        if err != nil {
            log.Fatalf("Failed to create consumer: %v", err)
        }

        if err := consumer.Subscribe(ctx); err != nil {
            log.Fatalf("Failed to subscribe: %v", err)
        }

        fmt.Println("✅ Consumer subscribed")
    }
    ```

=== "Python"

    ```python
    import asyncio
    from danube import DanubeClientBuilder, SubType

    async def main():
        client = await (
            DanubeClientBuilder()
            .service_url("http://127.0.0.1:6650")
            .build()
        )

        consumer = (
            client.new_consumer()
            .with_topic("/default/my-topic")
            .with_consumer_name("my-consumer")
            .with_subscription("my-subscription")
            .with_subscription_type(SubType.EXCLUSIVE)
            .build()
        )

        await consumer.subscribe()
        print("✅ Consumer subscribed")

    asyncio.run(main())
    ```

=== "Java"

    ```java
    import com.danubemessaging.client.Consumer;
    import com.danubemessaging.client.DanubeClient;
    import com.danubemessaging.client.SubType;

    public class Main {
        public static void main(String[] args) throws Exception {
            DanubeClient client = DanubeClient.builder()
                    .serviceUrl("http://127.0.0.1:6650")
                    .build();

            Consumer consumer = client.newConsumer()
                    .withTopic("/default/my-topic")
                    .withConsumerName("my-consumer")
                    .withSubscription("my-subscription")
                    .withSubscriptionType(SubType.EXCLUSIVE)
                    .build();

            consumer.subscribe();
            System.out.println("Consumer subscribed");

            client.close();
        }
    }
    ```

**Key concepts:**

- **Topic:** Source of messages
- **Consumer Name:** Unique identifier for this consumer instance
- **Subscription:** Logical grouping of consumers (multiple consumers can share)
- **Subscription Type:** Controls how messages are distributed (Exclusive, Shared, Failover)

---

## Subscription Types

### Exclusive

Only **one consumer** can be active. Guarantees message order.

**Characteristics:**

- ✅ Message ordering guaranteed
- ✅ Simple failure handling
- ❌ No horizontal scaling
- **Use case:** Order processing, sequential workflows

### Shared

**Multiple consumers** receive messages in round-robin. Scales horizontally.

**Characteristics:**

- ✅ Horizontal scaling
- ✅ Load distribution
- ❌ No ordering guarantee
- **Use case:** Log processing, analytics, parallel processing

### Failover

Like Exclusive, but allows **standby consumers**. One active, others wait.

**Characteristics:**

- ✅ Message ordering guaranteed
- ✅ Automatic failover to standby
- ✅ High availability
- **Use case:** Critical services needing HA with ordering

---

## Receiving Messages

### Basic Message Loop

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SubType};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut consumer = client
            .new_consumer()
            .with_topic("/default/events")
            .with_consumer_name("event-processor")
            .with_subscription("event-sub")
            .with_subscription_type(SubType::Exclusive)
            .build()?;

        consumer.subscribe().await?;

        // Start receiving
        let mut message_stream = consumer.receive().await?;

        while let Some(message) = message_stream.recv().await {
            // Access message data
            let payload = String::from_utf8_lossy(&message.payload);
            println!("📥 Received: {}", payload);

            // Acknowledge the message
            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "context"
        "fmt"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("Failed to create client: %v", err)
        }

        ctx := context.Background()

        consumer, err := client.NewConsumer().
            WithConsumerName("event-processor").
            WithTopic("/default/events").
            WithSubscription("event-sub").
            WithSubscriptionType(danube.Exclusive).
            Build()
        if err != nil {
            log.Fatalf("Failed to create consumer: %v", err)
        }

        if err := consumer.Subscribe(ctx); err != nil {
            log.Fatalf("Failed to subscribe: %v", err)
        }

        // Start receiving
        stream, err := consumer.Receive(ctx)
        if err != nil {
            log.Fatalf("Failed to receive: %v", err)
        }

        for msg := range stream {
            payload := string(msg.GetPayload())
            fmt.Printf("📥 Received: %s\n", payload)

            // Acknowledge the message
            if _, err := consumer.Ack(ctx, msg); err != nil {
                log.Printf("Failed to ack: %v\n", err)
            }
        }
    }
    ```

=== "Python"

    ```python
    import asyncio
    from danube import DanubeClientBuilder, SubType

    async def main():
        client = await (
            DanubeClientBuilder()
            .service_url("http://127.0.0.1:6650")
            .build()
        )

        consumer = (
            client.new_consumer()
            .with_topic("/default/events")
            .with_consumer_name("event-processor")
            .with_subscription("event-sub")
            .with_subscription_type(SubType.EXCLUSIVE)
            .build()
        )

        await consumer.subscribe()

        # Start receiving
        queue = await consumer.receive()

        while True:
            message = await queue.get()
            payload = message.payload.decode()
            print(f"📥 Received: {payload}")

            # Acknowledge the message
            await consumer.ack(message)

    asyncio.run(main())
    ```

=== "Java"

    ```java
    import com.danubemessaging.client.Consumer;
    import com.danubemessaging.client.DanubeClient;
    import com.danubemessaging.client.SubType;
    import com.danubemessaging.client.model.StreamMessage;
    import java.util.concurrent.CountDownLatch;
    import java.util.concurrent.Flow;

    DanubeClient client = DanubeClient.builder()
            .serviceUrl("http://127.0.0.1:6650")
            .build();

    Consumer consumer = client.newConsumer()
            .withTopic("/default/events")
            .withConsumerName("event-processor")
            .withSubscription("event-sub")
            .withSubscriptionType(SubType.EXCLUSIVE)
            .build();

    consumer.subscribe();

    CountDownLatch shutdown = new CountDownLatch(1);

    // Start receiving via Flow.Publisher
    consumer.receive().subscribe(new Flow.Subscriber<>() {
        @Override
        public void onSubscribe(Flow.Subscription s) {
            s.request(Long.MAX_VALUE);
        }

        @Override
        public void onNext(StreamMessage msg) {
            String payload = new String(msg.payload());
            System.out.println("Received: " + payload);

            // Acknowledge the message
            consumer.ack(msg);
        }

        @Override public void onError(Throwable t) { shutdown.countDown(); }
        @Override public void onComplete() { shutdown.countDown(); }
    });

    shutdown.await();
    ```

---

## Message Handling Essentials

=== "Rust"

    ```rust
    while let Some(message) = message_stream.recv().await {
        let payload = String::from_utf8_lossy(&message.payload);
        println!("📥 {}", payload);

        if let Some(attributes) = &message.attributes {
            for (key, value) in attributes {
                println!("  {}: {}", key, value);
            }
        }

        // Only ack after successful processing
        consumer.ack(&message).await?;
    }
    ```

=== "Go"

    ```go
    for msg := range stream {
        payload := string(msg.GetPayload())
        fmt.Printf("📥 %s\n", payload)

        for key, value := range msg.GetAttributes() {
            fmt.Printf("  %s: %s\n", key, value)
        }

        if _, err := consumer.Ack(ctx, msg); err != nil {
            log.Printf("Failed to ack: %v", err)
        }
    }
    ```

=== "Python"

    ```python
    while True:
        message = await queue.get()
        payload = message.payload.decode()
        print(f"📥 {payload}")

        # Only ack after successful processing
        await consumer.ack(message)
    ```

=== "Java"

    ```java
    // Inside onNext(StreamMessage msg):
    String payload = new String(msg.payload());
    System.out.println("Received: " + payload);

    Map<String, String> attributes = msg.attributes();
    attributes.forEach((k, v) -> System.out.printf("  %s: %s%n", k, v));

    // Only ack after successful processing
    consumer.ack(msg);
    ```

**Tip:** Only ack after successful processing; unacked messages are redelivered.

---

## Negative Acknowledgment (NACK)

When a consumer cannot process a message (e.g., validation failure, transient dependency error), it can explicitly reject the message with `nack()`. The broker will schedule a redelivery based on the subscription's failure policy.

=== "Rust"

    ```rust
    while let Some(message) = message_stream.recv().await {
        match process(&message) {
            Ok(_) => consumer.ack(&message).await?,
            Err(e) => {
                // NACK with optional delay (ms) and reason
                consumer.nack(
                    &message,
                    Some(1000),                         // retry after 1s minimum
                    Some(format!("processing failed: {}", e)),
                ).await?;
            }
        }
    }
    ```

=== "Go"

    ```go
    for msg := range stream {
        if err := process(msg); err != nil {
            // NACK with optional delay and reason
            consumer.Nack(ctx, msg, 1000, fmt.Sprintf("processing failed: %v", err))
        } else {
            consumer.Ack(ctx, msg)
        }
    }
    ```

=== "Python"

    ```python
    while True:
        message = await queue.get()
        try:
            process(message)
            await consumer.ack(message)
        except Exception as e:
            # NACK with optional delay (ms) and reason
            await consumer.nack(message, delay_ms=1000, reason=f"failed: {e}")
    ```

=== "Java"

    ```java
    // Inside onNext(StreamMessage msg):
    try {
        process(msg);
        consumer.ack(msg);
    } catch (Exception e) {
        // NACK with optional delay (ms) and reason
        consumer.nack(msg, 1000, "processing failed: " + e.getMessage());
    }
    ```

**Parameters:**

- **`delay_ms`** (optional) — Minimum delay before redelivery, in milliseconds. The broker uses the larger of this value and the subscription's configured backoff delay.
- **`reason`** (optional) — Human-readable failure reason, attached to metrics and dead-letter metadata.

**What happens after NACK:**

The broker applies the subscription's failure policy to decide what to do next. If the message has remaining delivery attempts, it is redelivered after the backoff delay. If `max_redelivery_count` is exceeded, the message is handled according to the configured poison policy (`dead_letter`, `drop`, or `block`). See [Dispatch Strategy - Failure Handling](../concepts/dispatch_strategy.md#failure-handling-reliable-dispatch) for details.

---

## Consuming from Partitioned Topics

When a topic has partitions, consumers automatically receive from all partitions.

### Automatic Partition Handling

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SubType};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        // Topic has 3 partitions: my-topic-part-0, my-topic-part-1, my-topic-part-2
        let mut consumer = client
            .new_consumer()
            .with_topic("/default/my-topic")  // Parent topic name
            .with_consumer_name("partition-consumer")
            .with_subscription("partition-sub")
            .with_subscription_type(SubType::Exclusive)
            .build()?;

        consumer.subscribe().await?;
        println!("✅ Subscribed to all partitions");

        // Automatically receives from all 3 partitions
        let mut message_stream = consumer.receive().await?;

        while let Some(message) = message_stream.recv().await {
            println!("📥 Received from partition: {}", message.payload);
            consumer.ack(&message).await?;
        }

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "context"
        "fmt"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("Failed to create client: %v", err)
        }

        ctx := context.Background()

        consumer, err := client.NewConsumer().
            WithConsumerName("partition-consumer").
            WithTopic("/default/my-topic").  // Parent topic
            WithSubscription("partition-sub").
            WithSubscriptionType(danube.Exclusive).
            Build()
        if err != nil {
            log.Fatalf("Failed to create consumer: %v", err)
        }

        if err := consumer.Subscribe(ctx); err != nil {
            log.Fatalf("Failed to subscribe: %v", err)
        }
        fmt.Println("✅ Subscribed to all partitions")

        stream, err := consumer.Receive(ctx)
        if err != nil {
            log.Fatalf("Failed to receive: %v", err)
        }

        for msg := range stream {
            fmt.Printf("📥 Received from partition: %s\n", string(msg.GetPayload()))
            consumer.Ack(ctx, msg)
        }
    }
    ```

=== "Python"

    ```python
    # Topic has 3 partitions: my-topic-part-0, my-topic-part-1, my-topic-part-2
    consumer = (
        client.new_consumer()
        .with_topic("/default/my-topic")  # Parent topic name
        .with_consumer_name("partition-consumer")
        .with_subscription("partition-sub")
        .with_subscription_type(SubType.EXCLUSIVE)
        .build()
    )

    await consumer.subscribe()
    print("✅ Subscribed to all partitions")

    # Automatically receives from all 3 partitions
    queue = await consumer.receive()

    while True:
        message = await queue.get()
        print(f"📥 Received from partition: {message.payload.decode()}")
        await consumer.ack(message)
    ```

=== "Java"

    ```java
    // Topic has 3 partitions: my-topic-part-0, my-topic-part-1, my-topic-part-2
    Consumer consumer = client.newConsumer()
            .withTopic("/default/my-topic")  // Parent topic name
            .withConsumerName("partition-consumer")
            .withSubscription("partition-sub")
            .withSubscriptionType(SubType.EXCLUSIVE)
            .build();

    consumer.subscribe();
    System.out.println("Subscribed to all partitions");

    // Automatically receives from all partitions via Flow.Publisher
    consumer.receive().subscribe(new Flow.Subscriber<>() {
        @Override public void onSubscribe(Flow.Subscription s) { s.request(Long.MAX_VALUE); }

        @Override
        public void onNext(StreamMessage msg) {
            System.out.printf("Received from partition '%s': %s%n",
                    msg.messageId().topicName(), new String(msg.payload()));
            consumer.ack(msg);
        }

        @Override public void onError(Throwable t) {}
        @Override public void onComplete() {}
    });
    ```

**What happens:**

- Client discovers all partitions automatically
- Creates one consumer per partition internally
- Messages from all partitions merged into single stream
- Ordering preserved per-partition, not cross-partition

---

## Shared and Failover Subscriptions

Use **Shared** for load-balanced processing across multiple consumers. Use **Failover** for active/standby high availability with ordering.

---

## Schema Integration

Consume typed messages validated against schemas (see [Schema Registry](schema-registry.md) for details).

### Minimal Schema Consumption

=== "Rust"

    ```rust
    let mut stream = consumer.receive().await?;
    while let Some(message) = stream.recv().await {
        let event: Event = serde_json::from_slice(&message.payload)?;
        println!("📥 Event: {:?}", event);
        consumer.ack(&message).await?;
    }
    ```

=== "Go"

    ```go
    for msg := range stream {
        var event Event
        if err := json.Unmarshal(msg.GetPayload(), &event); err != nil {
            log.Printf("decode failed: %v", err)
            continue
        }

        fmt.Printf("📥 Event: %+v\n", event)
        consumer.Ack(ctx, msg)
    }
    ```

=== "Python"

    ```python
    import json

    while True:
        message = await queue.get()
        event = json.loads(message.payload)
        print(f"📥 Event: {event}")
        await consumer.ack(message)
    ```

=== "Java"

    ```java
    // Inside onNext(StreamMessage msg):
    try {
        // Deserialize JSON payload
        String json = new String(msg.payload());
        System.out.println("Received: " + json);

        // Only ack after successful processing
        consumer.ack(msg);
    } catch (Exception e) {
        System.err.println("Failed to process message: " + e.getMessage());
        // Do not ack — message will be redelivered
    }
    ```

For validation strategies and schema details, see [Schema Registry](schema-registry.md) and the full examples in the client repos.

---

## Full Examples

For complete runnable consumers, see the client repositories:

- Rust: https://github.com/danube-messaging/danube/tree/main/danube-client/examples
- Go: https://github.com/danube-messaging/danube-go/tree/main/examples
- Python: https://github.com/danube-messaging/danube-py/tree/main/examples
- Java: https://github.com/danube-messaging/danube-java/tree/main/examples
