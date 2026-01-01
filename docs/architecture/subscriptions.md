# Subscription

**A Danube subscription** is a named configuration rule that determines how messages are delivered to consumers. It is a lease on a topic established by a group of consumers.

Danube permits multiple producers and subscribers to the same topic. The Subscription Types can be combined to obtain [message queueing or fan-out](messaging_patterns_queuing_vs_pubsub.md) pub-sub messaging patterns.

![Producers  Consumers](img/producers_consumers.png "Producers Consumers")

## Exclusive

The **Exclusive** type is a subscription type that only allows a single consumer to attach to the subscription. If multiple consumers subscribe to a topic using the same subscription, an error occurs.
This consumer has exclusive access to all messages published to the topic or partition.

### Exclusive subscription on Non-Partitioned Topic

* `Consumer`: Only one consumer can be attached to the topic with an Exclusive subscription.
* `Message Handling`: The single consumer handles all messages from the topic, receiving every message published to that topic.

![Exclusive Non-Partitioned](img/exclusive_subscription_non_partitioned.png "Exclusive Non-Partitioned")

### Exclusive subscription on Partitioned Topic (Multiple Partitions)

* `Consumer`: One consumer is allowed to connect to the subscription across all partitions of the partitioned topic.
* `Message Handling` : This single consumer processes messages from all partitions of the partitioned topic. If a topic is partitioned into multiple partitions, the exclusive consumer handles messages from every partition.

![Exclusive Partitioned](img/exclusive_subscription_partitioned.png "Exclusive Partitioned")

## Shared

In Danube, the **Shared** subscription type allows multiple consumers to attach to the same subscription. Messages are delivered in a round-robin distribution across consumers, and any given message is delivered to only one consumer.

### Shared subscription on Non-Partitioned Topic

* `Consumers`: Multiple consumers can subscribe to the same topic.
* `Message Handling`: Messages are distributed among all consumers in a round-robin fashion.

![Shared Non-Partitioned](img/shared_subscription_non_partitioned.png "Shared Non-Partitioned")

### Shared subscription on Partitioned Topic (Multiple Partitions)

* `Consumers`: Multiple consumers can subscribe to the partitioned topic.
* `Message Handling`: Messages are distributed across all partitions, and then among consumers in a round-robin fashion. Each message from any partition is delivered to only one consumer.

![Shared Partitioned](img/shared_subscription_partitioned.png "Shared Partitioned")

## Failover

The **Failover** subscription type allows multiple consumers to attach to the same subscription, with one active consumer at a time. If the active consumer disconnects or becomes unhealthy, another consumer automatically takes over. This preserves ordering and minimizes downtime.

### Failover subscription on Non-Partitioned Topic

* `Consumers`: One active consumer processes all messages; additional consumers are in standby.
* `Message Handling`: If the active consumer fails, a standby consumer takes over and continues from the last acknowledged position.

![Failover Non-Partitioned](img/failover_subscription_non_partitioned.png "Failover Non-Partitioned")

### Failover subscription on Partitioned Topic (Multiple Partitions)

* `Consumers`: One active consumer per partition; other consumers remain on standby for each partition.
* `Message Handling`: Failover occurs independently per partition, ensuring continuity and ordering within each partition.

![Failover Partitioned](img/failover_subscription_partitioned.png "Failover Partitioned")
