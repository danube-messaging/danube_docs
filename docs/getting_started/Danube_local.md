
# Run Danube Broker on your local machine

## Start Metadata Storage (ETCD)

Danube uses [ETCD](https://etcd.io/) for metadata storage to provide high availability and scalability. Run ETCD using Docker:

```bash
docker run -d --name etcd-danube -p 2379:2379 quay.io/coreos/etcd:latest etcd --advertise-client-urls http://0.0.0.0:2379 --listen-client-urls http://0.0.0.0:2379
```

Verify ETCD is running:

```bash
$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED          STATUS          PORTS                                                 NAMES
27792bce6077   quay.io/coreos/etcd:latest   "etcd --advertise-cl…"   35 seconds ago   Up 34 seconds   0.0.0.0:2379->2379/tcp, :::2379->2379/tcp, 2380/tcp   etcd-danube
```

## Configure and Run Danube Broker

### Create and configure broker config

Create a local config file, use the [sample config file](https://github.com/danube-messaging/danube/blob/main/config/danube_broker.yml) as a reference.

```bash
touch danube_broker.yml
```

### Download and run the Danube Broker

Download the latest binary from the [releases](https://github.com/danube-messaging/danube/releases) page.

If you would like to run Danube Brokers in cluster, you need to upload the binary to each machine and use the same cluster configuration name.

Run the Danube Broker:

```bash
touch broker.log
```

```bash
RUST_LOG=info ./danube-broker-linux --config-file danube_broker.yml --broker-addr "0.0.0.0:6650" --admin-addr "0.0.0.0:50051" > broker.log 2>&1 &
```

Check the logs:

```bash
tail -n 100 -f broker.log
```

```bash
2025-01-12T06:15:53.705416Z  INFO danube_broker: Use ETCD storage as metadata persistent store
2025-01-12T06:15:53.705665Z  INFO danube_broker: Start the Danube Service
2025-01-12T06:15:53.705679Z  INFO danube_broker::danube_service: Setting up the cluster MY_CLUSTER
2025-01-12T06:15:53.707988Z  INFO danube_broker::danube_service::local_cache: Initial cache populated
2025-01-12T06:15:53.709521Z  INFO danube_broker::danube_service: Started the Local Cache service.
2025-01-12T06:15:53.713329Z  INFO danube_broker::danube_service::broker_register: Broker 15139934490483381581 registered in the cluster
2025-01-12T06:15:53.714977Z  INFO danube_broker::danube_service: Namespace default already exists.
2025-01-12T06:15:53.716405Z  INFO danube_broker::danube_service: Namespace system already exists.
2025-01-12T06:15:53.717979Z  INFO danube_broker::danube_service: Namespace default already exists.
2025-01-12T06:15:53.718012Z  INFO danube_broker::danube_service: cluster metadata setup completed
2025-01-12T06:15:53.718092Z  INFO danube_broker::danube_service:  Started the Broker GRPC server
2025-01-12T06:15:53.718116Z  INFO danube_broker::broker_server: Server is listening on address: 0.0.0.0:6650
2025-01-12T06:15:53.718191Z  INFO danube_broker::danube_service: Started the Leader Election service
2025-01-12T06:15:53.722454Z  INFO danube_broker::danube_service: Started the Load Manager service.
2025-01-12T06:15:53.724727Z  INFO danube_broker::danube_service:  Started the Danube Admin GRPC server
2025-01-12T06:15:53.724727Z  INFO danube_broker::admin: Admin is listening on address: 0.0.0.0:50051
```

## Use Danube CLI to Publish and Consume Messages

Download the latest Danube CLI binary from the [releases](https://github.com/danube-messaging/danube/releases) page and run it:

```bash
./danube-cli-linux produce -s http://127.0.0.1:6650 -t /default/demo_topic -c 1000 -m "Hello, Danube!"
```

```bash
Message sent successfully with ID: 9
Message sent successfully with ID: 10
Message sent successfully with ID: 11
Message sent successfully with ID: 12
```

Open a new terminal and run the below command to consume the messages:

```bash
./danube-cli-linux consume -s http://127.0.0.1:6650 -t /default/demo_topic -m my_subscription
```

```bash
Received bytes message: 9, with payload: Hello, Danube!
Received bytes message: 10, with payload: Hello, Danube!
Received bytes message: 11, with payload: Hello, Danube!
Received bytes message: 12, with payload: Hello, Danube!
```

## Validate

Ensure ETCD is running and accessible. You can check its status by accessing `http://<ETCD_SERVER_IP>:2379` from a browser or using `curl`:

   ```bash
   curl http://<ETCD_SERVER_IP>:2379/v3/version
   ```

Ensure each broker instance is running and listening on the specified port. You can check this with `netstat` or `ss`:

   ```bash
   netstat -tuln | grep 6650
   ```

For debugging, check the logs of each Danube broker instance.

## Cleanup

Stop the Danube Broker:

```bash
pkill danube-broker
```

Stop and remove ETCD container

```bash
docker stop etcd-danube
```

```bash
docker rm -f etcd-danube
```

Verify cleanup

```bash
ps aux | grep danube-broker
docker ps | grep etcd-danube
```
