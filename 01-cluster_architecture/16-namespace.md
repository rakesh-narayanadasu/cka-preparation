# Namespaces

Namespaces allow you to group and manage resources differently based on their context and intended use.

Namespaces allow you to set distinct policies and resource limits for different environments. This isolation prevents one namespace from interfering with another. For instance, you can apply separate resource quotas for CPU, memory, and the total number of pods to ensure fair usage across environments.

### Default Namespace and System Namespaces

By default, when you create objects such as pods, deployments, and services in your cluster, they are placed within a specific namespace (similar to being "inside a house"). The default namespace is automatically created during the Kubernetes cluster setup. Additionally, several system namespaces are created at startup:

- kube-system: Contains core system components like network services and DNS, segregated from user operations to prevent accidental changes.
- kube-public: Intended for resources that need to be publicly accessible across users.

Note: If you're running a small environment or a personal cluster for learning, you might predominantly use the default namespace. In enterprise or production environments, however, namespaces provide essential isolation and resource management by allowing environments like development and production to coexist on the same cluster.

Within a single namespace, resources can refer to each other directly via their simple names. For example, a web application pod in the default namespace can access a database service simply by using its service name (db-service).

If the web app pod needs to communicate with a service located in a different namespace, you must use its fully qualified DNS name. For example, connecting to a database service named "db-service" in the "dev" namespace follows this format:
```
mysql.connect("db-service.dev.svc.cluster.local")
```
db-service -> database service name

svc -> indicates the service subdomain

dev -> namespace

cluster.local -> kubernetes default domain

To create the same pod in the "dev" namespace, you can either include the namespace option:
```
kubectl create -f pod-definition.yml --namespace=dev
```

Or define the namespace within the pod definition file:

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev
  labels:
    app: myapp
    type: front-end
spec:
  containers:
    - name: nginx-container
      image: nginx
```

```
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

If you're working across multiple namespaces and want to avoid repeatedly specifying the namespace flag, you can set the default namespace for your current context:
```
kubectl config set-context $(kubectl config current-context) --namespace=dev
```

## Controlling Resource Usage with ResourceQuotas

To ensure that no single namespace overconsumes cluster resources, Kubernetes allows you to define ResourceQuotas. For example, create a file named compute-quota.yaml with the following content:

```
apiVersion: v1
kind: ResourceQuota
metadata:
    name: compute-quota
    namespace: dev
spec:
    hard:
        pods: "10"
        requests.cpu: "4"
        requests.memory: 5Gi
        limits.cpu: "10"
        limits.memory: 10Gi
```

This configuration guarantees that the "dev" namespace does not exceed the specified resource limits.

