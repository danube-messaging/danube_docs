# Producer Basics

Producers send messages to Danube topics. This guide covers fundamental producer operations.

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
        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()

        ctx := context.Background()

        producer, err := client.NewProducer(ctx).
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

**Use cases:**

- Routing hints
- Message metadata
- Custom headers
- Tracing IDs

---

## Complete Example

=== "Rust"

    ```rust
    use danube_client::DanubeClient;
    use std::collections::HashMap;
    use tokio::time::{sleep, Duration};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        // 1. Setup client
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        // 2. Create producer
        let mut producer = client
            .new_producer()
            .with_topic("/default/events")
            .with_name("event-producer")
            .build()?;

        producer.create().await?;
        println!("✅ Producer created");

        // 3. Send messages
        for i in 1..=10 {
            // Prepare message
            let message = format!("Event #{}", i);
            
            // Add attributes
            let mut attributes = HashMap::new();
            attributes.insert("event_id".to_string(), i.to_string());
            attributes.insert("timestamp".to_string(), 
                chrono::Utc::now().to_rfc3339());

            // Send
            match producer.send(message.as_bytes().to_vec(), Some(attributes)).await {
                Ok(msg_id) => println!("📤 Sent: {} (ID: {})", message, msg_id),
                Err(e) => eprintln!("❌ Failed to send: {}", e),
            }

            sleep(Duration::from_millis(500)).await;
        }

        println!("✅ Sent 10 messages");
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
        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()

        ctx := context.Background()

        // 2. Create producer
        producer, err := client.NewProducer(ctx).
            WithName("event-producer").
            WithTopic("/default/events").
            Build()
        if err != nil {
            log.Fatalf("Failed to create producer: %v", err)
        }

        if err := producer.Create(ctx); err != nil {
            log.Fatalf("Failed to initialize producer: %v", err)
        }

        fmt.Println("✅ Producer created")

        // 3. Send messages
        for i := 1; i <= 10; i++ {
            message := fmt.Sprintf("Event #%d", i)
            
            attributes := map[string]string{
                "event_id":  fmt.Sprintf("%d", i),
                "timestamp": time.Now().Format(time.RFC3339),
            }

            msgID, err := producer.Send(ctx, []byte(message), attributes)
            if err != nil {
                fmt.Printf("❌ Failed to send: %v\n", err)
                continue
            }

            fmt.Printf("📤 Sent: %s (ID: %v)\n", message, msgID)
            time.Sleep(500 * time.Millisecond)
        }

        fmt.Println("✅ Sent 10 messages")
    }
    ```

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

### Best Practices

✅ **Do:**

- Use descriptive names: `/default/payment-processed`
- Group by domain: `/inventory/stock-updates`
- Include environment in namespace: `/prod/...`, `/dev/...`

❌ **Don't:**

- Use special characters except `-` and `_`
- Make names too long (keep under 255 chars)
- Mix environments in same namespace
