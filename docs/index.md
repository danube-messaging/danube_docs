# Welcome to Danube Pub/Sub messaging docs

[Danube](https://github.com/danrusei/danube) is an open-source, distributed publish-subscribe (Pub/Sub) message broker system developed in Rust. Inspired by the Apache Pulsar messaging and streaming platform, Danube incorporates some similar concepts but is designed to carve its own path within the distributed messaging ecosystem.

Currently, the Danube platform exclusively supports Non-persistent messages. Meaning that  the messages reside solely in memory and are promptly distributed to consumers if they are available, utilizing a dispatch mechanism based on subscription types.

⚠️ The messsaging platform is currently under active development and may have missing or incomplete functionalities. Use with caution.

I'm continuously working on enhancing and adding new features. Contributions are welcome, and you can also report any issues you encounter. The client library is currently written in Rust, with a Go client potentially coming soon. Contributions in other languages, such as Python, Java, etc., are also greatly appreciated.
