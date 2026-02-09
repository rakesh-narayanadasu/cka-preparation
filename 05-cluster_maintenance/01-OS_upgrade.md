# OS Upgrades

Imagine a cluster with several nodes where each node runs pods serving your applications. What happens if a node goes down? The pods on that node become inaccessible, and depending on your deployment strategy, this could impact your users. 

Kubernetes offers mechanisms to handle such situations:

- If a node comes back online quickly, the pods are simply restarted.
- If a node remains down for more than five minutes, Kubernetes marks the pods on that node as dead. For pods managed by a ReplicaSet, new pods are automatically created on other nodes. In contrast, **pods that are not part of a ReplicaSet will not be restarted, potentially leading to downtime**.

## Draining a Node

For situations where the recovery time is uncertain, the safer method is to drain the node. Draining involves gracefully terminating the pods running on that node so they are recreated on other nodes. At the same time, draining marks the node as unschedulable (cordoned), preventing new pods from being scheduled on it until explicitly allowed.

After the node’s workloads have been safely relocated, you can reboot the node. When it comes back online, it remains unschedulable until you uncordon it, allowing new pods to be scheduled. It is important to note that pods moved to other nodes do not automatically revert to the original node after uncordoning. If they were deleted or if new pods have been scheduled across the cluster, Kubernetes maintains the current distribution.

To perform these operations, run the following commands:
```
kubectl drain node-1
kubectl uncordon node-1
```

### Drain vs Cordon

In addition to draining and uncordoning, Kubernetes provides the cordon command. Unlike drain, **cordon only marks a node as unschedulable without terminating or relocating the currently running pods**. This ensures that no new pods will be scheduled on the node.

```
kubectl cordon node-1
```

> Be cautious when using cordon on a node with critical workloads. Since existing pods remain and new pods cannot be scheduled, your application might become overloaded or experience unexpected behavior.

