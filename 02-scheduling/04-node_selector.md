# Node Selectors

Node selector ensure that specific pods run only on designated nodes within your cluster. Node selectors help align your pod deployments with the underlying hardware characteristics of your nodes, enhancing performance and resource management.

Imagine managing a three-node cluster where two nodes have limited hardware resources, and one node is equipped with higher resources. Different workloads run in your cluster, and data processing tasks that demand more computing power should ideally be scheduled on the larger node. Without any scheduling constraints, any pod might land on any node even on one with insufficient resources leading to performance bottlenecks.

**Node selectors restrict pod placement by matching key-value pairs defined in the pod’s specification against the labels on the nodes.**

### Configuring Node Selectors

To ensure that a pod is restricted to run on a specific node, you can modify the pod's definition file using node selectors. Below is an example of a pod definition YAML file that deploys a data processing image exclusively on a node labeled as "Large":

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: data-processor
      image: data-processor
  nodeSelector:
    size: Large
```

In this configuration, the Kubernetes scheduler identifies the appropriate node by matching the label with the key-value pair `size: Large`. Ensure that you pre-label the target node accordingly.

### Labeling a Node

Before deploying your pod, you must label your node so that it can be recognized by the selector. Use the following command to label a node (for example, `node-1`) as "Large":

```
kubectl label nodes node-1 size=Large
```

### Limitations and Advanced Scheduling

While node selectors are ideal for simple scenarios involving a single label, they come with certain limitations. For instance, if you need to schedule a pod on a node that is either large or medium, or on any node that is not labeled as small, a basic node selector may not suffice. In these cases, consider using node affinity and anti-affinity features, which offer advanced scheduling capabilities to define more complex placement rules.
