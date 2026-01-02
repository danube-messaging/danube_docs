# Advanced Consumer Features

This guide covers advanced consumer capabilities including partitioned topics, multiple consumers, and integration with schemas.

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
            .build();

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
        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()

        ctx := context.Background()

        consumer, err := client.NewConsumer(ctx).
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

**What happens:**

- Client discovers all partitions automatically
- Creates one consumer per partition internally
- Messages from all partitions merged into single stream
- Ordering preserved per-partition, not cross-partition

---

## Shared Subscription Patterns

### Load Balancing Across Consumers

With **Shared** subscription type, multiple consumers can subscribe to the same topic with the same subscription name. Messages are distributed round-robin across all active consumers.

**How it works:**

1. Deploy multiple consumer instances with:
   - Same topic: `/default/work-queue`
   - Same subscription: `"work-sub"`
   - Subscription type: `SubType::Shared`
   - Different consumer names: `"worker-1"`, `"worker-2"`, etc.

2. Broker distributes messages:
   - Message 1 → Worker 1
   - Message 2 → Worker 2  
   - Message 3 → Worker 3
   - Message 4 → Worker 1 (round-robin continues)

**Benefits:**

- ✅ Horizontal scaling - add more consumers to increase throughput
- ✅ Load distribution - work shared across all consumers
- ✅ Dynamic scaling - add/remove workers without coordination
- ✅ High throughput - parallel processing

**Trade-offs:**

- ❌ No ordering guarantee - messages may be processed out of order
- ❌ No affinity - same entity may go to different consumers

**Use cases:**

- Log processing pipelines
- Analytics workloads
- Image processing queues
- Any workload where order doesn't matter

---

## Failover Pattern

### High Availability Setup

With **Failover** subscription type, multiple consumers can subscribe to the same topic, but only one is active at a time. The others remain in standby. If the active consumer fails, the broker automatically promotes a standby consumer.

**How it works:**

1. Deploy multiple consumer instances with:
   - Same topic: `/default/critical-orders`
   - Same subscription: `"order-processor"`
   - Subscription type: `SubType::FailOver`
   - Different consumer names: `"processor-1"`, `"processor-2"`, etc.

2. Broker manages active consumer:
   - First connected consumer becomes **active** (receives messages)
   - Other consumers remain in **standby** (no messages received)
   - If active disconnects/fails, broker promotes next standby instantly
   - New active continues from last acknowledged message

**Behavior:**

- ✅ Only one consumer active at a time
- ✅ Automatic failover - no manual intervention
- ✅ Message ordering preserved (single active consumer)
- ✅ High availability - standbys ready to take over
- ✅ Zero message loss - standby resumes from last ack

**Use cases:**

- Critical order processing (needs ordering + HA)
- Financial transactions
- State machine workflows
- Any workload requiring both ordering and high availability

---

## Schema Integration

Consume typed messages validated against schemas (see [Schema Registry](schema-registry.md) for details).

**Note:** Schema Registry integration is not yet available in the Go client.

### Basic Schema Consumption

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SubType};
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
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

        let mut consumer = client
            .new_consumer()
            .with_topic("/default/events")
            .with_consumer_name("event-consumer")
            .with_subscription("event-sub")
            .with_subscription_type(SubType::Exclusive)
            .build();

        consumer.subscribe().await?;

        let mut stream = consumer.receive().await?;

        while let Some(message) = stream.recv().await {
            // Deserialize JSON message
            match serde_json::from_slice::<Event>(&message.payload) {
                Ok(event) => {
                    println!("📥 Event: {:?}", event);
                    consumer.ack(&message).await?;
                }
                Err(e) => {
                    eprintln!("❌ Deserialization failed: {}", e);
                    // Don't ack invalid messages
                }
            }
        }

        Ok(())
    }
    ```

### Validated Schema Consumption

Validate your Rust struct against the registered schema **at startup** to catch incompatibilities before processing messages (Rust only):

=== "Rust"

    ```rust
    use danube_client::{DanubeClient, SchemaRegistryClient, SubType};
    use serde::{Deserialize, Serialize};
    use jsonschema::JSONSchema;

    #[derive(Deserialize, Serialize, Debug)]
    struct MyMessage {
        field1: String,
        field2: i32,
    }

    /// Validates that consumer struct matches the schema in the registry
    async fn validate_struct_against_registry<T: Serialize>(
        schema_client: &mut SchemaRegistryClient,
        subject: &str,
        sample: &T,
    ) -> Result<u32, Box<dyn std::error::Error>> {
        println!("🔍 Fetching schema from registry: {}", subject);

        let schema_response = schema_client.get_latest_schema(subject).await?;
        println!("📋 Schema version: {}", schema_response.version);

        // Parse schema definition
        let schema_def: serde_json::Value = 
            serde_json::from_slice(&schema_response.schema_definition)?;

        // Compile JSON Schema validator
        let validator = JSONSchema::compile(&schema_def)
            .map_err(|e| format!("Invalid schema: {}", e))?;

        // Validate sample struct
        let sample_json = serde_json::to_value(sample)?;
        
        if let Err(errors) = validator.validate(&sample_json) {
            eprintln!("❌ VALIDATION FAILED: Struct incompatible with schema v{}", 
                schema_response.version);
            for error in errors {
                eprintln!("   - {}", error);
            }
            return Err("Struct validation failed".into());
        }

        println!("✅ Struct validated against schema v{}", schema_response.version);
        Ok(schema_response.version)
    }

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        let mut schema_client = SchemaRegistryClient::new(&client).await?;

        // VALIDATE BEFORE CONSUMING - fails fast if struct is wrong!
        let schema_version = validate_struct_against_registry(
            &mut schema_client,
            "my-app-events",
            &MyMessage {
                field1: "test".to_string(),
                field2: 0,
            },
        ).await?;

        println!("✅ Consumer validated - safe to deserialize\n");

        // Now create consumer
        let mut consumer = client
            .new_consumer()
            .with_topic("/default/test_topic")
            .with_consumer_name("validated-consumer")
            .with_subscription("validated-sub")
            .with_subscription_type(SubType::Exclusive)
            .build();

        consumer.subscribe().await?;
        let mut stream = consumer.receive().await?;

        while let Some(message) = stream.recv().await {
            match serde_json::from_slice::<MyMessage>(&message.payload) {
                Ok(msg) => {
                    println!("📥 Message: {:?}", msg);
                    consumer.ack(&message).await?;
                }
                Err(e) => {
                    eprintln!("❌ Deserialization failed: {}", e);
                    eprintln!("   Schema drift detected - check version {}", schema_version);
                    // Don't ack - message will retry or go to DLQ
                }
            }
        }

        Ok(())
    }
    ```

**Why validate at startup?**

- ✅ Fail fast - catch schema mismatches before processing messages
- ✅ Clear errors - know exactly which fields don't match
- ✅ Prevent runtime failures - no surprises during message processing
- ✅ Safe deployments - validates before going live

**Note:** Requires `jsonschema` crate dependency.

---

## Performance Tuning

### Batch Processing

Process messages in batches for efficiency:

=== "Rust"

    ```rust
    use tokio::time::{timeout, Duration};

    let mut stream = consumer.receive().await?;
    let mut batch = Vec::new();
    let batch_size = 100;
    let batch_timeout = Duration::from_millis(500);

    loop {
        match timeout(batch_timeout, stream.recv()).await {
            Ok(Some(message)) => {
                batch.push(message);

                if batch.len() >= batch_size {
                    // Process batch
                    process_batch(&batch).await?;

                    // Ack all
                    for msg in &batch {
                        consumer.ack(msg).await?;
                    }

                    batch.clear();
                }
            }
            Ok(None) => break,  // Stream closed
            Err(_) => {
                // Timeout - process partial batch
                if !batch.is_empty() {
                    process_batch(&batch).await?;
                    for msg in &batch {
                        consumer.ack(msg).await?;
                    }
                    batch.clear();
                }
            }
        }
    }
    ```
