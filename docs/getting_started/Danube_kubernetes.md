# Run Danube on Kubernetes

This guide walks you through deploying a Danube messaging cluster on Kubernetes
and verifying it with a producer and consumer. It uses a local
[Kind](https://kind.sigs.k8s.io/) cluster for simplicity, but the same Helm
charts work on any Kubernetes cluster (EKS, GKE, AKS, etc.).

## Overview

A Danube deployment consists of two Helm charts:

| Chart | What it deploys |
|-------|----------------|
| **danube-envoy** | An [Envoy](https://www.envoyproxy.io/) gRPC proxy that routes client requests to the correct broker |
| **danube-core** | Danube brokers (StatefulSet), etcd for metadata, and Prometheus for metrics |

The proxy is installed first because the brokers need to know its external
address at startup. This address is called the **`connectUrl`** — it tells
clients how to reach the cluster from outside Kubernetes.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) (for local testing)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm 3.0+](https://helm.sh/docs/intro/install/)
- [danube-cli](https://github.com/danube-messaging/danube/releases) (for testing)

## Step 0: Prepare a Kubernetes cluster

If you already have a cluster, skip to Step 1.

Create a local Kind cluster:

```bash
kind create cluster
kubectl cluster-info --context kind-kind
```

## Step 1: Add the Danube Helm repository

```bash
helm repo add danube https://danrusei.github.io/danube_helm
helm repo update
```

This makes two charts available: `danube/danube-envoy` and `danube/danube-core`.

## Step 2: Install the Envoy proxy

The Envoy proxy is the single entry point for all client traffic. It handles
gRPC routing so that each request reaches the broker that owns the target topic.

```bash
kubectl create namespace danube
helm install danube-envoy danube/danube-envoy -n danube
```

Wait for the proxy pod to be ready:

```bash
kubectl get pods -n danube -w
```

You should see:

```bash
NAME                           READY   STATUS    AGE
danube-envoy-xxxxxxxxx         1/1     Running   30s
```

## Step 3: Discover the proxy address

The proxy service is exposed as a Kubernetes
[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport).
You need the node IP and the assigned port to construct the external address:

```bash
PROXY_PORT=$(kubectl get svc danube-envoy -n danube \
  -o jsonpath='{.spec.ports[?(@.name=="grpc")].nodePort}')
NODE_IP=$(kubectl get nodes \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Proxy address: ${NODE_IP}:${PROXY_PORT}"
```

Save this address — you will use it in the next step and when connecting clients.

> **Cloud clusters**: If you are running on a managed Kubernetes service, you can
> change the proxy service type to `LoadBalancer` in the danube-envoy values and
> use the external IP instead of `NodePort`.

## Step 4: Install Danube core

Danube brokers read their configuration from a Kubernetes ConfigMap. Create it
from the example config file, then install the chart with the proxy address:

```bash
kubectl create configmap danube-broker-config \
  --from-file=danube_broker.yml=danube_broker.yml \
  -n danube
```

```bash
helm install danube-core danube/danube-core -n danube \
  -f values-minimal.yaml \
  --set broker.externalAccess.connectUrl="${NODE_IP}:${PROXY_PORT}"
```

> **Note**: The `danube_broker.yml` config file and `values-minimal.yaml` values
> file are available in the
> [danube_helm repository](https://github.com/danube-messaging/danube_helm/tree/main/charts/danube-core/examples).
> Download them or clone the repository to get started.

This deploys:

- **3 broker pods** (StatefulSet) — the messaging engine, with persistent storage
- **1 etcd pod** — metadata store for topic assignments and cluster state
- **1 Prometheus pod** — metrics collection

The `connectUrl` parameter tells each broker to advertise the Envoy proxy as
the client-facing address. This enables **proxy mode**: clients connect to the
proxy, which routes each request to the correct broker based on topic ownership.

Wait for all pods to be ready:

```bash
kubectl get pods -n danube -w
```

Expected output:

```bash
NAME                                      READY   STATUS    AGE
danube-core-broker-0                      1/1     Running   2m
danube-core-broker-1                      1/1     Running   2m
danube-core-broker-2                      1/1     Running   2m
danube-core-etcd-0                        1/1     Running   3m
danube-core-prometheus-xxxxxxxxx          1/1     Running   3m
danube-envoy-xxxxxxxxx                    1/1     Running   5m
```

> **Tip**: The first broker may restart a few times while waiting for etcd to
> become ready. This is normal — Kubernetes will keep restarting it until etcd
> accepts connections.

## Step 5: Verify the deployment

Check that the brokers registered in proxy mode:

```bash
kubectl logs danube-core-broker-0 -n danube | grep "broker registered"
```

You should see each broker reporting a unique `broker_url` (its internal DNS
name) and a shared `connect_url` (the Envoy proxy address):

```bash
broker registered broker_url=http://danube-core-broker-0.danube-core-broker-headless.danube.svc.cluster.local:6650 connect_url=http://172.19.0.2:30700
```

This confirms proxy mode is active. The broker knows its internal identity
(`broker_url`) and advertises the proxy (`connect_url`) to all clients.

## Step 6: Produce and consume messages

Use `danube-cli` to send messages through the proxy.

**Terminal 1 — Produce 5 messages:**

```bash
danube-cli produce \
  -s http://${NODE_IP}:${PROXY_PORT} \
  -t /default/test_topic \
  -m "Hello from Danube" -c 5
```

Expected output:

```bash
✅ Producer 'test_producer' created successfully
📤 Message 1/5 sent successfully (ID: 2)
📤 Message 2/5 sent successfully (ID: 3)
📤 Message 3/5 sent successfully (ID: 4)
📤 Message 4/5 sent successfully (ID: 5)
📤 Message 5/5 sent successfully (ID: 6)

📊 Summary:
   ✅ Success: 5
```

**Terminal 2 — Start a consumer:**

```bash
danube-cli consume \
  -s http://${NODE_IP}:${PROXY_PORT} \
  -t /default/test_topic \
  -m test_sub
```

Now produce more messages in Terminal 1 — the consumer receives them in real
time:

```bash
Received message: Hello from Danube
Size: 17 bytes, Total received: 17 bytes
```

## How proxy mode works

In a multi-broker cluster, each topic is owned by a specific broker. When a
client connects, it needs to reach the right broker for its topic.

1. The client connects to the **Envoy proxy** (the `connectUrl`).
2. It sends a **topic lookup** request, which Envoy round-robins to any broker.
3. The broker responds with its internal address (`broker_url`) for that topic.
4. On all subsequent requests, the client includes an `x-danube-broker-url`
   gRPC metadata header with the target broker's internal address.
5. Envoy's **Dynamic Forward Proxy** reads this header, resolves the broker's
   internal DNS name, and routes the request directly to the correct pod.

This means clients only need a single external address (the proxy) regardless
of how many brokers are in the cluster.

## Inspect etcd (optional)

To browse the cluster metadata, forward the etcd port to your local machine:

```bash
kubectl port-forward svc/danube-core-etcd 2379:2379 -n danube
etcdctl --endpoints=http://localhost:2379 get --prefix /
```

## Access Prometheus (optional)

```bash
kubectl port-forward svc/danube-core-prometheus 9090:9090 -n danube
```

Open `http://localhost:9090` in your browser to query Danube metrics.

## Cleanup

```bash
helm uninstall danube-core -n danube
helm uninstall danube-envoy -n danube
kubectl delete namespace danube
kind delete cluster
```

This removes all Kubernetes resources including PersistentVolumeClaims (since
the namespace is deleted).
