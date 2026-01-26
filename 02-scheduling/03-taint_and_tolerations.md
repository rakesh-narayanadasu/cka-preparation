# Taints and Tolerations

In Kubernetes, a **taint is applied to a node** (the repellent) to repel pods unless they have a matching toleration (an immunity to the repellent). **This ensures that only intended pods can run on tainted nodes**.

### How Taints and Tolerations Work

Two factors determine whether a pod can run on a node:

1) The taint applied on the node.
2) The pod's toleration for that taint.

In Kubernetes, taints and tolerations are used solely for scheduling control, not security. Consider a simple cluster with three worker nodes (node one, two, and three) and four pods (A, B, C, and D). By default, without any taints or tolerations, the scheduler distributes the pods evenly across all nodes.

Suppose you want to dedicate node one to a specific application:

- First, taint node one (using, for example, the key-value pair "app=blue") so that no pods are scheduled there by default.
- Then, add an appropriate toleration to the dedicated pod (Pod D) so that it alone can run on node one.

When the scheduler assigns pods:

- Pod A, lacking a toleration, will be scheduled on node two.
- Pod B is repelled from node one and lands on node three.
- Pod C, without a matching toleration, is scheduled on node two or three.
- Pod D, having the correct toleration, is scheduled on node one.

### Tainting a Node

To taint a node, use the following command where you specify the node name along with a taint in the format of a key-value pair and taint effect:
```
kubectl taint nodes node-name key=value:taint-effect
```

For example, to taint node one so that it only accepts pods associated with the application blue, run:

```
kubectl taint nodes node1 app=blue:NoSchedule
```
To untaint the node use `-` at the end of the command
```
kubectl taint nodes node1 app=blue:NoSchedule-
```

```
kubectl describe node node01 | grep -i taints
```

There are three taint effects:

- **NoSchedule**: Pods that do not tolerate the taint will not be scheduled on the node.
- **PreferNoSchedule**: The scheduler tries to avoid placing non-tolerating pods on the node, but it is not strictly enforced.
- **NoExecute**: New pods without a matching toleration will not be scheduled, and existing pods will be evicted from the node.

### Adding Tolerations to Pods

To allow a specific pod to run on a tainted node, add a toleration to the pod’s manifest. For example, to enable Pod D to run on node one (tainted with "app=blue:NoSchedule"), modify the pod definition as follows:

```
kubectl taint nodes node1 app=blue:NoSchedule
```

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
```

>Note: Taints and tolerations are used to ensure that pods are not accidentally scheduled on nodes that should run only specific workloads, but **they do not guarantee that a pod will run on a particular node**. For node-specific scheduling, consider using node affinity.

#### Master Node Taints

While the discussion so far has focused on worker nodes, master nodes are also capable of running pods. By default, Kubernetes applies a taint to master nodes to prevent workload pods from being scheduled there. This setup ensures that master nodes are reserved for critical system components. To check the taint on a master node, use the following command:
```
kubectl describe node kubemaster | grep Taint
```
The output will display a taint similar to:
```
Taints:             node-role.kubernetes.io/master:NoSchedule
```
This configuration follows best practices by ensuring that only essential components run on the master node.


| Feature | Description | Example Command or Manifest Snippet |
|-------|-------------|-------------------------------------|
| Tainting a Node | Prevents pods from being scheduled on a node unless they tolerate the taint. | `kubectl taint nodes node1 app=blue:NoSchedule` |
| Toleration in a Pod | Allows a pod to run on a tainted node by including a toleration in its manifest. | ```See the YAML snippet above``` |
| Taint Effect: NoSchedule | Blocks new pods from being scheduled on the node unless they tolerate the taint. **Existing pods are not affected**. | `kubectl taint nodes node1 app=blue:NoSchedule` |
| Taint Effect: PreferNoSchedule | Kubernetes will **try** to avoid scheduling pods on the node unless they tolerate the taint, but it is not guaranteed. | `kubectl taint nodes node1 app=blue:PreferNoSchedule` |
| Taint Effect: NoExecute | **Evicts existing pods** that do not tolerate the taint and prevents new non-tolerant pods from being scheduled. | `kubectl taint nodes node1 app=blue:NoExecute` |

**Remember that taints and tolerations only influence pod scheduling. If you need to enforce pod placement on specific nodes, use node affinity rules.**
