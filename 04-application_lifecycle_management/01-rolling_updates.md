# Rolling Updates and Rollbacks

### Understanding Rollouts and Versioning

When you create a deployment, Kubernetes initiates a rollout that establishes the first deployment revision (revision one). Later, when you update your application say by changing the container image version Kubernetes triggers another rollout, creating a new revision (revision two). These revisions help you track changes and enable rollbacks to previous versions if issues arise.

Check the rollout status:
```
kubectl rollout status deployment/myapp-deployment
```

View the history of rollouts:
```
kubectl rollout history deployment/myapp-deployment
```

If an issue is detected after an upgrade, you can revert to the previous version using the rollback feature. To perform a rollback, run:
```
kubectl rollout undo deployment/myapp-deployment
```

Verify the state of ReplicaSets before and after a rollback with:
```
kubectl get replicasets
```

### Deployment Strategies

There are different strategies to update your applications. For example, consider a scenario where your web application is running five replicas.

One approach is the "recreate" strategy, which involves shutting down all existing instances before deploying new ones. However, this method results in temporary downtime as the application becomes inaccessible during the update.

A more seamless approach is the "rolling update" strategy. Here, instances are updated one at a time, ensuring continuous application availability throughout the process.

**If no strategy is specified when creating a deployment, Kubernetes uses the rolling update strategy by default.**

### Viewing Deployment Details

To retrieve detailed information about your deployment—including rollout strategy, scaling events, and more use:

```
kubectl describe deployment myapp-deployment
```

This output shows different details depending on the strategy used:

- Recreate Strategy: Events indicate that the old ReplicaSet is scaled down to zero before scaling up the new ReplicaSet.
- Rolling Update Strategy: The old ReplicaSet is gradually scaled down while the new ReplicaSet scales up.

For example, a deployment with the recreate strategy might display the following events:

```
Name:                   myapp-deployment
Namespace:              default
CreationTimestamp:      Sat, 03 Mar 2018 17:01:55 +0000
Labels:                 app=myapp
Annotations:            deployment.kubernetes.io/revision=2
                        kubectl.kubernetes.io/change-cause=kubectl apply --filename=deployment-definition.yml
Selector:               5 desired, 1 updated, 5 total, 5 available, 0 unavailable
StrategyType:           Recreate
MinReadySeconds:        0
Pod Template:
  Labels:  app=myapp
           type=front-end
  Containers:
   nginx-container:
    Image:      nginx:1.7.1
    Port:       <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   myapp-deployment-54c7d6ccc (5/5 replicas created)
Events:  
  Type    Reason             Age   From                    Message
  -----   ------             ----  ----                    -------
  Normal  ScalingReplicaSet  11m   deployment-controller  Scaled up replica set myapp-deployment-6795844b58 to 5
  Normal  ScalingReplicaSet  11m   deployment-controller  Scaled down replica set myapp-deployment-6795844b58 to 0
  Normal  ScalingReplicaSet  56s   deployment-controller  Scaled up replica set myapp-deployment-54c7d6ccc to 5
```

In contrast, a rolling update strategy output would reflect gradual scaling changes:

```
Name:                   myapp-deployment
Namespace:              default
CreationTimestamp:      Sat, 03 Mar 2018 17:16:53 +0800
Labels:                 app=myapp
Annotations:            deployment.kubernetes.io/revision=2
                        kubectl.kubernetes.io/change-cause=kubectl apply --filename=deployment-definition.yml
Selector:               6 desired, 5 updated, 6 total, 4 available, 2 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=myapp
           type=front-end
  Containers:
   nginx-container:
    Image:      nginx
    Port:       <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:   myapp-deployment-67c749c58c (1/1 replicas created)
NewReplicaSet:    myapp-deployment-75d7bdbd8d (5/5 replicas created)
Events:  
  Type    Reason             Age   From                    Message
  -----   ------             ----  ----                    -------
  Normal  ScalingReplicaSet  1m    deployment-controller   Scaled up replica set myapp-deployment-67c749c58c to 5
  Normal  ScalingReplicaSet  1m    deployment-controller   Scaled down replica set myapp-deployment-75d7bdbd8d to 2
  Normal  ScalingReplicaSet  1m    deployment-controller   Scaled up replica set myapp-deployment-67c749c58c to 4
  Normal  ScalingReplicaSet  1m    deployment-controller   Scaled down replica set myapp-deployment-75d7bdbd8d to 3
  Normal  ScalingReplicaSet  0s    deployment-controller   Scaled down replica set myapp-deployment-75d7bdbd8d to 1
  Normal  ScalingReplicaSet  0s    deployment-controller   Scaled down replica set myapp-deployment-67c749c58c to 0
```
