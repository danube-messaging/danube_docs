# Welcome to Danube Pub/Sub messaging docs

Danube is an open-source distributed Pub/Sub messaging platform (inspired by Apache Pulsar).

Check-out the Docs for more details of the Danube Architecture and the supported concepts.

⚠️ The messsaging platform is currently under active development and may have missing or incomplete functionalities. Use with caution.

I'm continuously working on enhancing and adding new features. Contributions are welcome, and you can also report any issues you encounter. The client library is currently written in Rust, with a Go client potentially coming soon. Contributions in other languages, such as Python, Java, etc., are also greatly appreciated.

Currently, the Danube system exclusively supports Non-persistent messaging. This means messages reside solely in memory and are promptly distributed to consumers if they are available, utilizing a dispatch mechanism based on subscription types.
