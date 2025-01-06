# Welcome to Danube Pub/Sub messaging docs

[Danube](https://github.com/danube-messaging/danube) is an open-source, distributed publish-subscribe (Pub/Sub) message broker system developed in Rust.
Danube aims to be a powerful, flexible and scalable messaging solution. Allows single or multiple Producers to publish on the Topics and multiple Subscriptions to consume the messages from the Topic.
Inspired by the Apache Pulsar messaging and streaming platform, Danube incorporates some similar concepts but is designed to carve its own path within the distributed messaging ecosystem.

## Core Capabilities of the Danube messaging Platform

[**Topics**](architecture/topics.md): A unit of storage that organizes messages into a stream.

* **Non-partitioned topics**: Served by a single broker.
* **Partitioned topics**: Divided into partitions, served by different brokers within the cluster, enhancing scalability and fault tolerance.

[**Message Dispatch**](architecture/dispatch_strategy.md):

* **Non-reliable Message Dispatch**: Messages reside in memory and are promptly distributed to consumers, ideal for scenarios where speed is crucial.
* **Reliable Message Dispatch**: Supports configurable storage options including in-memory, disk, and S3, ensuring message persistence and durability.

**Metadata Store**:
**ETCD as Default**: Provides a reliable and consistent Metadata store for cluster synchronization.
**Configurable Options**: Allows customization of metadata storage to fit specific requirements.

[**Subscription Types**](architecture/subscriptions.md):
Supports various subscription types (exclusive, shared, failover) enabling different messaging patterns such as message queueing and pub-sub.

**Flexible Message Schemas**:
Supports multiple message schemas (bytes, string, int64, JSON) providing flexibility in message format and structure.

### Crates within the [Danube workspace](https://github.com/danube-messaging/danube)

The crates part of the Danube workspace:

* [danube-broker](https://github.com/danube-messaging/danube/tree/main/danube-broker) - The main crate, danube pubsub platform
  * [danube-reliable-dispatch](https://github.com/danube-messaging/danube/tree/main/danube-reliable-dispatch/src) - Part of danube-broker, responsible of reliable dispatching
  * [danube-metadata-store](https://github.com/danube-messaging/danube/tree/main/danube-metadata-store/src) - Part of danube-broker, responsibile of Metadata storage
* [danube-client](https://github.com/danube-messaging/danube/tree/main/danube-client) - An async Rust client library for interacting with Danube Pub/Sub messaging platform
* [danube-cli](https://github.com/danube-messaging/danube/tree/main/danube-cli) - Client CLI to handle message publishing and consumption
* [danube-admin-cli](https://github.com/danube-messaging/danube/tree/main/danube-admin-cli) - Admin CLI designed for interacting with and managing the Danube cluster

## Danube client libraries

* [danube-client](https://crates.io/crates/danube-client) - Danube Pub/Sub async Rust client library
* [danube-go](https://pkg.go.dev/github.com/danube-messaging/danube-go) - Danube Pub/Sub Go client library

Contributions in other languages, such as Python, Java, etc., are also greatly appreciated.

## Articles

Some of the early articles may not be accurate as the API has changed significantly with the latest releases.

* [Danube - Pub-Sub message broker - intro](https://dev-state.com/posts/danube_intro/)
* [Danube: Queuing and Pub/Sub patterns](https://dev-state.com/posts/danube_pubsub/)
* [Setting Up Danube Go Client with Message Brokers on Kubernetes](https://dev-state.com/posts/danube_demo/)
* [Danube platform updates - v0.2.0](https://dev-state.com/posts/danube_update_020/)
