# Producer

A producer is a process that attaches to a topic and publishes messages to a Danube broker. The Danube broker processes the messages.

**Access Mode** is a mechanism to determin the permissions of producers on topics.

* **Shared** - Multiple producers can publish on a topic.
* **Exclusive** - If there is already a producer connected, other producers trying to publish on this topic get errors immediately.

## Example

=== "Rust"

    ```rust
    let topic = "/default/test_topic".to_string();

    let json_schema = r#"{"type": "object", "properties": {"field1": {"type": "string"}, "field2": {"type": "integer"}}}"#.to_string();

    let mut producer = client
        .new_producer()
        .with_topic(topic)
        .with_name("test_producer1")
        .with_schema("my_app".into(), SchemaType::Json(json_schema))
        .build();

    let prod_id = producer.create().await?;
    info!("The Producer was created with ID: {:?}", prod_id);

    let data = json!({
            "field1": format!{"value{}", i},
            "field2": 2020+i,
        });

    // Convert to string and encode to bytes
    let json_string = serde_json::to_string(&data).unwrap();
    let encoded_data = json_string.as_bytes().to_vec();

    // let json_message = r#"{"field1": "value", "field2": 123}"#.as_bytes().to_vec();
    let message_id = producer.send(encoded_data).await?;
    println!("The Message with id {} was sent", message_id);
    ```

=== "Go"

    ```go
    ctx := context.Background()
    topic := "/default/test_topic"
    jsonSchema := `{"type": "object", "properties": {"field1": {"type": "string"}, "field2": {"type": "integer"}}}`

    producer, err := client.NewProducer(ctx).
        WithName("test_producer").
        WithTopic(topic).
        WithSchema("test_schema", danube.SchemaType_JSON, jsonSchema).
        Build()
    if err != nil {
        log.Fatalf("unable to initialize the producer: %v", err)
    }

    producerID, err := producer.Create(ctx)
    if err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }
    log.Printf("The Producer was created with ID: %v", producerID)

    data := map[string]interface{}{
        "field1": fmt.Sprintf("value%d", 24),
        "field2": 2024,
     }

    jsonData, err := json.Marshal(data)
    if err != nil {
        log.Fatalf("Failed to marshal data: %v", err)
    }

    messageID, err := producer.Send(ctx, jsonData)
    if err != nil {
        log.Fatalf("Failed to send message: %v", err)
    }
        log.Printf("The Message with id %v was sent", messageID)
    ```

## Complete example

For a complete example implementation of the above code using producers and consumers, check the examples:

* [Rust Examples](https://github.com/danrusei/danube/tree/main/danube-client/examples)
* [Go Examples](https://github.com/danrusei/danube-go/tree/main/examples)
