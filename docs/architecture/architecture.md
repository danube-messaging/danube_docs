# Danube Messaging Architecture

The Danube messaging system is a distributed messaging system, based on a publish-subscribe model, aiming to provide high throughput and low latency.

The Danube Messaging system's architecture is designed for flexibility and scalability, making it suitable for event-driven and cloud-native applications. Its decoupled and pluggable architecture allows for independent scaling and easy integration of various storage backends. Using the dispatch strategies and the subscription models the system can accommodate different messaging patterns.

![Danube Messaging Architecture](img/Danube_architecture.png "Danube Messaging Architecture")

### Brokers

Brokers are the core of the Danube Messaging system, responsible for routing and distributing messages, managing client connections and subscriptions, and implementing both reliable and non-reliable dispatch strategies. They act as the main entry point for publishers and subscribers, ensuring efficient and effective message flow.

The Producers and Consumers connect to the Brokers to publish and consume messages, and use the subscription and dispatch mechanisms to accommodate various messaging patterns and reliability requirements.

### Metadata Storage

The ETCD cluster serves as the metadata storage for the system by maintaining configuration data, topic information, and broker coordination and load-balancing, ensuring the entire system operates with high availability and consistent state management across all nodes.

### Storage Layer

The Storage Layer is responsible for message durability and replay. Danube now uses a cloud-native model based on a local Write-Ahead Log (WAL) for the hot path and background persistence to cloud object storage via OpenDAL. This keeps publish/dispatch latency low while enabling durable, elastic storage across providers (S3, GCS, Azure Blob, local FS, memory).

Readers use tiered access: if data is within local WAL retention it is served from WAL/cache; otherwise historical data is streamed from cloud objects (using ETCD metadata) and seamlessly handed off to the WAL tail.

### Non-Reliable Dispatch

Non-Reliable Dispatch operates with zero storage overhead, as messages flow directly from publishers to subscribers without intermediate persistence. This mode delivers maximum performance and lowest latency, making it ideal for scenarios where occasional message loss is acceptable, such as real-time metrics or live streaming data.

### Reliable Dispatch

Reliable Dispatch offers guaranteed message delivery backed by the WAL + cloud persistence:

* **WAL on local disk** for low-latency appends and fast replay from in-memory cache and WAL files.
* **Cloud object storage via OpenDAL** (S3/GCS/Azure/FS/Memory) for durable, scalable historical data, uploaded asynchronously by a background uploader with resumable checkpoints.
* **ETCD metadata** tracks cloud objects and sparse indexes for efficient historical reads.

The ability to choose between these dispatch modes gives users the flexibility to optimize their messaging infrastructure based on their specific requirements for performance, reliability, and resource utilization.

For details and provider-specific configuration, see [Persistence (WAL + Cloud)](persistence.md).

### Design Considerations

#### Decoupled Architecture

The Danube Messaging system features a decoupled architecture where components are loosely coupled, allowing for independent scaling, easy maintenance and upgrades, and failure isolation.

#### Plugin Architecture

With a plugin architecture, the system supports flexible storage backend options, making it easy to extend and customize according to different use cases. This adaptability ensures that the system can meet diverse application requirements and is cloud-native ready.

#### Event-Driven Focus

Optimized for event-driven systems, the Danube Messaging system supports various message delivery patterns and scalable message processing. Its design is well-suited for microservices, providing efficient and scalable handling of event-driven workloads.
