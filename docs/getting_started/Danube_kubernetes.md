# Run Danube messaging on Kubernetes

This documentation covers the instalation of the Danube cluster on the kubernetes. The Helm chart deploys the Danube Cluster with ETCD as metadata storage in the same namespace.

The documentation assumes that you have a Kubernetes cluster running and that you have installed the Helm package manager. For local testing you can use [kind](https://kind.sigs.k8s.io/).

## Install the Ngnix Ingress controller

Using the Official NGINX Ingress Helm Chart. This is required in order to route traffic to each broker service in the cluster. The Broker configuration is provisioned in the danube_helm, you can tweak the values.yaml per your needs.

The Danube messaging **has no dependency on ngnix**, can work with any ingress controller of your choice.

Install the NGINX Ingress Controller using Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Run the NGINX Ingress Controller on your local kubernetes cluster

For local testing the NGINX Ingress Controller can be exposed using a NodePort service so that the traffic from the local machine (outside the cluster) can reach the Ingress controller.

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.service.type=NodePort
```

You can find out which port is assigned by running

```bash
kubectl get svc

NAME                                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
kubernetes                                         ClusterIP   10.96.0.1       <none>        443/TCP                      4m17s
nginx-ingress-ingress-nginx-controller             NodePort    10.96.245.118   <none>        80:30115/TCP,443:30294/TCP   2m58s
nginx-ingress-ingress-nginx-controller-admission   ClusterIP   10.96.169.82    <none>        443/TCP                      2m58s
```

If ngnix is running as NodePort (usually for testing), you need local port in this case **30115**, in order to provide to danube_helm installation.

### Run the NGINX Ingress Controller in a remote cluster (cloud)

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true
```

- The publishService feature enables the Ingress controller to publish information about itself (such as its external IP or hostname) in a Kubernetes Service resource.
- This is particularly useful when you are running the Ingress controller in a cloud environment (like AWS, GCP, or Azure) and need it to publish its external IP address to handle incoming traffic

## Install Danube Messaging Brokers

First, add the repository to your Helm client:

```sh
helm repo add danube https://danube-messaging.github.io/danube_helm
helm repo update
```

You can install the chart with the release name `my-danube-cluster` using the below command. This will deploy the Danube Broker and an ETCD instance with the default configuration.

### Running on the local kubernetes cluster

```sh
helm install my-danube-cluster danube/danube-helm-chart --set broker.service.advertisedPort=30115
```

The advertisedPort is used to allow the client to reach the brokers, through the ingress NodePort.

### Running on the remote cluster (cloud)

The Danube cluster configuration from the values.yaml file has to be adjusted for your needs.

You can override the default values by providing a custom `values.yaml` file:

```sh
helm install my-danube-cluster danube/danube-helm-chart -f custom-values.yaml
```

Alternatively, you can specify individual values using the `--set` flag:

```sh
helm install my-danube-cluster danube/danube-helm-chart --set broker.service.type="ClusterIP"
```

You can further customize the installation, check the readme file. The default configuration is running 3 Danube Brokers in cluster.

## Check the install

Make sure that the brokers, etcd and the ngnix ingress are running properly in the cluster.

### Example running on local kubernetes cluster

```bash
kubectl get all

NAME                                                          READY   STATUS    RESTARTS   AGE
pod/my-danube-cluster-danube-broker1-766665d6f4-qdbf6         1/1     Running   0          12s
pod/my-danube-cluster-danube-broker2-5774ff4dd6-dvx66         1/1     Running   0          12s
pod/my-danube-cluster-danube-broker3-6db6b5fccd-dkr2k         1/1     Running   0          12s
pod/my-danube-cluster-etcd-867f5b85f8-g4m9m                   1/1     Running   0          12s
pod/nginx-ingress-ingress-nginx-controller-7bc7c7776d-wqc5g   1/1     Running   0          47m

NAME                                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                       AGE
service/kubernetes                                         ClusterIP   10.96.0.1       <none>        443/TCP                       48m
service/my-danube-cluster-danube-broker1                   ClusterIP   10.96.40.244    <none>        6650/TCP,50051/TCP,9040/TCP   12s
service/my-danube-cluster-danube-broker2                   ClusterIP   10.96.204.21    <none>        6650/TCP,50051/TCP,9040/TCP   12s
service/my-danube-cluster-danube-broker3                   ClusterIP   10.96.46.5      <none>        6650/TCP,50051/TCP,9040/TCP   12s
service/my-danube-cluster-etcd                             ClusterIP   10.96.232.70    <none>        2379/TCP                      12s
service/nginx-ingress-ingress-nginx-controller             NodePort    10.96.245.118   <none>        80:30115/TCP,443:30294/TCP    47m
service/nginx-ingress-ingress-nginx-controller-admission   ClusterIP   10.96.169.82    <none>        443/TCP                       47m

NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-danube-cluster-danube-broker1         1/1     1            1           12s
deployment.apps/my-danube-cluster-danube-broker2         1/1     1            1           12s
deployment.apps/my-danube-cluster-danube-broker3         1/1     1            1           12s
deployment.apps/my-danube-cluster-etcd                   1/1     1            1           12s
deployment.apps/nginx-ingress-ingress-nginx-controller   1/1     1            1           47m

NAME                                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/my-danube-cluster-danube-broker1-766665d6f4         1         1         1       12s
replicaset.apps/my-danube-cluster-danube-broker2-5774ff4dd6         1         1         1       12s
replicaset.apps/my-danube-cluster-danube-broker3-6db6b5fccd         1         1         1       12s
replicaset.apps/my-danube-cluster-etcd-867f5b85f8                   1         1         1       12s
replicaset.apps/nginx-ingress-ingress-nginx-controller-7bc7c7776d   1         1         1       47m
```

Validate that the brokers have started correctly:

```bash
kubectl logs pod/my-danube-cluster-danube-broker1-766665d6f4-qdbf6

initializing metrics exporter
2024-08-28T04:30:22.969462Z  INFO danube_broker: Use ETCD storage as metadata persistent store
2024-08-28T04:30:22.969598Z  INFO danube_broker: Start the Danube Service
2024-08-28T04:30:22.969612Z  INFO danube_broker::danube_service: Setting up the cluster MY_CLUSTER
2024-08-28T04:30:22.971978Z  INFO danube_broker::danube_service::local_cache: Initial cache populated
2024-08-28T04:30:22.972013Z  INFO danube_broker::danube_service: Started the Local Cache service.
2024-08-28T04:30:22.990763Z  INFO danube_broker::danube_service::broker_register: Broker 14150019297734190044 registered in the cluster
2024-08-28T04:30:22.991620Z  INFO danube_broker::danube_service: Namespace default already exists.
2024-08-28T04:30:22.991926Z  INFO danube_broker::danube_service: Namespace system already exists.
2024-08-28T04:30:22.992480Z  INFO danube_broker::danube_service: Namespace default already exists.
2024-08-28T04:30:22.992490Z  INFO danube_broker::danube_service: cluster metadata setup completed
2024-08-28T04:30:22.992551Z  INFO danube_broker::danube_service:  Started the Broker GRPC server
2024-08-28T04:30:22.992563Z  INFO danube_broker::broker_server: Server is listening on address: 0.0.0.0:6650
2024-08-28T04:30:22.992605Z  INFO danube_broker::danube_service: Started the Leader Election service
2024-08-28T04:30:22.993050Z  INFO danube_broker::danube_service: Started the Load Manager service.
2024-08-28T04:30:22.993143Z  INFO danube_broker::danube_service:  Started the Danube Admin GRPC server
2024-08-28T04:30:22.993274Z  INFO danube_broker::admin: Admin is listening on address: 0.0.0.0:50051
```

### Setup in order to communicate with cluster danube brokers

If you would like to communicate to the messaging sytem by using the danube-cli tool, or your own danube clients running locally, you can do the following:

```bash
kubectl get nodes -o wide
NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION       CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   53m   v1.30.0   172.20.0.2    <none>        Debian GNU/Linux 12 (bookworm)   5.15.0-118-generic   containerd://1.7.15
```

Use the **INTERNAL-IP** to route the traffic to broker hosts. Add the following in the hosts file, but make sure you match the number and the name of the brokers from the helm values.yaml file.

```bash
cat /etc/hosts
172.20.0.2 broker1.example.com broker2.example.com broker3.example.com
```

### Inspect the etcd instance

If you want to connect from your local machine, use kubectl port-forward to forward the etcd port to your local machine:

Port Forward etcd Service:

```bash
kubectl port-forward service/my-danube-cluster-etcd 2379:2379
```

Once port forwarding is set up, you can run etcdctl commands from your local machine:

```bash
etcdctl --endpoints=http://localhost:2379 watch --prefix /
```

## Cleanup

To uninstall the `my-danube-cluster` release:

```sh
helm uninstall my-danube-cluster
```

This command removes all the Kubernetes components associated with the chart and deletes the release.
