# Welcome to Danube Pub/Sub messaging docs

[Danube](https://github.com/danube-messaging/danube) is an open-source, distributed messaging broker platform, developed in Rust.

Danube aims to be a lightweight yet powerful, secure and scalable messaging platform, suitable for event-driven applications. Allows single or multiple **producers** to publish messages on the **topics** and multiple **consumers**, using the **subscription** models, to consume the messages from the topics.

Inspired by the Apache Pulsar messaging and streaming platform, Danube incorporates some similar concepts but is designed to carve its own path within the distributed messaging ecosystem. For additional design considerations, please refer to the [Danube Architecture](architecture/architecture.md) section.

## Core Capabilities of the Danube Messaging Platform

* [**Topics**](architecture/topics.md): A unit of storage that organizes messages into a stream.
  * **Non-partitioned topics**: Served by a single broker.
  * **Partitioned topics**: Divided into partitions, served by different brokers within the cluster, enhancing scalability and fault tolerance.
* [**Message Dispatch**](architecture/dispatch_strategy.md):
  * **Non-reliable Message Dispatch**: Messages reside in memory and are promptly distributed to consumers, ideal for scenarios where speed is crucial. The acknowledgement mechanism is ignored.
  * **Reliable Message Dispatch**: The acknowledgement mechanism and persistent storage is used to ensure guaranteed message delivery.
* [**Subscription Types:**](architecture/subscriptions.md):
  * Supports various subscription types (**Exclusive**, **Shared**, **Failover**) enabling different messaging patterns such as message queueing and pub-sub.
* **Flexible Message Schemas**
  * Supports multiple message schemas (**Bytes**, **String**, **Int64**, **JSON**) providing flexibility in message format and structure.

### Crates within the [Danube workspace](https://github.com/danube-messaging/danube)

Danube Broker core crates:

* [danube-broker](https://github.com/danube-messaging/danube/tree/main/danube-broker) - The main crate, danube pubsub platform
* [danube-core](https://github.com/danube-messaging/danube/tree/main/danube-core) - Danube messaging core types and traits
* [danube-metadata-store](https://github.com/danube-messaging/danube/tree/main/danube-metadata-store) - Responsibile of Metadata storage and cluster coordination.
* [danube-reliable-dispatch](https://github.com/danube-messaging/danube/tree/main/danube-reliable-dispatch/src) - Responsible of reliable dispatching and topic storage.
* [danube-persisitent-storage](https://github.com/danube-messaging/danube/tree/main/danube-persistent-storage) - Responsible of persistent storage mechanism.

Danube CLIs and client library:

* [danube-client](https://github.com/danube-messaging/danube/tree/main/danube-client) - An async Rust client library for interacting with Danube messaging system
* [danube-cli](https://github.com/danube-messaging/danube/tree/main/danube-cli) - Client CLI to handle message publishing and consumption
* [danube-admin-cli](https://github.com/danube-messaging/danube/tree/main/danube-admin-cli) - Admin CLI designed for interacting with and managing the Danube cluster

## Danube client libraries

* [danube-client](https://crates.io/crates/danube-client) - Danube messaging async Rust client library
* [danube-go](https://pkg.go.dev/github.com/danube-messaging/danube-go) - Danube messaging Go client library

Contributions in other languages, such as Python, Java, etc., are greatly appreciated.
