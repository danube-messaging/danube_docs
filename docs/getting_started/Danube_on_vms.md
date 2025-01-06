# Danube Installation Guide

Danube is an open-source distributed Pub/Sub messaging platform written in Rust. It relies on ETCD for persistent metadata storage. This guide will walk you through installing Danube brokers and ETCD.

## Prerequisites

**ETCD**: Follow the [ETCD installation instructions](https://etcd.io/docs/v3.5/install/) to set up ETCD on a VM or bare metal server.

**Number of VMs**: Recommended 4 Linux machines or VMs:

* One or more VMs for running ETCD
* 3 VMs for running Danube brokers

## Installation Steps

### 1. Install ETCD

1. Follow the [ETCD installation guide](https://etcd.io/docs/v3.5/install/) to set up ETCD on a separate VM or server.
2. **Configuration**: Ensure ETCD listens on port `2379` for client requests.

### 2. Download Danube Broker

1. Visit the [Danube releases page](https://github.com/danube-messaging/danube/releases).
2. Download the latest binary for Linux named `danube-broker`.

### 3. Install Danube Brokers

1. **Upload the `danube-broker` binary** to each of the 3 VMs designated for brokers.

2. **Customize the Danube cluster config file**

   A sample file can be found [HERE](https://github.com/danube-messaging/danube/tree/main/config).

3. **Run the Broker**: Start each broker with the appropriate configuration.

   Example command to start a broker:

   ```bash
   RUST_LOG=danube_broker=info ./danube-broker --config-file config/danube_broker.yml
   ```

   Replace `ETCD_SERVER_IP` with the IP address of your ETCD server.

**Repeat** for each additional broker instance.

### 4. Verify the Setup

1. **ETCD**: Ensure ETCD is running and accessible. You can check its status by accessing `http://<ETCD_SERVER_IP>:2379` from a browser or using `curl`:

   ```bash
   curl http://<ETCD_SERVER_IP>:2379/v3/version
   ```

2. **Danube Brokers**: Ensure each broker instance is running and listening on the specified port. You can check this with `netstat` or `ss`:

   ```bash
   netstat -tuln | grep 6650
   ```

**Log Files**: For debugging, check the logs of each Danube broker instance.

## Additional Information

For more details, visit the [Danube GitHub repository](https://github.com/danube-messaging/danube) or contact the project maintainers for support.
