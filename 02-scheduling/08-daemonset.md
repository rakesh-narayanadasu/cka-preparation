# DaemonSets

DaemonSets ensure that exactly one copy of a pod runs on every node in your Kubernetes cluster. When you add a new node, the DaemonSet automatically deploys the pod on the new node. Likewise, when a node is removed, the corresponding pod is also removed. This guarantees that a single instance of the pod is consistently available on each node.

### Use Cases for DaemonSets

DaemonSets are particularly useful in scenarios where you need to run background services or agents on every node. Some common use cases include:

- Monitoring agents and log collectors: Deploy monitoring tools or log collectors across every node to ensure comprehensive cluster-wide visibility without manual intervention.
- Essential Kubernetes components: Deploy critical components, such as kube-proxy, which Kubernetes requires on all worker nodes.
- Networking solutions: Ensure consistent deployment of networking agents like those used in VNet or weave-net across all nodes.

### Creating a DaemonSet

```
# daemon-set-definition.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
        - name: monitoring-agent
          image: monitoring-agent
```
Verify the DaemonSet's creation by running:
```
kubectl get daemonsets
```
For more detailed information on your DaemonSet, use:
```
kubectl describe daemonset monitoring-daemon
```

### How DaemonSets Schedule Pods

Prior to Kubernetes version 1.12, scheduling a pod on a specific node was often achieved by manually setting the `nodeName` property within the pod specification. However, since version 1.12, DaemonSets leverage the default scheduler in conjunction with **node affinity rules**. This improvement ensures that a pod is automatically scheduled on every node without manual intervention.

