# Consumer

A consumer is a process that attaches to a topic via a subscription and then receives messages.

**Subscription Types** - describe the way the consumers receive the messages from topics

* **Exclusive** -  Only one consumer can subscribe, guaranteeing message order.
* **Shared** -  Multiple consumers can subscribe, messages are delivered round-robin, offering good scalability but no order guarantee.
* **Failover** - Similar to shared subscriptions, but multiple consumers can subscribe, and one actively receives messages.

## Example

=== "Rust"

    ```rust
    let topic = "/default/test_topic".to_string();

    let mut consumer = client
        .new_consumer()
        .with_topic(topic.clone())
        .with_consumer_name("test_consumer")
        .with_subscription("test_subscription")
        .with_subscription_type(SubType::Exclusive)
        .build();

    // Subscribe to the topic
    let consumer_id = consumer.subscribe().await?;
    println!("The Consumer with ID: {:?} was created", consumer_id);

    let _schema = client.get_schema(topic).await.unwrap();

    // Start receiving messages
    let mut message_stream = consumer.receive().await?;

     while let Some(message) = message_stream.next().await {
        
        //process the message and ack for receive
     
     }
    ```

=== "Go"

    ```go
    ctx := context.Background()
    topic := "/default/test_topic"
    subType := danube.Exclusive

    consumer, err := client.NewConsumer(ctx).
        WithConsumerName("test_consumer").
        WithTopic(topic).
        WithSubscription("test_subscription").
        WithSubscriptionType(subType).
        Build()
    if err != nil {
        log.Fatalf("Failed to initialize the consumer: %v", err)
    }

    consumerID, err := consumer.Subscribe(ctx)
    if err != nil {
        log.Fatalf("Failed to subscribe: %v", err)
    }
    log.Printf("The Consumer with ID: %v was created", consumerID)

    // Receiving messages
    streamClient, err := consumer.Receive(ctx)
    if err != nil {
        log.Fatalf("Failed to receive messages: %v", err)
    }

    for {
        msg, err := streamClient.Recv()
       
       //process the message and ack for receive
    
    }
    ```

## Complete example

For a complete example implementation of the above code using producers and consumers, check the examples:

* [Rust Examples](https://github.com/danube-messaging/danube/tree/main/danube-client/examples)
* [Go Examples](https://github.com/danube-messaging/danube-go/tree/main/examples)
