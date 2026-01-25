# Pods

A pod represents a single instance of an application and is the smallest deployable unit in Kubernetes.

Typically, each pod hosts a single container running your main application. However, a pod can also contain multiple containers, which are usually complementary rather than redundant. For example, you might include a helper container alongside your main application container to support tasks like data processing or file uploads. Both containers in the pod share the same network namespace (allowing direct communication via localhost), storage volumes, and lifecycle events, ensuring they start and stop together.

When a pod is defined with multiple containers, they share storage, the network namespace, and lifecycle events ensuring seamless coordination and simplifying management.

#### Deploying Pods

A common method to deploy pods is using the kubectl run command. For example, the following command creates a pod that deploys an instance of the nginx Docker image, pulling it from a Docker repository:
```
kubectl run nginx --image nginx
```
Once deployed, you can verify the pod's status with the `kubectl get pods` command. Initially, the pod might be in a "ContainerCreating" state, followed by a transition to the "Running" state as the application container becomes active. Below is an example session:
```
kubectl get pods
```


# Pods with YAML

Kubernetes leverages YAML files to define objects such as Pods, ReplicaSets, Deployments, and Services. These definitions adhere to a consistent structure, with four essential top-level properties: apiVersion, kind, metadata, and spec.

Every Kubernetes definition file must include the following four fields:
```
apiVersion:
kind:
metadata:
spec:
```

**1) apiVersion**

This field indicates the version of the Kubernetes API you are using. For a Pod, set `apiVersion: v1`. Depending on the object you define, you might need different versions such as apps/v1, extensions/v1beta1, etc.

**2) kind**

This specifies the type of object being created. In this lesson, since we're creating a Pod, you'll define it as `kind: Pod`. Other objects might include ReplicaSet, Deployment, or Service.

**3) metadata**

The metadata section provides details about the object, including its `name` and `labels`. It is represented as a **dictionary**. It is essential to maintain consistent indentation for sibling keys to ensure proper YAML nesting. For example:
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
```

**4) spec**

The spec section provides specific configuration details for the object. For a Pod, this is where you define its containers. Since a Pod can run multiple containers, the `containers` field is an **array**. In our example, with a single container, the array has just one item. The dash (`-`) indicates a list item, and each container must be defined with at least `name` and `image` keys.

Below is the complete YAML configuration for our Pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```
#### Creating and Verifying the Pod

After you have saved your configuration (for example, as `pod-definition.yaml`), use the following command to create the Pod:
```
kubectl create -f pod-definition.yaml
```
Once the Pod is created, you can verify its status by listing all Pods:
```
kubectl get pods
```
You should see output similar to this:
```
NAME         READY   STATUS    RESTARTS   AGE
myapp-pod    1/1     Running   0          20s
```
To view detailed information about the Pod, run:
```
kubectl describe pod myapp-pod
```

`Using kubectl describe helps you gain detailed insights into the internal state of your Pod, which can be invaluable for troubleshooting.`
