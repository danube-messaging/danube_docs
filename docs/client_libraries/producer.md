# Producer

A producer is a process that attaches to a topic and publishes messages to a Danube broker. The Danube broker processes the messages.

**Access Mode** is a mechanism to determin the permissions of producers on topics.

* **Shared** - Multiple producers can publish on a topic.
* **Exclusive** - If there is already a producer connected, other producers trying to publish on this topic get errors immediately.

Before an application creates a producer/consumer, the  client library needs to initiate a setup phase including two steps:

* The client attempts to determine the owner of the topic by sending a Lookup request to Broker.  
* Once the client library has the broker address, it creates a RPC connection (or reuses an existing connection from the pool) and (in later stage authenticates it ).
* Within this connection, the clients (producer, consumer) and brokers exchange RPC commands. At this point, the client sends a command to create producer/consumer to the broker, which will comply after doing some validation checks.

## Create Producer

=== "Rust"

    ```rust
    let topic = "/default/topic_test";
    let producer_name = "producer_test";

    // Create the producer
    let mut producer = danube_client
        .new_producer()
        .with_topic(topic)
        .with_name(producer_name)
        .with_schema("my_schema".into(), SchemaType::String)
        .build();
    
    producer.create().await?;
    
    producer
        .send("Hello Danube".as_bytes().into(), None)
        .await?;
    ```

=== "Go"

    ```go
    topic := "/default/topic_test"
    producerName := "producer_test"

    producer, err := client.NewProducer(ctx).
        WithName(producerName).
        WithTopic(topic).
        WithSchema("test_schema", danube.SchemaType_STRING).
        Build()
    if err != nil {
        log.Fatalf("unable to initialize the producer: %v", err)
    }

    if err := producer.Create(ctx); err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }

    payload := fmt.Sprintf("Hello Danube %d", i)
    // Convert string to bytes
    bytes_payload := []byte(payload)

    messageID, err := producer.Send(ctx, bytes_payload , nil)
    if err != nil {
        log.Fatalf("Failed to send message: %v", err)
    }
    ```

## Create Producer with partitioned topic

Here we create a producer with a partitioned topic. A partitioned topic is implemented as N internal topics, where N is the number of partitions. When publishing messages to a partitioned topic, each message is routed to one of several brokers. The distribution of partitions across brokers is handled automatically.

=== "Rust"

    ```rust
    let topic = "/default/topic_test";
    let producer_name = "producer_test";
    let partitions = 3

    // Create the producer
    let mut producer = danube_client
        .new_producer()
        .with_topic(topic)
        .with_name(producer_name)
        .with_schema("my_schema".into(), SchemaType::String)
        .with_partitions(partitions)
        .build();
    
    producer.create().await?;

    producer
        .send("Hello Danube".as_bytes().into(), None)
        .await?;
    ```

=== "Go"

    ```go
    topic := "/default/topic_test"
    producerName := "producer_test"

    producer, err := client.NewProducer(ctx).
        WithName(producerName).
        WithTopic(topic).
        WithSchema("test_schema", danube.SchemaType_STRING).
        WithPartitions(3).
        Build()
    if err != nil {
        log.Fatalf("unable to initialize the producer: %v", err)
    }

    if err := producer.Create(ctx); err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }

    payload := fmt.Sprintf("Hello Danube %d", i)
    // Convert string to bytes
    bytes_payload := []byte(payload)

    messageID, err := producer.Send(ctx, bytes_payload , nil)
    if err != nil {
        log.Fatalf("Failed to send message: %v", err)
    }
    ```

## Create Producer with reliable dispatch topic

Here we create Producer with reliable dispatch topic. This strategy ensures guaranteed message delivery by implementing a store-and-forward mechanism. When a message arrives, it's first stored in the chosen storage backend before being dispatched to subscribers.

=== "Rust"

    ```rust
    use danube_client::{
    ConfigReliableOptions, ConfigRetentionPolicy, DanubeClient, SchemaType };

    let topic = "/default/topic_test";
    let producer_name = "producer_test";

    let reliable_options =
        ConfigReliableOptions::new(5, ConfigRetentionPolicy::RetainUntilExpire, 3600);

    // Create the producer
    let mut producer = danube_client
        .new_producer()
        .with_topic(topic)
        .with_name(producer_name)
        .with_schema("my_schema".into(), SchemaType::String)
        .with_reliable_dispatch(reliable_options)
        .build();
    
    producer.create().await?;
    
    producer
        .send("Hello Danube".as_bytes().into(), None)
        .await?;
    ```

=== "Go"

    ```go
    topic := "/default/topic_test"
    producerName := "producer_test"

    // For reliable strategy
    reliableOpts := danube.NewReliableOptions(
        10, // 10MB segment size
        danube.RetainUntilExpire,
        3600, // retention period in seconds
        )
    reliableStrategy := danube.NewReliableDispatchStrategy(reliableOpts)

    producer, err := client.NewProducer(ctx).
        WithName(producerName).
        WithTopic(topic).
        WithSchema("test_schema", danube.SchemaType_STRING).
        WithDispatchStrategy(reliableStrategy).
        Build()
    if err != nil {
        log.Fatalf("unable to initialize the producer: %v", err)
    }

    if err := producer.Create(ctx); err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }

    payload := fmt.Sprintf("Hello Danube %d", i)
    // Convert string to bytes
    bytes_payload := []byte(payload)

    messageID, err := producer.Send(ctx, bytes_payload , nil)
    if err != nil {
        log.Fatalf("Failed to send message: %v", err)
    }
    ```

## Complete examples

For complete code examples of using producers and consumers, check the links:

* [Rust Examples](https://github.com/danube-messaging/danube/tree/main/danube-client/examples)
* [Go Examples](https://github.com/danube-messaging/danube-go/tree/main/examples)
