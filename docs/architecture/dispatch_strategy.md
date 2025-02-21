# Dispatch Strategy

The dispatch strategies in Danube represent two distinct approaches to message delivery, each serving different use cases:

### Non-Reliable Dispatch Strategy

This strategy prioritizes speed and minimal resource usage by delivering messages directly from producers to subscribers without storing them. Messages flow through the broker in a "fire and forget" manner, achieving the lowest possible latency. It's particularly effective for use cases like real-time metrics, live data streams, or scenarios where occasional message loss is acceptable in favor of maximum throughput and performance.

### Reliable Dispatch Strategy

This strategy ensures guaranteed message delivery by implementing a store-and-forward mechanism. When a message arrives, it's first stored in the chosen storage backend before being dispatched to subscribers. The storage options include:

* Local Disk: Messages are persisted to the broker's filesystem, providing durability with lightweight operation
* GRPC Storage Interface: Connects to external storage systems using [danube-storage](https://github.com/danube-messaging/danube-storage) components. It can be implemented for cloud storage as AWS S3, GCP Storage or Azure Blob Storage, also for distributed object / block storage systems like MINIO,ROOK, LONGHORN etc.
* InMemory: Messages are kept in RAM for quick access, only for development & teting environments

The Reliable Dispatch maintains message state and tracks delivery acknowledgments, ensuring messages are not lost even if subscribers are temporarily unavailable.

These dispatch strategies embody Danube's design principle of flexibility, letting users choose the right balance between performance and reliability based on their specific requirements. The ability to switch between these strategies or use them simultaneously for different topics makes Danube adaptable to various messaging patterns in distributed systems.
