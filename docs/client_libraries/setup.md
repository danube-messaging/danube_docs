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
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        client, err := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }
        _ = client
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
    import (
        "log"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        // TLS with custom CA certificate
        builder, err := danube.NewClient().
            ServiceURL("127.0.0.1:6650").
            WithTLS("./certs/ca-cert.pem")
        if err != nil {
            log.Fatalf("failed to configure TLS: %v", err)
        }

        client, err := builder.Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }
        _ = client
    }
    ```

    For mutual TLS (mTLS) with client certificates:

    ```go
    // mTLS with CA, client cert, and client key
    builder, err := danube.NewClient().
        ServiceURL("127.0.0.1:6650").
        WithMTLS("./certs/ca-cert.pem", "./certs/client-cert.pem", "./certs/client-key.pem")
    if err != nil {
        log.Fatalf("failed to configure mTLS: %v", err)
    }

    client, err := builder.Build()
    if err != nil {
        log.Fatalf("failed to create client: %v", err)
    }
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
    import (
        "log"
        "os"

        "github.com/danube-messaging/danube-go"
    )

    func main() {
        apiKey := os.Getenv("DANUBE_API_KEY")
        if apiKey == "" {
            log.Fatal("DANUBE_API_KEY environment variable not set")
        }

        // WithAPIKey automatically enables TLS with system CA roots
        client, err := danube.NewClient().
            ServiceURL("127.0.0.1:6650").
            WithAPIKey(apiKey).
            Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }
        _ = client
    }
    ```

    To combine API key authentication with a custom CA certificate:

    ```go
    builder, err := danube.NewClient().
        ServiceURL("127.0.0.1:6650").
        WithTLS("./certs/ca-cert.pem")
    if err != nil {
        log.Fatalf("failed to configure TLS: %v", err)
    }

    client, err := builder.WithAPIKey(apiKey).Build()
    if err != nil {
        log.Fatalf("failed to create client: %v", err)
    }
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

## Environment-Based Configuration

```bash
# Production
export DANUBE_URL=https://danube.example.com:6650
export DANUBE_CA_CERT=/etc/danube/certs/ca.pem
export DANUBE_API_KEY=your-secret-api-key
```

---

## Next Steps

- **[Producer Guide](producer.md)** - Start sending messages
- **[Consumer Guide](consumer.md)** - Start receiving messages
- **[Schema Registry](schema-registry.md)** - Add type safety with schemas
