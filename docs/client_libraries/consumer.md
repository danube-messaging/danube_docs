# Consumer

A consumer is a process that attaches to a topic via a subscription and then receives messages.

**Subscription Types** - describe the way the consumers receive the messages from topics

* **Exclusive** -  Only one consumer can subscribe, guaranteeing message order.
* **Shared** -  Multiple consumers can subscribe, messages are delivered round-robin, offering good scalability but no order guarantee.
* **Failover** - Similar to shared subscriptions, but multiple consumers can subscribe, and one actively receives messages.

## Example

=== "Rust"

    ```rust
    let topic = "/default/test_topic";
    
      // Create the Exclusive consumer
    let mut consumer = danube_client
        .new_consumer()
        .with_topic(topic.to_string())
        .with_consumer_name(consumer_name.to_string())
        .with_subscription(format!("test_subscription_{}", consumer_name))
        .with_subscription_type(SubType::Exclusive)
        .build();

    // Subscribe to the topic
    consumer.subscribe().await?;

    // Start receiving messages
    let mut message_stream = consumer.receive().await?;

     if let Some(stream_message) = message_stream.recv().await {
        
        //process the message and ack for receive
        consumer.ack(&stream_message).await?
     
     }
    ```

=== "Go"

    ```go
    ctx := context.Background()
    topic := "/default/topic_test"
    consumerName := "consumer_test"
    subscriptionName := "subscription_test"
    subType := danube.Exclusive

    consumer, err := client.NewConsumer(ctx).
        WithConsumerName(consumerName).
        WithTopic(topic).
        WithSubscription(subscriptionName).
        WithSubscriptionType(subType).
        Build()
    if err != nil {
        log.Fatalf("Failed to initialize the consumer: %v", err)
    }

    if err := consumer.Subscribe(ctx); err != nil {
        log.Fatalf("Failed to subscribe: %v", err)
    }
    log.Printf("The Consumer %s was created", consumerName)


 
    // Receiving messages
    stream, err := consumer.Receive(ctx)
    if err != nil {
        log.Fatalf("Failed to receive messages: %v", err)
    }

    for msg := range stream {
        fmt.Printf("Received message: %+v\n", string(msg.GetPayload()))

        if _, err := consumer.Ack(ctx, msg); err != nil {
            log.Fatalf("Failed to acknowledge message: %v", err)
        }
    }
    ```

## Complete example

For complete code examples of using producers and consumers, check the links:

* [Rust Examples](https://github.com/danube-messaging/danube/tree/main/danube-client/examples)
* [Go Examples](https://github.com/danube-messaging/danube-go/tree/main/examples)
