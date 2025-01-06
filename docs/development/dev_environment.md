# Development Environment Setup for Danube Broker

This document guides you through setting up the development environment, running danube broker instances, and be able to effectively contribute to the code.

## Prerequisites

Before you get started, make sure you have the following installed:

- **Rust**: Ensure you have Rust installed. You can download and install it from [the Rust website](https://www.rust-lang.org/tools/install).

- **Docker**: Install Docker if you haven’t already. Follow the installation instructions on the [Docker website](https://docs.docker.com/get-docker/).

## Contributing to the Repository

1. **Fork the Repository**:

   - Go to the [Danube Broker GitHub repository](https://github.com/danube-messaging/danube).
   - Click the "Fork" button on the top right corner of the page to create your own copy of the repository.

2. **Clone Your Fork**:

   Once you have forked the repository, clone your forked repository:

   ```sh
   git clone https://github.com/<your-username>/danube.git
   cd danube
   ```

3. **Add the Original Repository as a Remote** (optional but recommended for keeping up-to-date):

   ```sh
   git remote add upstream https://github.com/danube-messaging/danube.git
   ```

## Building the Project

1. **Build the Project**:

   To build the Danube Broker:

   ```sh
   cargo build 
   or  
   cargo build --release
   ```

## Running ETCD

1. **Start ETCD**:

   Use the Makefile to start an ETCD instance. This will run ETCD in a Docker container.

   ```sh
   make etcd
   ```

2. **Clean Up ETCD**:

   To stop and remove the ETCD instance and its data:

   ```sh
   make etcd-clean
   ```

## Running a Single Broker Instance

1. **Start ETCD**:

   Ensure ETCD is running. If not, use the `make etcd` command to start it.

2. **Run the Broker**:

   Use the following command to start a single broker instance:

   ```sh
   RUST_LOG=danube_broker=info target/debug/danube-broker --config-file config/danube_broker.yml
   ```

## Running Multiple Broker Instances

1. **Start ETCD**:

   Ensure ETCD is running. Use:

   ```sh
   make etcd
   ```

2. **Run Multiple Brokers**:

   Use the following Makefile command to start multiple broker instances:

   ```sh
   make brokers
   ```

   This will start brokers on ports 6650, 6651, and 6652. Logs for each broker will be saved in `temp/` directory.

3. **Clean Up Broker Instances**:

   To stop all running broker instances:

   ```sh
   make brokers-clean
   ```

## Reading Logs

Logs for each broker instance are stored in the `temp/` directory. You can view them using:

```sh
cat temp/broker_<port>.log
```

Replace `<port>` with the actual port number (6650, 6651, or 6652).

## Inspecting ETCD Metadata

1. **Set Up `etcdctl`**:

   Export the following environment variables:

   ```sh
   export ETCDCTL_API=3
   export ETCDCTL_ENDPOINTS=http://localhost:2379
   ```

2. **Inspect Metadata**:

   Use `etcdctl` commands to inspect metadata. For example, to list all keys:

   ```sh
   etcdctl get "" --prefix
   ```

   To get a specific key:

   ```sh
   etcdctl get <key>
   ```

## Makefile Targets Summary

- `make etcd`: Starts an ETCD instance in Docker.
- `make etcd-clean`: Stops and removes the ETCD instance and its data.
- `make brokers`: Builds and starts broker instances on predefined ports.
- `make brokers-clean`: Stops and removes all running broker instances.

## Troubleshooting

- **ETCD Not Starting**: Check Docker logs and ensure no other service is using port 2379.
- **Broker Not Starting**: Ensure ETCD is running and accessible at the specified address and port.
