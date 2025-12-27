# Kube Controller Manager

The Kube Controller Manager, a vital component in Kubernetes responsible for managing a variety of controllers within your cluster. For instance, one controller might monitor the health of nodes, while another ensures that the desired number of pods is always running. These controllers constantly observe system changes to drive the cluster toward its intended state.

The **Node Controller**, for example, checks node statuses every five seconds through the Kube API Server. If a node stops sending heartbeats, it is not immediately marked as unreachable; instead, there is a grace period of 40 seconds followed by an additional five minutes for potential recovery before its pods are rescheduled onto a healthy node.

- node monitor period: 5s
- node monitor grace period 40s
- pod eviction timeout 5m0s

Checking Node Statuses
```
kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
worker-1     Ready    <none>   8d    v1.13.0
worker-2     Ready    <none>   8d    v1.13.0
```
In the case where a node fails to recover, the output might look like this:
```
kubectl get nodes
NAME         STATUS     ROLES    AGE   VERSION
worker-1     Ready      <none>   8d    v1.13.0
worker-2     NotReady   <none>   8d    v1.13.0
```

Another essential controller is the **Replication Controller**, which ensures that the specified number of pods is maintained by creating new pods when needed. This mechanism reinforces the resilience and reliability of your Kubernetes cluster.

All core Kubernetes constructs such as Deployments, Services, Namespaces, and Persistent Volumes rely on these controllers. Essentially, controllers serve as the "brains" behind many operations in a Kubernetes cluster.

### How Controllers Are Packaged

All individual controllers are bundled into a single process known as the Kubernetes Controller Manager. When you deploy the Controller Manager, every associated controller is started together. This unified deployment simplifies management and configuration.

### Installing and Configuring the Kube Controller Manager
To install and view the Kube Controller Manager, follow these steps:

- Download the Kube Controller Manager from the Kubernetes release page.
- Extract the binary and run it as a service.
- Review the configurable options provided, which allow you to tailor its behavior.

#### Downloading the Controller Manager
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-controller-manager
```
#### Sample Service Configuration
Below is an example of a service file `kube-controller-manager.service` used to run the Controller Manager:
```
ExecStart=/usr/local/bin/kube-controller-manager \
    --address=0.0.0.0 \
    --cluster-cidr=10.200.0.0/16 \
    --cluster-name=kubernetes \
    --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
    --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
    --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
    --leader-elect=true \
    --root-ca-file=/var/lib/kubernetes/ca.pem \
    --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
    --service-cluster-ip-range=10.32.0.0/24 \
    --use-service-account-credentials=true \
    --v=2
    --node-monitor-period=5s
    --node-monitor-grace-period=40s
    --pod-eviction-timeout=5m0s
```    
This configuration includes additional options for the Node Controller, such as node monitor period, grace period, and eviction timeout. Additionally, you can control which controllers are enabled through the --controllers flag.

