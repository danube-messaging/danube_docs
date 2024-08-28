# Danube Cluster Instalation Guide on Kubernetes with Helm Chart

The Helm chart deploys the Danube Cluster with ETCD as metadata storage in the same namespace.

If you would like to configure for testing purposes you may want to see: [Run Danube on Kubernetes on Local Machine](https://dev-state.com/danube_docs/getting_started/Danube_local_machine_k8s/).

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+

## Installation

### Install NGNIX Ingress Controller

This is required in order to route traffic to each broker service in the cluster. The configuration of the controller is already provided into the danube_helm, just need to tweak the values.yaml per your needs.

You can install the NGINX Ingress Controller using Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

You can expose the NGINX Ingress controller using a NodePort service so that traffic from the local machine (outside the cluster) can reach the Ingress controller.

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true
```

- The publishService feature enables the Ingress controller to publish information about itself (such as its external IP or hostname) in a Kubernetes Service resource.
- This is particularly useful when you are running the Ingress controller in a cloud environment (like AWS, GCP, or Azure) and need it to publish its external IP address to handle incoming traffic

The Danube is not dependent on ngnix, can work with any ingress controller of your choice.

### Add Danube Helm Repository

First, add the repository to your Helm client:

```sh
helm repo add danube https://danrusei.github.io/danube_helm
helm repo update
```

### Install the Danube Helm Chart

You can install the chart with the release name `my-danube-cluster` using the following command:

```sh
helm install my-danube-cluster danube/danube-helm-chart
```

This will deploy the Danube Broker and an ETCD instance with the default configuration.

## Configuration

The Danube cluster configuration from the values.yaml file has to be adjusted for your needs.

You can override the default values by providing a custom `values.yaml` file:

```sh
helm install my-danube-cluster danube/danube-helm-chart -f custom-values.yaml
```

Alternatively, you can specify individual values using the `--set` flag:

```sh
helm install my-danube-cluster danube/danube-helm-chart --set broker.service.type="ClusterIP"
```

## Resource consideration

The default configuration is just OK for testing, but you need to reconsider the values for production env.

### Sizing for Production

**Small to Medium Load**:

- CPU Requests: 500m to 1 CPU
- CPU Limits: 1 CPU to 2 CPUs
- Memory Requests: 512Mi to 1Gi
- Memory Limits: 1Gi to 2Gi

**Heavy Load:**

- CPU Requests: 1 CPU to 2 CPUs
- CPU Limits: 2 CPUs to 4 CPUs
- Memory Requests: 1Gi to 2Gi
- Memory Limits: 2Gi to 4Gi

## Uninstallation

To uninstall the `my-danube-cluster` release:

```sh
helm uninstall my-danube-cluster
```

This command removes all the Kubernetes components associated with the chart and deletes the release.

## Troubleshooting

To get the status of the ETCD and Broker pods:

```sh
kubectl get all
```

To view the logs of a specific broker pod:

```sh
kubectl logs <broker-pod-name>
```
