# Danube Client Libraries

Danube provides official client libraries for multiple programming languages, allowing you to integrate messaging capabilities into your applications. All clients follow consistent patterns and support core Danube features including topics, subscriptions, partitions, and schema registry.

## Supported Languages

### Rust Client

The official [danube-client](https://crates.io/crates/danube-client) is an asynchronous Rust client library built on Tokio.

**Installation:**

```bash
cargo add danube-client
```

**Features:**

- ✅ Full async/await support with Tokio
- ✅ Type-safe schema registry integration
- ✅ Partitioned topics
- ✅ Reliable dispatch
- ✅ TLS and JWT authentication
- ✅ All subscription types (Exclusive, Shared, Failover)
- ✅ Schema validation (JSON Schema, Avro, Protobuf)

**Learn more:** [Rust Examples](https://github.com/danube-messaging/danube/tree/main/danube-client/examples)

---

### Go Client

The official [danube-go](https://pkg.go.dev/github.com/danube-messaging/danube-go) library provides Go language bindings.

**Installation:**

```bash
go get github.com/danube-messaging/danube-go
```

**Features:**

- ✅ Context-based operations
- ✅ Partitioned topics
- ✅ Reliable dispatch
- ✅ All subscription types (Exclusive, Shared, Failover)
- ✅ Schema registry (JSON Schema, Avro, Protobuf)
- ✅ Schema versioning and compatibility checking
- ✅ TLS and mTLS support
- ✅ JWT authentication (API key)

**Learn more:** [Go Examples](https://github.com/danube-messaging/danube-go/tree/main/examples)

---

## Feature Comparison Matrix

| Feature | Rust | Go | Python* | Java* |
|---------|------|-----|---------|-------|
| **Core Messaging** |
| Producers | ✅ | ✅ | ⏳ | ⏳ |
| Consumers | ✅ | ✅ | ⏳ | ⏳ |
| Partitioned Topics | ✅ | ✅ | ⏳ | ⏳ |
| Reliable Dispatch | ✅ | ✅ | ⏳ | ⏳ |
| **Subscriptions** |
| Exclusive | ✅ | ✅ | ⏳ | ⏳ |
| Shared | ✅ | ✅ | ⏳ | ⏳ |
| Failover | ✅ | ✅ | ⏳ | ⏳ |
| **Schema Registry** |
| JSON Schema | ✅ | ✅ | ⏳ | ⏳ |
| Avro | ✅ | ✅ | ⏳ | ⏳ |
| Protobuf | ✅ | ✅ | ⏳ | ⏳ |
| Compatibility Checking | ✅ | ✅ | ⏳ | ⏳ |
| **Security** |
| TLS / mTLS | ✅ | ✅ | ⏳ | ⏳ |
| JWT Authentication | ✅ | ✅ | ⏳ | ⏳ |

_* Coming soon - community contributions welcome_

---

## Community Clients

We encourage the community to develop and maintain clients for additional languages. If you're building a Danube client:

- Follow the [protocol specification](https://github.com/danube-messaging/danube/tree/main/danube-core/proto)
- Reference existing clients for patterns
- Submit a PR to add your client to this list

### Guidelines for Client Development

**Core Requirements:**

- Support for producer/consumer operations
- Schema registry integration
- Topic lookup and partitioning
- Subscription management (Exclusive, Shared, Failover)
- Message acknowledgment
- Error handling and retries

**Recommended Features:**

- TLS support
- JWT authentication
- Connection pooling
- Graceful shutdown

---

## Next Steps

- **[Client Setup](setup.md)** - Configure and connect your client
- **[Producer Guide](producer.md)** - Send messages to topics
- **[Consumer Guide](consumer.md)** - Receive and process messages
- **[Schema Registry](schema-registry.md)** - Work with typed messages

## Getting Help

- **Documentation:** [Danube Docs](https://github.com/danube-messaging/danube_docs)
- **Examples:** [Rust](https://github.com/danube-messaging/danube/tree/main/danube-client/examples) | [Go](https://github.com/danube-messaging/danube-go/tree/main/examples)
- **Issues:** [GitHub Issues](https://github.com/danube-messaging/danube/issues)
