# Danube client library

Currently, the supported clients are the [Rust Client](https://github.com/danrusei/danube/tree/main/danube-client) and [Go Client](https://github.com/danrusei/danube-go) clients. However, the community is encouraged to contribute by developing clients in other programming languages.

## Patterns

The Danube permits multiple topics and subcriber to the same topic. The [Subscription Types](../architecture/Queuing_PubSub_messaging.md) can be combined to obtain message queueing or fan-out pub-sub messaging patterns.

![Producers  Consumers](../architecture/img/producers_consumers.png "Producers Consumers")

## Rust client

The Rust [danube-client](https://crates.io/crates/danube-client) is an asynchronous Rust client library. To start using the `danube-client` library in your Rust project, you need to add it as a dependency. You can do this by running the following command:

```bash
cargo add danube-client
```

This command will add danube-client to your `Cargo.toml` file. Once added, you can import and use the library in your Rust code to interact with the Danube Pub/Sub messaging platform.

## Go client

To start using the [danube-go](https://pkg.go.dev/github.com/danrusei/danube-go) library in your Go project, you need to add it as a dependency. You can do this by running the following command:

```bash
go get github.com/danrusei/danube-go
```

This command will fetch the `danube-go` library and add it to your `go.mod` file. Once added, you can import and use the library in your Go code to interact with the Danube Pub/Sub messaging platform.

## Community Danube clients

TBD
