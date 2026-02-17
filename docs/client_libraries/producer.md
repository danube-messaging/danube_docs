# Producer Guide

Producers send messages to Danube topics. This guide covers both basic and advanced producer capabilities.

## Creating a Producer

### Simple Producer

The minimal setup to send messages:

=== "Rust"

    ```rust
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut producer = client
            .new_producer()
            .with_topic("/default/my-topic")
            .with_name("my-producer")
            .build()?;

        producer.create().await?;
        println!("✅ Producer created");

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

        producer, err := client.NewProducer().
            WithName("my-producer").
            WithTopic("/default/my-topic").
            Build()
        if err != nil {
            log.Fatalf("Failed to create producer: %v", err)
        }

        if err := producer.Create(ctx); err != nil {
            log.Fatalf("Failed to initialize producer: %v", err)
        }

        fmt.Println("✅ Producer created")
    }
    ```

=== "Python"

    ```python
    import asyncio
    from danube import DanubeClientBuilder

    async def main():
        client = await (
            DanubeClientBuilder()
            .service_url("http://127.0.0.1:6650")
            .build()
        )

        producer = (
            client.new_producer()
            .with_topic("/default/my-topic")
            .with_name("my-producer")
            .build()
        )

        await producer.create()
        print("✅ Producer created")

    asyncio.run(main())
    ```

**Key concepts:**

- **Topic:** Destination for messages (e.g., `/default/my-topic`)
- **Name:** Unique producer identifier
- **Create:** Registers producer with broker before sending

---

## Sending Messages

### Byte Messages

Send raw byte data:

=== "Rust"

    ```rust
    let message = "Hello Danube!";
    let message_id = producer
        .send(message.as_bytes().to_vec(), None)
        .await?;

    println!("📤 Sent message ID: {}", message_id);
    ```

=== "Go"

    ```go
    message := "Hello Danube!"
    messageID, err := producer.Send(ctx, []byte(message), nil)
    if err != nil {
        log.Fatalf("Failed to send: %v", err)
    }

    fmt.Printf("📤 Sent message ID: %v\n", messageID)
    ```

=== "Python"

    ```python
    message = "Hello Danube!"
    message_id = await producer.send(message.encode())

    print(f"📤 Sent message ID: {message_id}")
    ```

**Returns:** Unique message ID for tracking

### Messages with Attributes

Add metadata to messages:

=== "Rust"

    ```rust
    use std::collections::HashMap;

    let mut attributes = HashMap::new();
    attributes.insert("source".to_string(), "app-1".to_string());
    attributes.insert("priority".to_string(), "high".to_string());

    let message_id = producer
        .send(b"Important message".to_vec(), Some(attributes))
        .await?;
    ```

=== "Go"

    ```go
    attributes := map[string]string{
        "source":   "app-1",
        "priority": "high",
    }

    messageID, err := producer.Send(ctx, []byte("Important message"), attributes)
    ```

=== "Python"

    ```python
    attributes = {
        "source": "app-1",
        "priority": "high",
    }

    message_id = await producer.send(b"Important message", attributes)
    ```

**Use cases:**

- Routing hints
- Message metadata
- Custom headers
- Tracing IDs

---

## Partitioned Topics

Partitions enable horizontal scaling by distributing messages across multiple brokers.

### Creating Partitioned Producers

=== "Rust"

    ```rust
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut producer = client
            .new_producer()
            .with_topic("/default/high-throughput")
            .with_name("partitioned-producer")
            .with_partitions(3)  // Create 3 partitions
            .build()?;

        producer.create().await?;

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "context"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("Failed to create client: %v", err)
        }

        ctx := context.Background()

        producer, err := client.NewProducer().
            WithName("partitioned-producer").
            WithTopic("/default/high-throughput").
            WithPartitions(3).  // Create 3 partitions
            Build()
        if err != nil {
            log.Fatalf("Failed to create producer: %v", err)
        }

        if err := producer.Create(ctx); err != nil {
            log.Fatalf("Failed to initialize producer: %v", err)
        }
    }
    ```

=== "Python"

    ```python
    producer = (
        client.new_producer()
        .with_topic("/default/high-throughput")
        .with_name("partitioned-producer")
        .with_partitions(3)  # Create 3 partitions
        .build()
    )

    await producer.create()
    ```

**What happens:**

- Topic splits into N partitions (e.g., `-part-0`, `-part-1`, `-part-2`).
- Producer routes messages round-robin across partitions.

---

## Reliable Dispatch

Reliable dispatch guarantees message delivery by persisting messages before acknowledging sends.

### Enabling Reliable Dispatch

=== "Rust"

    ```rust
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut producer = client
            .new_producer()
            .with_topic("/default/critical-events")
            .with_name("reliable-producer")
            .with_reliable_dispatch()  // Enable persistence
            .build()?;

        producer.create().await?;

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "context"
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("Failed to create client: %v", err)
        }

        ctx := context.Background()

        // Configure reliable dispatch strategy
        reliableStrategy := danube.NewReliableDispatchStrategy()

        producer, err := client.NewProducer().
            WithName("reliable-producer").
            WithTopic("/default/critical-events").
            WithDispatchStrategy(reliableStrategy).
            Build()
        if err != nil {
            log.Fatalf("Failed to create producer: %v", err)
        }

        if err := producer.Create(ctx); err != nil {
            log.Fatalf("Failed to initialize producer: %v", err)
        }
    }
    ```

=== "Python"

    ```python
    from danube import DispatchStrategy

    producer = (
        client.new_producer()
        .with_topic("/default/critical-events")
        .with_name("reliable-producer")
        .with_dispatch_strategy(DispatchStrategy.RELIABLE)  # Enable persistence
        .build()
    )

    await producer.create()
    ```

**Notes:** Reliable dispatch persists messages before acking. Use it for critical events that must not be lost.

---

## Schema Integration

Link producers to schemas for type safety (see [Schema Registry](schema-registry.md) for details).

### Minimal Schema Usage

=== "Rust"

    ```rust
    let mut producer = client
        .new_producer()
        .with_topic("/default/events")
        .with_name("schema-producer")
        .with_schema_subject("event-schema")
        .build()?;

    producer.create().await?;
    ```

=== "Go"

    ```go
    producer, err := client.NewProducer().
        WithTopic("/default/events").
        WithName("schema-producer").
        WithSchemaSubject("event-schema").
        Build()
    if err != nil {
        log.Fatalf("Failed to build producer: %v", err)
    }

    if err := producer.Create(ctx); err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }
    ```

=== "Python"

    ```python
    producer = (
        client.new_producer()
        .with_topic("/default/events")
        .with_name("schema-producer")
        .with_schema_subject("event-schema")
        .build()
    )

    await producer.create()
    ```

For schema registration and versioning, see [Schema Registry](schema-registry.md).

---

## Full Examples

For complete runnable producers, see the client repositories:

- Rust: https://github.com/danube-messaging/danube/tree/main/danube-client/examples
- Go: https://github.com/danube-messaging/danube-go/tree/main/examples
- Python: https://github.com/danube-messaging/danube-py/tree/main/examples

---

## Topic Naming

### Topic Format

Topics follow a namespace structure:

```bash
/namespace/topic-name
```

**Examples:**

- `/default/orders` - Orders in default namespace
- `/production/user-events` - User events in production namespace
- `/staging/logs` - Logs in staging namespace
