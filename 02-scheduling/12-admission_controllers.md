# Admission Controllers

An admission controller is a piece of logic that runs after a request is authenticated and authorized, but before the object is actually stored in etcd. They can inspect, modify, or reject API requests for Kubernetes objects like Pods, Deployments, Services, etc.

kubectl apply → Authentication → Authorization (RBAC) → Admission Controllers → etcd

### Authentication and Authorization Flow

1) Authentication:
The API server authenticates the request. For example, when you run a command via kubectl, the KubeConfig file provides the necessary certificates. Consider the following configuration:

```
> cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tCIlRjdG......BQyMjAwFURs9tCg==
    server: https://example.com
  name: example-cluster
```

2) Authorization (RBAC):
Once authenticated, the API server checks if the user is authorized to perform the requested action. Role-based Access Control (RBAC) is commonly used here. For instance, if a user is assigned the following role:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
```

The user is permitted to list, get, create, update, or delete pods. RBAC can also restrict access to specific resource names or namespaces:

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create"]
  resourceNames: ["blue", "orange"]
```

**These RBAC rules are enforced at the API level and determine which API operations a user can access.**

## The Role of Admission Controllers

While RBAC handles basic authorization, it does not offer advanced validations or mutations. Admission controllers step in to provide additional security by:

- Validating pod specifications (e.g., ensuring that images are not from a public Docker Hub registry or enforcing the prohibition of the "latest" tag).
- Rejecting pods running containers as the root user, or enforcing specific Linux capabilities.
- Ensuring required metadata like labels is included.

Admission controllers can validate requests, modify configurations, or perform extra operations before the resources are persisted to etcd.

### Built-In Admission Controllers
Some of the built-in admission controllers in Kubernetes include:

- AlwaysPullImages: Forces image pulling on each pod creation.
- DefaultStorageClass: Automatically assigns a default storage class to PVCs if none is specified.
- EventRateLimit: Limits the number of concurrent API server requests to prevent overload.
- NamespaceExists: Rejects requests to operate in non-existent namespaces.

### Detailed Example: Namespace Existence Check

Consider attempting to create a pod in a non-existent namespace named "blue." When you execute:
```
kubectl run nginx --image nginx --namespace blue
```
The request proceeds through authentication and authorization, then reaches the admission controllers. The NamespaceExists admission controller checks for the "blue" namespace and rejects the request if it does not exist:
```
Error from server (NotFound): namespaces "blue" not found
```
Alternatively, Kubernetes offers the NamespaceAutoProvision admission controller (not enabled by default) to automatically create a missing namespace.

To check the enabled admission controllers, run:
```
kube-apiserver -h | grep enable-admission-plugins
```
This command lists the active admission plugins, including defaults such as NamespaceLifecycle, LimitRanger, ServiceAccount, among others.

### Configuring Admission Controllers

To add a new admission controller, update the `--enable-admission-plugins` flag on the Kube API server service. `/etc/kubernetes/manifests/kube-apiserver.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.17.0.107
    - --allow-privileged=true
    - --enable-bootstrap-token-auth=true
    - --enable-admission-plugins=NodeRestriction,NamespaceAutoProvision
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    name: kube-apiserver
```

**To disable specific admission controllers, use the `--disable-admission-plugins` flag in a similar manner.**

### Admission Controllers Overview

Admission Controllers perform crucial tasks during the API request lifecycle:

- **Mutating Admission Controllers**: Modify the object (mutate the request) before it is created.
- **Validating Admission Controllers**: Inspect a request to decide whether it should be allowed or rejected.
- **Dual-Purpose Controllers**: Some controllers support both mutating and validating functions.

Typically, **mutating controllers are invoked before validating controllers**. This order ensures that any changes made during mutation are considered during validation. For example, if the namespace auto-provisioning admission controller (mutating) runs before the namespace existence admission controller (validating), the missing namespace is created, preventing potential validation failures.

### External Admission Webhooks

While built-in admission controllers are compiled with Kubernetes, you can implement custom logic using external admission webhooks. Two 
specific controllers support webhooks:

- Mutating Admission Webhook
- Validating Admission Webhook
#### How Admission Webhooks Work

1) Webhook Configuration: After passing through all built-in admission controllers, requests are forwarded to the webhook server.
2) Admission Review Request: The API server sends a JSON-formatted admission review object containing request details.
3) Webhook Response: The webhook processes the request and sends back an AdmissionReview response.
