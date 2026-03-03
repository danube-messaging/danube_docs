# Danube Messaging Architecture

The Danube messaging system is a distributed messaging system, based on a publish-subscribe model, aiming to provide high throughput and low latency.

The Danube Messaging system's architecture is designed for flexibility and scalability, making it suitable for event-driven and cloud-native applications. Its decoupled and pluggable architecture allows for independent scaling and easy integration of various storage backends. Using the dispatch strategies and the subscription models the system can accommodate different messaging patterns.

![Danube Messaging Architecture](../assets/img/architecture/Danube_architecture.png "Danube Messaging Architecture")

### Brokers

Brokers are the core of the Danube Messaging system, responsible for routing and distributing messages, managing client connections and subscriptions, and implementing both reliable and non-reliable dispatch strategies. They act as the main entry point for publishers and subscribers, ensuring efficient and effective message flow.

The Producers and Consumers connect to the Brokers to publish and consume messages, and use the subscription and dispatch mechanisms to accommodate various messaging patterns and reliability requirements.

### Metadata Storage

Each broker embeds a Raft consensus node (powered by [openraft](https://github.com/databendlabs/openraft) with [redb](https://github.com/cberner/redb) for durable log storage). Together the brokers form a replicated state machine that stores cluster metadata: configuration, topic ownership, broker registration, and load-balancing state. Writes go through Raft consensus for strong consistency; reads are served from the local in-memory state machine with zero network hops. No external metadata store is required.

### Storage Layer

The Storage Layer is responsible for message durability and replay. Danube now uses a cloud-native model based on a local Write-Ahead Log (WAL) for the hot path and background persistence to cloud object storage via OpenDAL. This keeps publish/dispatch latency low while enabling durable, elastic storage across providers (S3, GCS, Azure Blob, local FS, memory).

Readers use tiered access: if data is within local WAL retention it is served from WAL/cache; otherwise historical data is streamed from cloud objects (using Raft-replicated metadata) and seamlessly handed off to the WAL tail.

### Non-Reliable Dispatch

Non-Reliable Dispatch operates with zero storage overhead, as messages flow directly from publishers to subscribers without intermediate persistence. This mode delivers maximum performance and lowest latency, making it ideal for scenarios where occasional message loss is acceptable, such as real-time metrics or live streaming data.

### Reliable Dispatch

Reliable Dispatch offers guaranteed message delivery backed by the WAL + cloud persistence:

* **WAL on local disk** for low-latency appends and fast replay from in-memory cache and WAL files.
* **Cloud object storage via OpenDAL** (S3/GCS/Azure/FS/Memory) for durable, scalable historical data, uploaded asynchronously by a background uploader with resumable checkpoints.
* **Raft-replicated metadata** tracks cloud objects and sparse indexes for efficient historical reads.

The ability to choose between these dispatch modes gives users the flexibility to optimize their messaging infrastructure based on their specific requirements for performance, reliability, and resource utilization.

For details and provider-specific configuration, see [Persistence (WAL + Cloud)](persistence.md).
