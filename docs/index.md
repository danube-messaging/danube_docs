# Welcome to Danube Pub/Sub messaging docs

[Danube](https://github.com/danrusei/danube) is an open-source, distributed publish-subscribe (Pub/Sub) message broker system developed in Rust. Inspired by the Apache Pulsar messaging and streaming platform, Danube incorporates some similar concepts but is designed to carve its own path within the distributed messaging ecosystem.

Currently, the Danube platform exclusively supports Non-persistent messages. Meaning that the messages reside solely in memory and are promptly distributed to consumers if they are available, utilizing a dispatch mechanism based on subscription types.

We are continuously working on enhancing and adding new features. Contributions are welcome, and you can also report any issues you encounter.

The following crates are part of the [Danube workspace](https://github.com/danrusei/danube):

* danube-broker - The main crate, danube pubsub platform
* danube-admin - Admin CLI designed for interacting with and managing the Danube cluster
* danube-client - An async Rust client library for interacting with Danube Pub/Sub messaging platform
* danube-pubsub - CLI to handle message publishing and consumption

## Danube client libraries

* [danube-client](https://crates.io/crates/danube-client) - Danube Pub/Sub async Rust client library
* [danube-go](https://pkg.go.dev/github.com/danrusei/danube-go) - Danube Pub/Sub Go client library

Contributions in other languages, such as Python, Java, etc., are also greatly appreciated.

## Articles

* [Danube - Pub-Sub message broker - intro](https://dev-state.com/posts/danube_intro/)
* [Danube: Queuing and Pub/Sub patterns](https://dev-state.com/posts/danube_pubsub/)
* [Setting Up Danube Go Client with Message Brokers on Kubernetes](https://dev-state.com/posts/danube_demo/)

⚠️ The messaging platform is currently in active development, so some features might be missing or incomplete. Encourage you to use it with care.
