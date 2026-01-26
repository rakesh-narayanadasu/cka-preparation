# Taints and Tolerations vs Node Affinity

In Kubernetes, taints and tolerations are primarily used to repel pods from nodes unless they explicitly tolerate the taint, whereas node affinity is used to attract pods to nodes that satisfy specific label criteria.

### Combining Taints and Tolerations with Node Affinity

For exclusive node usage, combining both strategies is the optimal solution. The integration works as follows:

1) Apply taints on nodes and specify corresponding tolerations in pod configurations to block any pod without the proper toleration.
2) Use node affinity rules to ensure that each pod is only scheduled on a node with a matching label.

This combined approach dedicates the nodes exclusively to the intended pods, assuring correct pod assignments and preventing interference by other workloads.

```
kubectl taint nodes prod-node-1 env=prod:NoSchedule
kubectl label nodes prod-node-1 env=prod
```

```
spec:
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: env
            operator: In
            values:
            - prod
```

**Use Taints and Tolerations to prevent other pods from being placed on our nodes (but our pods may end up in other nodes) and use Node Affinity to prevent our pods from being place on other nodes (but other pods may end up in our nodes).**
