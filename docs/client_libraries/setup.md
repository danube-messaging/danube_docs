# Configure Danube Client

First you need to create the `DanubeClient`.
The method `service_url` configures the base URI, that is used for connecting to the Danube Messaging System. The URI should include the protocol and address of the Danube service.

=== "Rust"

    ```rust
    use danube_client::DanubeClient;

    #[tokio::main]
    async fn main() -> Result<()> {
        // Setup tracing
        tracing_subscriber::fmt::init();

        let client = DanubeClient::builder()
            .service_url("http://127.0.0.1:6650")
            .build()
            .await?;
    }
    ```

=== "Go"

    ```go
    import "github.com/danube-messaging/danube-go"

    func main() {

        client := danube.NewClient().ServiceURL("127.0.0.1:6650").Build()

    }
    ```

## Configure Danube client with TLS

To enable TLS for secure communication between the client and the Danube broker, you need to configure the client with the appropriate certificate.

=== "Rust"

    ```rust
    use danube_client::DanubeClient;
    use rustls::crypto;
    use tokio::sync::OnceCell;

    static CRYPTO_PROVIDER: OnceCell<()> = OnceCell::const_new();

    #[tokio::main]
    async fn main() -> Result<()> {
        CRYPTO_PROVIDER.get_or_init(|| async {
            let crypto_provider = crypto::ring::default_provider();
            crypto_provider
                .install_default()
                .expect("Failed to install default CryptoProvider");
        })
        .await;

    let client = DanubeClient::builder()
        .service_url("https://127.0.0.1:6650")
        .with_tls("../cert/ca-cert.pem")?
        .build()
        .await?;
    }
    ```

=== "Go"

    ```go
    import "github.com/danube-messaging/danube-go"

    func main() {

       //  TLS support not yet implemented

    }
    ```

## Configure Danube client with TLS and JWT token for authentication

In addition to TLS connectivity in the below example we are using the JWT token to autheticate the requests. This token is usually obtained by logging into an application service and generating an API key, or provided by the admin of the service.

The API key is used to request a JWT token from the authentication service. The JWT token includes claims that identify and authorize the client.
Once a JWT token is obtained, the Danube client will include it in the `Authorization` header of all the next requests.

=== "Rust"

    ```rust
    use danube_client::DanubeClient;
    use rustls::crypto;
    use tokio::sync::OnceCell;


    static CRYPTO_PROVIDER: OnceCell<()> = OnceCell::const_new();

    #[tokio::main]
    async fn main() -> Result<()> {
        CRYPTO_PROVIDER.get_or_init(|| async {
            let crypto_provider = crypto::ring::default_provider();
            crypto_provider
                .install_default()
                .expect("Failed to install default CryptoProvider");
        })
        .await;

    let client = DanubeClient::builder()
        .service_url("https://127.0.0.1:6650")
        .with_tls("../cert/ca-cert.pem")?
        .with_api_key("provided_api_key".to_string())
        .build()
        .await?;
    }
    ```

=== "Go"

    ```go
    import "github.com/danube-messaging/danube-go"

    func main() {

       //  TLS support not yet implemented

    }
    ```
