# Pub-Sub messaging

**Danube** is built on the [publish-subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) pattern. In this pattern, producers publish messages to topics; consumers subscribe to those topics, process incoming messages, and send acknowledgments to the broker when processing is finished.

## Queuing vs Pub/Sub (fan-out)

Below is some general documentation about Queuing and Pub-Sub, not related to Danube implementation, but just to ensure we are aware of the standards.

Messaging queuing and pub-sub are both messaging patterns used for asynchronous communication between applications, but they differ in their approach:

### Queuing Messaging

* **Model**: Point-to-Point (One-to-One). A message producer sends a message to a specific queue, and only one consumer can receive and process that message.
* **Order**: Messages are typically processed in the order they are received (FIFO - First-In-First-Out). This ensures tasks are completed sequentially.
* **Delivery**: Messages are guaranteed to be delivered at least once. This reliability is crucial for critical tasks.

Examples: Use cases include processing orders, sending emails, or handling failed transactions.

### Pub/Sub Messaging (fan-out)

* **Model**: Publish-Subscribe (One-to-Many). A producer publishes messages to a topic, and any interested subscribers can receive the message. Multiple subscribers can receive the same message.
* **Order**: Message order is not guaranteed. Subscribers receive messages as they are published. This is suitable for real-time updates or notifications.
* **Delivery**: Delivery is often "fire-and-forget," meaning there's no guarantee a subscriber receives the message. This is acceptable for non-critical data.

Examples: Use cases include broadcasting stock price updates, sending chat messages, or triggering real-time analytics.

In Summary:

* **Messaging queues** are for reliable, ordered delivery to a single consumer, ideal for task processing.
* **Pub-sub** is for broadcasting messages to many interested parties, good for real-time updates.

## Danube Subscriptions

**A Danube subscription** is a named configuration rule that determines how messages are delivered to consumers. It is a lease on a topic established by a group of consumers:

* **Exclusive (can be used for pub-sub)** - The exclusive type is a subscription type that only allows a single consumer to attach to the subscription. If multiple consumers subscribe to a topic using the same subscription, an error occurs.
* **Shared (for queuing)** - The shared subscription type Danube allows multiple consumers to attach to the same subscription. Messages are delivered in a round-robin distribution across consumers, and any given message is delivered to only one consumer.

### Queuing or Pub/Sub (fan-out) using Danube

* If you want to achieve **message queuing** among consumers, **share the same subscription name** among multiple consumers
* If you want to achieve traditional **fan-out pub-sub messaging** among consumers, **specify a unique subscription name for each consumer** with an exclusive subscription type.
