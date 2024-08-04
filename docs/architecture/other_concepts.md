# Other concepts

**This is a placeholder for some concepts may be considered to be implemented in the future**.

## Multi-topic subscriptions

Using regex subscription.

## Partitioned topics

Normal topics are served only by a single broker, which limits the maximum throughput of the topic. Partitioned topic is a special type of topic handled by multiple brokers, thus allowing for higher throughput.

A partitioned topic is implemented as N internal topics, where N is the number of partitions. When publishing messages to a partitioned topic, each message is routed to one of several brokers. The distribution of partitions across brokers is handled automatically.

![Partitioned Topics](img/partitioned_topics.png "Partitioned topics")

Messages for the topic are broadcast to two consumers. The **routing mode** determines each message should be published to which partition, while the **subscription type** determines which messages go to which consumers.

### Routing modes

When publishing to partitioned topics, you must specify a routing mode. The routing mode determines each message should be published to which partition or which internal topic.

* **RoundRobinPartition** - The producer will publish messages across all partitions in round-robin fashion to achieve maximum throughput. If a key is specified on the message, the partitioned producer will **hash the key and assign message to a particular partition**.
* **SinglePartition** - If no key is provided, the producer will randomly pick one single partition and publish all the messages into that partition. While if a key is specified on the message, the partitioned producer will hash the key and assign message to a particular partition.
