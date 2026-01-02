# Advanced Producer Features

This guide covers advanced producer capabilities including partitions, reliable dispatch, and schema integration.

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
            .build();

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
        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()

        ctx := context.Background()

        producer, err := client.NewProducer(ctx).
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

**What happens:**

- Topic `/default/high-throughput` becomes:
  - `/default/high-throughput-part-0`
  - `/default/high-throughput-part-1`
  - `/default/high-throughput-part-2`
- Messages distributed using round-robin routing
- Each partition can be on different brokers

### Sending to Partitions

Messages are automatically routed:

=== "Rust"

    ```rust
    // Automatic round-robin distribution
    for i in 0..9 {
        let message = format!("Message {}", i);
        producer.send(message.as_bytes().to_vec(), None).await?;
    }

    // Result:
    // Message 0 -> partition 0
    // Message 1 -> partition 1
    // Message 2 -> partition 2
    // Message 3 -> partition 0 (round-robin)
    // ...
    ```

=== "Go"

    ```go
    import "fmt"

    // Automatic round-robin distribution
    for i := 0; i < 9; i++ {
        message := fmt.Sprintf("Message %d", i)
        payload := []byte(message)
        producer.Send(ctx, payload, nil)
    }
    ```

### When to Use Partitions

✅ **Use partitions when:**

- High message throughput (>1K msgs/sec)
- Messages can be processed in any order
- Horizontal scaling needed
- Multiple brokers available

❌ **Avoid partitions when:**

- Strict message ordering required
- Low message volume
- Single broker deployment

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
            .build();

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
        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()

        ctx := context.Background()

        // Configure reliable dispatch strategy
        reliableStrategy := danube.NewReliableDispatchStrategy()

        producer, err := client.NewProducer(ctx).
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

### How Reliable Dispatch Works

```bash
1. Producer sends message
2. Broker persists to WAL (Write-Ahead Log)
3. Broker uploads to cloud storage (background)
4. Broker acknowledges send
5. Message delivered to consumers from WAL/cloud
```

**Guarantees:**

- ✅ Message never lost (persisted to disk + cloud)
- ✅ Survives broker restarts
- ✅ Replay from historical offset
- ✅ Consumer can restart and resume

**Trade-offs:**

- Slightly higher latency (~5-10ms added)
- Storage costs for persistence
- Good for: Critical business events, audit logs, transactions

### When to Use Reliable Dispatch

✅ **Use for:**

- Financial transactions
- Order processing
- Audit logs
- Critical notifications

❌ **Avoid for:**

- Real-time metrics (can tolerate loss)
- High-frequency sensor data
- Live streaming (freshness > durability)

---

## Combining Features

### Partitions + Reliable Dispatch

Scale and durability together:

=== "Rust"

    ```rust
    let mut producer = client
        .new_producer()
        .with_topic("/default/orders")
        .with_name("order-producer")
        .with_partitions(5)           // Scale across 5 partitions
        .with_reliable_dispatch()     // Persist all messages
        .build();

    producer.create().await?;

    // High throughput + guaranteed delivery
    for order_id in 1..=10000 {
        let order = format!("{{\"order_id\": {}}}", order_id);
        producer.send(order.as_bytes().to_vec(), None).await?;
    }
    ```

---

## Schema Integration

Link producers to schemas for type safety (see [Schema Registry](schema-registry.md) for details).

**Note:** Schema Registry integration is not yet available in the Go client.

### Basic Schema Usage

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SchemaRegistryClient, SchemaType};
    use serde::Serialize;

    #[derive(Serialize)]
    struct Event {
        event_id: String,
        timestamp: i64,
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        // 1. Register schema
        let json_schema = r#"{
            "type": "object",
            "properties": {
                "event_id": {"type": "string"},
                "timestamp": {"type": "integer"}
            },
            "required": ["event_id", "timestamp"]
        }"#;

        let mut schema_client = SchemaRegistryClient::new(&client).await?;
        let schema_id = schema_client
            .register_schema("event-schema")
            .with_type(SchemaType::JsonSchema)
            .with_schema_data(json_schema.as_bytes())
            .execute()
            .await?;

        println!("📋 Registered schema ID: {}", schema_id);

        // 2. Create producer with schema reference
        let mut producer = client
            .new_producer()
            .with_topic("/default/events")
            .with_name("schema-producer")
            .with_schema_subject("event-schema")  // Link to schema
            .build();

        producer.create().await?;

        // 3. Send typed messages
        let event = Event {
            event_id: "evt-123".to_string(),
            timestamp: 1234567890,
        };

        let json_bytes = serde_json::to_vec(&event)?;
        let msg_id = producer.send(json_bytes, None).await?;

        println!("📤 Sent validated message: {}", msg_id);

        Ok(())
    }
    ```

**Benefits:**

- Messages validated against schema before sending
- Schema ID included in message metadata (8 bytes vs KB of schema)
- Consumers know message structure
- Safe schema evolution with compatibility checking
