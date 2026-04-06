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

=== "Python"

    ```python
    import asyncio
    from danube import DanubeClientBuilder

    async def main():
        client = await (
            DanubeClientBuilder()
            .service_url("http://127.0.0.1:6650")
            .build()
        )

    asyncio.run(main())
    ```

=== "Java"

    ```java
    import com.danubemessaging.client.DanubeClient;

    public class Main {
        public static void main(String[] args) throws Exception {
            DanubeClient client = DanubeClient.builder()
                    .serviceUrl("http://127.0.0.1:6650")
                    .build();

            // use client ...

            client.close();
        }
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

=== "Python"

    ```python
    import asyncio
    from danube import DanubeClientBuilder

    async def main():
        # TLS with custom CA certificate
        client = await (
            DanubeClientBuilder()
            .service_url("https://127.0.0.1:6650")
            .with_tls("./certs/ca-cert.pem")
            .build()
        )

    asyncio.run(main())
    ```

    For mutual TLS (mTLS) with client certificates:

    ```python
    # mTLS with CA, client cert, and client key
    client = await (
        DanubeClientBuilder()
        .service_url("https://127.0.0.1:6650")
        .with_mtls(
            "./certs/ca-cert.pem",
            "./certs/client-cert.pem",
            "./certs/client-key.pem",
        )
        .build()
    )
    ```

=== "Java"

    ```java
    import com.danubemessaging.client.DanubeClient;
    import java.nio.file.Path;

    // TLS with custom CA certificate
    DanubeClient client = DanubeClient.builder()
            .serviceUrl("https://127.0.0.1:6650")
            .withTls(Path.of("./certs/ca-cert.pem"))
            .build();
    ```

    For mutual TLS (mTLS) with client certificates:

    ```java
    // mTLS with CA, client cert, and client key
    DanubeClient client = DanubeClient.builder()
            .serviceUrl("https://127.0.0.1:6650")
            .withMutualTls(
                    Path.of("./certs/ca-cert.pem"),
                    Path.of("./certs/client-cert.pem"),
                    Path.of("./certs/client-key.pem"))
            .build();
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

When the broker is running with `auth.mode: tls`, all client requests must carry a valid JWT token. Tokens are created offline using `danube-admin` and passed to the client at construction time — **no API key exchange occurs at runtime**.

### Creating a Token

Tokens are generated offline using `danube-admin` (no broker connection needed):

```bash
# Create a service account token (default: 1-year TTL)
danube-admin security tokens create \
  --subject my-app \
  --secret-key your-secret-key

# Create a token with custom TTL and issuer
danube-admin security tokens create \
  --subject my-app \
  --ttl 24h \
  --issuer danube-auth \
  --secret-key your-secret-key
```

The `secret_key` must match the broker's `jwt.secret_key` configuration. The token's `sub` claim (subject) is the principal name used for RBAC authorization.

For a complete guide on tokens, roles, bindings, and RBAC setup, see [Security Concepts](../concepts/security.md).

### Connecting with a Token

Use `with_token` to authenticate. This automatically enables TLS:

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

        let token = std::env::var("DANUBE_TOKEN")
            .expect("DANUBE_TOKEN environment variable not set");

        let client = DanubeClient::builder()
            .service_url("https://127.0.0.1:6650")
            .with_tls("./certs/ca-cert.pem")?
            .with_token(&token)
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
        token := os.Getenv("DANUBE_TOKEN")
        if token == "" {
            log.Fatal("DANUBE_TOKEN environment variable not set")
        }

        // WithToken automatically enables TLS with system CA roots
        client, err := danube.NewClient().
            ServiceURL("127.0.0.1:6650").
            WithToken(token).
            Build()
        if err != nil {
            log.Fatalf("failed to create client: %v", err)
        }
        _ = client
    }
    ```

    To combine token authentication with a custom CA certificate:

    ```go
    builder, err := danube.NewClient().
        ServiceURL("127.0.0.1:6650").
        WithTLS("./certs/ca-cert.pem")
    if err != nil {
        log.Fatalf("failed to configure TLS: %v", err)
    }

    client, err := builder.WithToken(token).Build()
    if err != nil {
        log.Fatalf("failed to create client: %v", err)
    }
    ```

=== "Python"

    ```python
    import asyncio
    import os
    from danube import DanubeClientBuilder

    async def main():
        token = os.environ["DANUBE_TOKEN"]

        # with_token automatically enables TLS with system CA roots
        client = await (
            DanubeClientBuilder()
            .service_url("https://127.0.0.1:6650")
            .with_token(token)
            .build()
        )

    asyncio.run(main())
    ```

    To combine token authentication with a custom CA certificate:

    ```python
    client = await (
        DanubeClientBuilder()
        .service_url("https://127.0.0.1:6650")
        .with_tls("./certs/ca-cert.pem")
        .with_token(token)
        .build()
    )
    ```

=== "Java"

    ```java
    import com.danubemessaging.client.DanubeClient;

    String token = System.getenv("DANUBE_TOKEN");
    if (token == null || token.isBlank()) {
        throw new IllegalStateException("DANUBE_TOKEN environment variable not set");
    }

    // withToken automatically enables TLS
    DanubeClient client = DanubeClient.builder()
            .serviceUrl("https://127.0.0.1:6650")
            .withToken(token)
            .build();
    ```

    To combine token authentication with a custom CA certificate:

    ```java
    import java.nio.file.Path;

    DanubeClient client = DanubeClient.builder()
            .serviceUrl("https://127.0.0.1:6650")
            .withTls(Path.of("./certs/ca-cert.pem"))
            .withToken(token)
            .build();
    ```

**How it works:**

1. Token is created offline using `danube-admin security tokens create`
2. Client sends the token as `Authorization: Bearer <token>` on every gRPC request
3. Broker validates the token signature and expiration, then checks RBAC policies
4. No server-side token exchange — the client never calls an authentication RPC

**Security best practices:**

- Store tokens in environment variables or secret managers
- Never hardcode tokens in source code
- Use short-lived tokens with `--ttl` for production workloads
- Assign minimal RBAC permissions via roles and bindings

---

## Token Rotation

For environments where tokens need to be refreshed at runtime (e.g., Kubernetes projected volumes, vault-injected secrets), use `with_token_supplier`. The supplier function is called **on every gRPC request** to get the current token:

=== "Rust"

    ```rust
    let client = DanubeClient::builder()
        .service_url("https://127.0.0.1:6650")
        .with_tls("./certs/ca-cert.pem")?
        .with_token_supplier(|| {
            std::fs::read_to_string("/var/run/secrets/danube/token")
                .unwrap_or_default()
        })
        .build()
        .await?;
    ```

=== "Go"

    ```go
    client, err := danube.NewClient().
        ServiceURL("127.0.0.1:6650").
        WithTLS("./certs/ca-cert.pem").
        WithTokenSupplier(func() string {
            data, _ := os.ReadFile("/var/run/secrets/danube/token")
            return string(data)
        }).
        Build()
    ```

=== "Python"

    ```python
    def read_token():
        with open("/var/run/secrets/danube/token") as f:
            return f.read().strip()

    client = await (
        DanubeClientBuilder()
        .service_url("https://127.0.0.1:6650")
        .with_tls("./certs/ca-cert.pem")
        .with_token_supplier(read_token)
        .build()
    )
    ```

=== "Java"

    ```java
    import java.nio.file.Files;
    import java.nio.file.Path;

    DanubeClient client = DanubeClient.builder()
            .serviceUrl("https://127.0.0.1:6650")
            .withTls(Path.of("./certs/ca-cert.pem"))
            .withTokenSupplier(() ->
                Files.readString(Path.of("/var/run/secrets/danube/token")).strip()
            )
            .build();
    ```

---

## Environment-Based Configuration

```bash
# Production
export DANUBE_URL=https://danube.example.com:6650
export DANUBE_CA_CERT=/etc/danube/certs/ca.pem
export DANUBE_TOKEN=$(danube-admin security tokens create \
  --subject my-app --secret-key your-secret-key)
```

---

## Next Steps

- **[Producer Guide](producer.md)** - Start sending messages
- **[Consumer Guide](consumer.md)** - Start receiving messages
- **[Schema Registry](schema-registry.md)** - Add type safety with schemas
- **[Security Concepts](../concepts/security.md)** - Full RBAC setup guide

