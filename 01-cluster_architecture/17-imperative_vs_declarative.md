# Imperative vs Declarative

### Imperative Approach

The imperative approach in Kubernetes involves executing specific commands to create, update, or delete objects. This method instructs Kubernetes on both what needs to be done and how it should be done. For example, you might run these commands:

```
kubectl run --image=nginx nginx
kubectl create deployment --image=nginx nginx
kubectl expose deployment nginx --port 80
kubectl edit deployment nginx
kubectl scale deployment nginx --replicas=5
kubectl set image deployment nginx nginx=nginx:1.18
kubectl create -f nginx.yaml
kubectl replace -f nginx.yaml
kubectl delete -f nginx.yaml
```

While the imperative approach is effective for quick tasks, it comes with some limitations:

- If a command partially executes, running it again may require extra checks (e.g., verifying if a resource already exists).
- Updating resources such as changing the image version—demands explicit re-execution with live adjustments.
- Commands executed interactively are often not persisted, making it challenging for teammates to trace the system’s original state.

In Kubernetes, imperative commands like `kubectl run`, `kubectl create deployment`, `kubectl expose`, and even editing commands such as kubectl edit or scaling commands are excellent for immediate changes but require careful tracking of the current state.

To list all resources
```
kubectl api-resources
```
```
kubectl explain pods
kubectl explain pods.spec
kubectl explain pods --recursive
```

#### POD

Create an NGINX Pod
```
kubectl run nginx --image=nginx
```
Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)
```
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

#### Deployment

Create a deployment
```
kubectl create deployment --image=nginx nginx
```
Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)
```
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```

Generate Deployment with 4 Replicas
```
kubectl create deployment nginx --image=nginx --replicas=4
```

You can also scale a deployment using the kubectl scale command.
```
kubectl scale deployment nginx --replicas=4
```

Another way to do this is to save the YAML definition to a file and modify
```
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

#### Service

Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379
```
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```
Or
```
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```

Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:
```
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml
```
Or
```
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

#### Reference:

https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

https://kubernetes.io/docs/reference/kubectl/conventions/


### Declarative Approach

The declarative approach enables you to specify the desired state of your infrastructure through configuration files (typically written in YAML).

Kubernetes will create or update the object automatically to match the state described in your YAML file. When you need to update the configuration—say, to change the image version—you modify the YAML file and apply it again.

This method ensures that your configuration files remain the single source of truth, which is especially valuable in team environments where version-controlled definitions are critical.

- Imperative Approach:
Use this method for rapid execution when you need to quickly create or modify Kubernetes objects, particularly during certification exams.

- Declarative Approach:
This approach is recommended for complex, long-term management scenarios. It enables a systematic management of configurations via YAML files, ensuring every change is recorded and version-controlled.


#### How kubectl apply Works Internally

When you run the kubectl apply command, it compares three sources:

1) The local configuration file (e.g., nginx.yaml).
2) The live object configuration stored on the Kubernetes cluster.
3) The last applied configuration stored as an annotation on the live object.

If the object does not exist, Kubernetes creates it based on your local configuration. During creation, Kubernetes internally adds additional fields to monitor the object's status. The YAML configuration is converted to JSON and stored as the "last applied configuration" in an annotation. This information is used during subsequent updates to identify any differences.

When kubectl apply is executed for the first time, the YAML configuration is converted to JSON and stored as an annotation under the key `kubectl.kubernetes.io/last-applied-configuration`. 

