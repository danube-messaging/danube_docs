# Run Danube with docker-compose and MINIO as persistence storage

This guide provides instructions on how to run Danube Messaging Pub/Sub using Docker and Docker Compose. It sets up ETCD for metadata storage and MinIO for topic persistence storage.

The MINIO storage is used as an example but can be used with any [implemented storage layers](https://github.com/danube-messaging/danube-storage).

The Danube supports two dispatch strategies:

* **Non-reliable dispatch**: This strategy prioritizes speed and minimal resource usage by delivering messages directly from producers to subscribers without storing them. Messages flow through the broker in a "fire and forget" manner, achieving the lowest possible latency.
* **Reliable dispatch**: This strategy prioritizes message delivery and reliability, by implementing a store-and-forward mechanism.

This guide focus on running Danube with a reliable dispatch strategy and MINIO as persistence storage.

## Prerequisites

Ensure you have the following installed on your system:

* Docker
* Docker Compose (version v2.32.0 or higher)
* The files mentioned below *docker-compose.yml* and *danube_broker.yml* are in [this repository](https://github.com/danube-messaging/danube-storage/tree/main/danube-minio-storage/test_minio_storage).

## Docker Compose architecture

![Danube docker Compose](img/reliable_with_minio_storage.png "Danube Docker Compose")

## Docker Compose Configuration

### Key Components

* **ETCD (etcd)**: Stores Danube's metadata.
* **Danube Brokers (broker1, broker2)**: Handles message routing.
* **MinIO (minio)**: Provides object storage for persistent topics.
* **Danube MinIO Storage** (danube-minio-storage): Bridges Danube and MinIO.

The [docker-compose.yml](https://github.com/danube-messaging/danube-storage/blob/main/danube-minio-storage/test_minio_storage/docker-compose.yml) file defines the following services:

### ETCD (Metadata Storage)

ETCD is used for storing metadata.

```yaml
  etcd:
    image: quay.io/coreos/etcd:latest
    container_name: etcd-danube
    environment:
      ETCDCTL_API: 3
      ETCD_ADVERTISE_CLIENT_URLS: "http://etcd:2379"
      ETCD_LISTEN_CLIENT_URLS: "http://0.0.0.0:2379"
      ETCD_INITIAL_CLUSTER: "etcd-danube=http://etcd-danube:2380"
      ETCD_NAME: "etcd-danube"
      ETCD_LISTEN_PEER_URLS: "http://0.0.0.0:2380"
      ETCD_INITIAL_ADVERTISE_PEER_URLS: "http://etcd-danube:2380"
    ports:
      - "2379:2379"
      - "2380:2380"
    healthcheck:
      test: ["CMD", "etcdctl", "--endpoints=http://127.0.0.1:2379", "endpoint", "health"]
      interval: 2s
      timeout: 5s
      retries: 10
```

### Danube Brokers (Message Brokers)

Two brokers are defined (broker1 and broker2) that handle messaging. As the prerequisites, the etcd service must be healthy.
Ensure that your [`danube_broker.yml`](https://github.com/danube-messaging/danube-storage/blob/main/danube-minio-storage/test_minio_storage/danube_broker.yml) configuration file is correctly set up.

```yaml
  broker1:
    image: ghcr.io/danube-messaging/danube-broker:latest
    container_name: broker1
    depends_on:
      etcd:
        condition: service_healthy
    volumes:
      - ./danube_broker.yml:/etc/danube_broker.yml:ro
    environment:
      RUST_LOG: danube_broker=info
    command: [
      "--config-file", "/etc/danube_broker.yml",
      "--broker-addr", "0.0.0.0:6650",
      "--admin-addr", "0.0.0.0:50051",
      "--prom-exporter", "0.0.0.0:3000"
    ]
    ports:
      - "6650:6650"
      - "50051:50051"
      - "3000:3000"
```

The second broker (broker2) is configured similarly but runs on different ports.

### MinIO (Persistent Storage)

MinIO is used for message persistence. [Danube-minio-storage](https://github.com/danube-messaging/danube-storage/tree/main/danube-minio-storage) implements the storage grpc service for Danube and connects to MinIO.

```yaml
  minio:
    image: minio/minio
    container_name: minio
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  danube-minio-storage:
    build:
      context: ../.
      dockerfile: Dockerfile
    container_name: danube-minio-storage
    environment:
      GRPC_MINIO_PORT: 50060
      MINIO_ENDPOINT: minio:9000
      MINIO_ACCESS_KEY_ID: minioadmin
      MINIO_SECRET_ACCESS_KEY: minioadmin
      MINIO_BUCKET_NAME: danube-messages
      MINIO_LOCATION: us-east-1
    depends_on:
      minio:
        condition: service_healthy
    ports:
      - "50060:50060"

volumes:
  minio_data:
```

## Running Danube Messaging

1. Start the services using Docker Compose:

    ```bash
    docker-compose up --build

    ✔ danube-minio-storage                Built  
    ✔ Network test_minio_storage_default  Created 
    ✔ Container etcd-danube               Created 
    ✔ Container minio                     Created 
    ✔ Container broker2                   Created 
    ✔ Container broker1                   Created  
    ✔ Container danube-minio-storage      Created 
    ```

    This command will download the necessary images and start the containers. Docker Compose will ensure that ETCD and MinIO are healthy before starting the Danube brokers and storage component.

2. Verify running containers:

    ```bash
    docker ps
    ```

    This command will show you the running containers and their status.

    ```bash
    $ docker ps
    CONTAINER ID   IMAGE                                           COMMAND                  CREATED          STATUS                    PORTS                                                                                                                                           NAMES
    d791e51e7a29   test_minio_storage-danube-minio-storage         "./danube-minio-stor…"   35 seconds ago   Up 4 seconds              50051/tcp, 0.0.0.0:50060->50060/tcp, :::50060->50060/tcp                                                                                       danube-minio-storage
    
    ef4fe0528a57   ghcr.io/danube-messaging/danube-broker:latest   "/usr/local/bin/danu…"   35 seconds ago   Up 32 seconds             0.0.0.0:3000->3000/tcp, :::3000->3000/tcp, 0.0.0.0:6650->6650/tcp, :::6650->6650/tcp, 0.0.0.0:50051->50051/tcp, :::50051->50051/tcp, 6651/tcp   broker1
    
    ca1a41d1ef62   ghcr.io/danube-messaging/danube-broker:latest   "/usr/local/bin/danu…"   35 seconds ago   Up 32 seconds             0.0.0.0:3001->3001/tcp, :::3001->3001/tcp, 0.0.0.0:6651->6651/tcp, :::6651->6651/tcp, 0.0.0.0:50052->50052/tcp, :::50052->50052/tcp, 6650/tcp   broker2
    
    8a8c4a2e2d03   minio/minio                                     "/usr/bin/docker-ent…"   35 seconds ago   Up 34 seconds (healthy)   0.0.0.0:9000-9001->9000-9001/tcp, :::9000-9001->9000-9001/tcp                                                                                   minio
    
    b7cc1c6a2a03   quay.io/coreos/etcd:latest                      "/usr/local/bin/etcd"    35 seconds ago   Up 34 seconds (healthy)   0.0.0.0:2379-2380->2379-2380/tcp, :::2379-2380->2379-2380/tcp  
    ```

3. Produce messages:

    Download the [Danube CLI](https://github.com/danube-messaging/danube/releases).

    Create a 100KB blob file, in order to fill faster the 5 MB segment size.

    ```bash
    yes 'Danube messaging platform is awesome!' | head -c 100K > test.blob
    ```

    Produce messages, (creating also a topic with reliable dispatch, meaning that the messages are stored and then delivered to the consumers).

    ```bash
    danube-cli produce -s http://localhost:6650 -m "none" -f ./test.blob -c 1000 --reliable  --segment-size 5  --retention expire --retention-period 7200
    ```

4. Consume messages:

    Consume messages from the topic, creating an exclusive subscription.

    ```bash
    danube-cli consume -s http://localhost:6650 -m my_exclusive --sub-type exclusive
    ```

    The output should look like this:

    ```bash
    Received reliable message: [binary data] 
    Segment: 2, Offset: 41, Size: 102400 bytes, Total received: 9728000 bytes
    Producer: 9791760036514492028, Topic: /default/test_topic

    Received reliable message: [binary data] 
    Segment: 2, Offset: 42, Size: 102400 bytes, Total received: 9830400 bytes
    Producer: 9791760036514492028, Topic: /default/test_topic

    Received reliable message: [binary data] 
    Segment: 2, Offset: 43, Size: 102400 bytes, Total received: 9932800 bytes
    Producer: 9791760036514492028, Topic: /default/test_topic
    ```

## Checking Stored Messages

Open the MinIO console in your browser: `http://localhost:9001/buckets`. Log in with `minioadmin` as both the username and password. You can see the buckets and objects created by Danube.

## Stopping Danube Messaging

To stop and remove the containers:

```bash
$ docker-compose down

[+] Running 6/6
 ✔ Container danube-minio-storage      Removed         
 ✔ Container broker1                   Removed                  
 ✔ Container broker2                   Removed    
 ✔ Container minio                     Removed                                    
 ✔ Container etcd-danube               Removed  
 ✔ Network test_minio_storage_default  Removed   
```
