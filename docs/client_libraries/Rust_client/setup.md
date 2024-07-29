# Getting Started with `danube-client`

The `danube-client` is an asynchronous Rust client library designed for interacting with the Danube Pub/Sub messaging platform. This guide will walk you through the steps to add the `danube-client` library to your Rust project.

## Adding `danube-client` to Your Project

Use the `cargo add` command to add `danube-client` to your `Cargo.toml` file.

``` bash
cargo add danube-client
```

## Client Setup

Before an application creates a producer/consumer, the  client library needs to initiate a setup phase including two steps:

* The client attempts to determine the owner of the topic by sending a Lookup request to Broker.  
* Once the client library has the broker address, it creates a RPC connection (or reuses an existing connection from the pool) and (in later stage authenticates it ).
* Within this connection, the clients (producer, consumer) and brokers exchange RPC commands. At this point, the client sends a command to create producer/consumer to the broker, which will comply after doing some validation checks.

``` rust
use danube_client::DanubeClient;

#[tokio::main]
async fn main() -> Result<()> {
    // Setup tracing
    tracing_subscriber::fmt::init();

    let client = DanubeClient::builder()
        .service_url("http://[::1]:6650")
        .build()
        .unwrap();
}
```

## Refer to the Documentation

For more details on how to use the library, including available methods and configuration options, refer to the [docs.rs](https://docs.rs/danube-client/latest/danube_client/) documentation.
