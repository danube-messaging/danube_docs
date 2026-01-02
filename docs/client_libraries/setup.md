# Client Setup and Configuration

This guide covers how to configure and connect Danube clients to your broker.

## Basic Connection

Connect to Danube broker with an gRPC endpoint:

=== "Rust"

    ```rust
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;

        Ok(())
    }
    ```

=== "Go"

    ```go
    import (
        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
    }
    ```

**Endpoint format:** `http://host:port` or `https://host:port` for TLS

---

## TLS Configuration

For secure production environments, enable TLS encryption:

=== "Rust"

    ```rust
    use danube_client::DanubeClient;
    use rustls::crypto;
    use tokio::sync::OnceCell;

    static CRYPTO_PROVIDER: OnceCell<()> = OnceCell::const_new();

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        // Initialize crypto provider (required once)
        CRYPTO_PROVIDER.get_or_init(|| async {
            let crypto_provider = crypto::ring::default_provider();
            crypto_provider
                .install_default()
                .expect("Failed to install default CryptoProvider");
        })
        .await;

        let client = DanubeClient::builder()
            .service_url("https://127.0.0.1:6650")
            .with_tls("./certs/ca-cert.pem")?
            .build()
            .await?;

        Ok(())
    }
    ```

=== "Go"

    ```go
    // TLS support coming soon
    ```

**Requirements:**

- CA certificate file (PEM format)
- HTTPS URL (`https://` instead of `http://`)
- Broker must be configured with TLS enabled

**Certificate paths:**

- Relative: `./certs/ca-cert.pem`
- Absolute: `/etc/danube/certs/ca-cert.pem`

---

## JWT Authentication

For authenticated environments, use API keys to obtain JWT tokens:

=== "Rust"

    ```rust
    use danube_client::DanubeClient;
    use rustls::crypto;
    use tokio::sync::OnceCell;

    static CRYPTO_PROVIDER: OnceCell<()> = OnceCell::const_new();

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        CRYPTO_PROVIDER.get_or_init(|| async {
            let crypto_provider = crypto::ring::default_provider();
            crypto_provider
                .install_default()
                .expect("Failed to install default CryptoProvider");
        })
        .await;

        let api_key = std::env::var("DANUBE_API_KEY")
            .expect("DANUBE_API_KEY environment variable not set");

        let client = DanubeClient::builder()
            .service_url("https://127.0.0.1:6650")
            .with_tls("./certs/ca-cert.pem")?
            .with_api_key(api_key)
            .build()
            .await?;

        Ok(())
    }
    ```

=== "Go"

    ```go
    // JWT authentication support coming soon
    ```

**How it works:**

1. Client exchanges API key for JWT token on first request
2. Token is cached and automatically renewed when expired
3. Token included in `Authorization` header for all requests
4. Default token lifetime: 1 hour

**Security best practices:**

- Store API keys in environment variables
- Never hardcode API keys in source code
- Use different API keys per environment (dev/staging/prod)
- Rotate API keys regularly

---

## Connection Options

### Connection Pooling

Clients automatically manage connection pools. Multiple producers/consumers share underlying connections efficiently.

=== "Rust"

    ```rust
    let client = DanubeClient::builder()
        .service_url("http://127.0.0.1:6650")
        .build()
        .await?;

    // All producers/consumers share the same connection pool
    let producer1 = client.new_producer().with_topic("/topic1").build();
    let producer2 = client.new_producer().with_topic("/topic2").build();
    let consumer = client.new_consumer().with_topic("/topic1").build();
    ```

### Service Discovery

For clustered deployments, the client performs automatic topic lookup:

    ```rust
// Client connects to any broker in the cluster
let client = DanubeClient::builder()
    .service_url("<http://broker1:6650>")
    .build()
    .await?;

// Topic lookup finds the owning broker
let producer = client.new_producer()
    .with_topic("/default/my-topic")
    .build();

// Producer connects to the correct broker automatically
producer.create().await?;
    ```

---

## Environment-Based Configuration

```bash
# Production
export DANUBE_URL=https://danube.example.com:6650
export DANUBE_CA_CERT=/etc/danube/certs/ca.pem
export DANUBE_API_KEY=your-secret-api-key
```

---

## Troubleshooting

### Connection Refused

```bash
Error: Connection refused (os error 111)
```

**Solutions:**

- Verify broker is running: `curl http://localhost:6650`
- Check firewall rules
- Confirm correct host and port

### TLS Certificate Errors

```bash
Error: InvalidCertificate
```

**Solutions:**

- Verify CA certificate path is correct
- Ensure certificate is PEM format
- Check certificate hasn't expired
- Confirm broker TLS configuration matches client

### Authentication Failures

```bash
Error: Unauthenticated
```

**Solutions:**

- Verify API key is valid
- Check broker authentication mode (tls vs tlswithjwt)
- Ensure token hasn't expired (client auto-renews, but check logs)

---

## Next Steps

- **[Producer Basics](producer-basics.md)** - Start sending messages
- **[Consumer Basics](consumer-basics.md)** - Start receiving messages
- **[Schema Registry](schema-registry.md)** - Add type safety with schemas
