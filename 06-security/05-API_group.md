# API Groups

Before diving into authorization, it's essential to understand the concept of API groups in Kubernetes and how they integrate with overall cluster operations. This article provides an in-depth look into Kubernetes API groups, their structure, and the methods for querying the API server.

## Understanding the Kubernetes API

The Kubernetes API is the primary interface for interacting with your cluster. Whether using the command-line tool `kubectl` or directly sending HTTP requests via REST, every interaction communicates with the API server. For example, to check your cluster's version, run:

```bash  theme={null}
curl https://kube-master:6443/version
```

The response may look like:

```json  theme={null}
{
  "major": "1",
  "minor": "13",
  "gitVersion": "v1.13.0",
  "gitCommit": "ddf47ac13c1a9483ea035a79cd7c1005ff21a6d",
  "gitTreeState": "clean",
  "buildDate": "2018-12-03T20:56:12Z",
  "goVersion": "go1.11.2",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Likewise, listing pods in the cluster involves accessing the `/api/v1/pods` endpoint.

## API Groups and Their Purpose

Kubernetes organizes its API into multiple groups based on specific functionality. These groups help in managing versioning, health metrics, logging, and more. For instance, the `/version` endpoint provides cluster version data, while endpoints like `/metrics` and `/healthz` offer insights into the cluster’s performance and health.

This focuses on two main API group categories:

1. **Core API Group:**\
   Contains the essential features of Kubernetes such as namespaces, pods, replication controllers, events, endpoints, nodes, bindings, persistent volumes, persistent volume claims, config maps, secrets, and services.

2. **Named API Groups:**\
   Provides an organized structure for newer features. These groups include apps, extensions, networking, storage, authentication, and authorization. For example, under the apps group, you’ll find Deployments, ReplicaSets, and StatefulSets, whereas the networking group hosts resources such as Network Policies. Certificate-related resources like Certificate Signing Requests are also grouped under their relevant named groups.

Every API group includes various resources along with associated actions (verbs) such as list, get, create, delete, update, and watch.

For detailed information on the objects in each API group, consult the official Kubernetes API reference documentation.

## Querying the API Server

To retrieve the list of available API groups, access the API server's root endpoint on port 6443:

```bash  theme={null}
curl http://localhost:6443 -k
```

The command returns a JSON response similar to:

```json  theme={null}
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/healthz",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/swagger-2.0.0.json"
  ]
}
```
>  When using `curl` without proper authentication, only selected endpoints (like `/version`) may be accessible. Unauthenticated requests to protected endpoints will result in a 403 Forbidden error.

For example, an unauthenticated request may yield:

```bash  theme={null}
curl http://localhost:6443 -k
```

```json  theme={null}
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

To fully access the API server, use your certificate files with `curl`:

```bash  theme={null}
curl http://localhost:6443 -k \
  --key admin.key \
  --cert admin.crt \
  --cacert <your-ca-cert-file>
```

Alternatively, you can use the `kubectl proxy` command, which starts a local HTTP proxy server on port 8001 using the credentials in your kubeconfig file. This eliminates the need to manually specify certificate files. Start the proxy by running:

```bash  theme={null}
kubectl proxy
```

The output confirms the proxy is running:

```bash  theme={null}
kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Now, you can access the API server through the proxy:

```bash  theme={null}
curl http://localhost:8001 -k
```

The typical response should be:

```json  theme={null}
{
  "paths": [
    "/api/",
    "/api/v1",
    "/apis",
    "/apis/",
    "/healthz",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/swagger-2.0.0.json"
  ]
}
```
>  Remember that "kube proxy" and "kubectl proxy" serve different purposes. The former facilitates pod-to-pod and service communication within the cluster, while the latter is a local HTTP proxy for accessing the API server.
