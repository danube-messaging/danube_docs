# Consumer Basics

Consumers receive messages from Danube topics via subscriptions. This guide covers fundamental consumer operations.

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

**Key concepts:**

- **Topic:** Source of messages
- **Consumer Name:** Unique identifier for this consumer instance
- **Subscription:** Logical grouping of consumers (multiple consumers can share)
- **Subscription Type:** Controls how messages are distributed (Exclusive, Shared, Failover)

---

## Subscription Types

### Exclusive

Only **one consumer** can be active. Guarantees message order.

=== "Rust"

    ```rust
    use danube_client::SubType;

    let mut consumer = client
        .new_consumer()
        .with_topic("/default/orders")
        .with_consumer_name("order-processor")
        .with_subscription("order-sub")
        .with_subscription_type(SubType::Exclusive)  // Only this consumer
        .build()?;
    ```

=== "Go"

    ```go
    consumer, err := client.NewConsumer().
        WithConsumerName("order-processor").
        WithTopic("/default/orders").
        WithSubscription("order-sub").
        WithSubscriptionType(danube.Exclusive).
        Build()
    if err != nil {
        log.Fatalf("Failed to create consumer: %v", err)
    }
    ```

**Characteristics:**

- ✅ Message ordering guaranteed
- ✅ Simple failure handling
- ❌ No horizontal scaling
- **Use case:** Order processing, sequential workflows

### Shared

**Multiple consumers** receive messages in round-robin. Scales horizontally.

=== "Rust"

    ```rust
    use danube_client::SubType;

    let mut consumer = client
        .new_consumer()
        .with_topic("/default/logs")
        .with_consumer_name("log-processor-1")
        .with_subscription("log-sub")
        .with_subscription_type(SubType::Shared)  // Multiple consumers allowed
        .build()?;
    ```

=== "Go"

    ```go
    consumer, err := client.NewConsumer().
        WithConsumerName("log-processor-1").
        WithTopic("/default/logs").
        WithSubscription("log-sub").
        WithSubscriptionType(danube.Shared).
        Build()
    if err != nil {
        log.Fatalf("Failed to create consumer: %v", err)
    }
    ```

**Characteristics:**

- ✅ Horizontal scaling
- ✅ Load distribution
- ❌ No ordering guarantee
- **Use case:** Log processing, analytics, parallel processing

### Failover

Like Exclusive, but allows **standby consumers**. One active, others wait.

=== "Rust"

    ```rust
    use danube_client::SubType;

    let mut consumer = client
        .new_consumer()
        .with_topic("/default/critical")
        .with_consumer_name("processor-1")
        .with_subscription("critical-sub")
        .with_subscription_type(SubType::FailOver)  // Hot standby
        .build()?;
    ```

=== "Go"

    ```go
    consumer, err := client.NewConsumer().
        WithConsumerName("processor-1").
        WithTopic("/default/critical").
        WithSubscription("critical-sub").
        WithSubscriptionType(danube.FailOver).
        Build()
    if err != nil {
        log.Fatalf("Failed to create consumer: %v", err)
    }
    ```

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

---

## Message Acknowledgment

Acknowledgment tells the broker the message was processed successfully.

### Ack Pattern

=== "Rust"

    ```rust
    while let Some(message) = message_stream.recv().await {
        // Process message
        match process_message(&message.payload).await {
            Ok(_) => {
                // Acknowledge success
                consumer.ack(&message).await?;
                println!("✅ Processed and acked");
            }
            Err(e) => {
                // Don't ack on failure - message will be redelivered
                eprintln!("❌ Processing failed: {}", e);
                // Message will be redelivered to this or another consumer
            }
        }
    }
    ```

=== "Go"

    ```go
    for msg := range stream {
        if err := processMessage(msg.GetPayload()); err != nil {
            // Don't ack on failure - message will be redelivered
            log.Printf("❌ Processing failed: %v", err)
            continue
        }

        // Acknowledge success
        if _, err := consumer.Ack(ctx, msg); err != nil {
            log.Printf("Failed to ack: %v", err)
        }

        fmt.Println("✅ Processed and acked")
    }
    ```

**Important:**

- ⚠️ Only ack after successful processing
- ⚠️ Unacked messages are redelivered
- ⚠️ Messages persist until acked or subscription expires

---

## Reading Message Attributes

Access metadata sent with messages:

=== "Rust"

    ```rust
    while let Some(message) = message_stream.recv().await {
        // Read payload
        let payload = String::from_utf8_lossy(&message.payload);
        
        // Read attributes (if any)
        if let Some(attributes) = &message.attributes {
            for (key, value) in attributes {
                println!("  {}: {}", key, value);
            }
        }

        // Read other metadata
        println!("Message ID: {:?}", message.msg_id);
        println!("Publish time: {}", message.publish_time);

        consumer.ack(&message).await?;
    }
    ```

=== "Go"

    ```go
    for msg := range stream {
        payload := string(msg.GetPayload())
        
        // Read attributes
        for key, value := range msg.GetAttributes() {
            fmt.Printf("  %s: %s\n", key, value)
        }

        // Read metadata
        fmt.Printf("Message ID: %v\n", msg.GetMessageId())
        fmt.Printf("Publish time: %d\n", msg.GetPublishTime())

        consumer.Ack(ctx, msg)
    }
    ```

---

## Complete Example

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SubType};
    use tokio::time::{sleep, Duration};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        // 1. Setup client
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        // 2. Create consumer
        let mut consumer = client
            .new_consumer()
            .with_topic("/default/events")
            .with_consumer_name("event-processor")
            .with_subscription("event-sub")
            .with_subscription_type(SubType::Exclusive)
            .build()?;

        consumer.subscribe().await?;
        println!("✅ Consumer subscribed and ready");

        // 3. Receive messages
        let mut message_stream = consumer.receive().await?;
        let mut count = 0;

        while let Some(message) = message_stream.recv().await {
            // Extract payload
            let payload = String::from_utf8_lossy(&message.payload);
            
            // Log receipt
            count += 1;
            println!("📥 Message #{}: {}", count, payload);

            // Check attributes
            if let Some(attrs) = &message.attributes {
                if let Some(priority) = attrs.get("priority") {
                    if priority == "high" {
                        println!("  ⚡ High priority message!");
                    }
                }
            }

            // Simulate processing
            sleep(Duration::from_millis(100)).await;

            // Acknowledge
            match consumer.ack(&message).await {
                Ok(_) => println!("  ✅ Acknowledged"),
                Err(e) => eprintln!("  ❌ Ack failed: {}", e),
            }
        }

        Ok(())
    }
    ```

=== "Go"

    ```go
    package main

    import (
        "context"
        "fmt"
        "log"
        "time"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        // 1. Setup client
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("Failed to create client: %v", err)
        }

        ctx := context.Background()

        // 2. Create consumer
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

        fmt.Println("✅ Consumer subscribed and ready")

        // 3. Receive messages
        stream, err := consumer.Receive(ctx)
        if err != nil {
            log.Fatalf("Failed to receive: %v", err)
        }

        count := 0
        for msg := range stream {
            payload := string(msg.GetPayload())
            
            count++
            fmt.Printf("📥 Message #%d: %s\n", count, payload)

            // Check attributes
            if priority, ok := msg.GetAttributes()["priority"]; ok {
                if priority == "high" {
                    fmt.Println("  ⚡ High priority message!")
                }
            }

            // Simulate processing
            time.Sleep(100 * time.Millisecond)

            // Acknowledge
            if _, err := consumer.Ack(ctx, msg); err != nil {
                fmt.Printf("  ❌ Ack failed: %v\n", err)
            } else {
                fmt.Println("  ✅ Acknowledged")
            }
        }
    }
    ```
