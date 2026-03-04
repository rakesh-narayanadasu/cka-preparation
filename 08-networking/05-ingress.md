# Ingress

We will explore the differences between Services and Ingress, explain when to use each, and demonstrate how to deploy and configure Ingress resources effectively. 

## Background: Using Services for External Access

Imagine you are deploying an application for a company with an online store at myonlinestore.com. You build your application as a Docker image and deploy it on a Kubernetes cluster as a pod within a Deployment. The application requires a database, so you deploy a MySQL pod and expose it with a ClusterIP Service named MySQL service. Internally, your application functions properly.

To expose the application externally, you create a Service of type NodePort, mapping the application to a high port (e.g., 38080) on your cluster nodes. In this configuration, users access your application using a URL like:

http//:<node_IP>:38080

As traffic increases, the Service load-balances the requests among multiple application pods.

In a production environment, however, you likely want users to access your application through a user-friendly domain name instead of a node IP address with a high port number. To achieve this, you would update your DNS configuration to point to your node IPs and deploy a proxy server that forwards requests from standard port 80 (or 443 for HTTPS) on your DNS to the NodePort defined in your cluster. With this approach, users can simply navigate to myonlinestore.com.

**Cloud-Native Approach with LoadBalancer**

If your application is hosted on a public cloud platform like [Google Cloud Platform (GCP)](https://cloud.google.com/), the process can be simplified further. Instead of creating a NodePort Service, you can deploy a Service of type LoadBalancer. In this setup:

* Kubernetes assigns an internal high port.
* It sends a request to GCP to deploy a network load balancer.
* The cloud load balancer routes traffic to the internal port across all nodes.
* An external IP is provided by the load balancer, which you point your DNS to.

This allows users to access your application directly using myonlinestore.com.

## The Need for Ingress

Imagine your company expands and you launch new services. For example, you might offer:

* A video streaming service on myonlinestore.com/watch
* The original application on myonlinestore.com/wear

Even if both applications run within the same cluster with separate Deployments and Services (e.g., a LoadBalancer Service for the video service), each service might get its own high port and cloud load balancer. Managing multiple load balancers can increase costs, add complexity, and complicate SSL/TLS (HTTPS) configurations.

### Introducing Ingress

Ingress simplifies external access by providing a single externally accessible IP for your Kubernetes applications. It allows you to configure URL-based routing rules, SSL termination, authentication, and more—acting as a built-in layer 7 load balancer.

Even with Ingress, you still need an initial exposure mechanism (via NodePort or a cloud-native LoadBalancer). However, once this is set up, all further changes are made through the Ingress controller.

Without Ingress, you would have to manually deploy and configure a reverse proxy or load balancer (such as NGINX, HAProxy, or Traefik) within your cluster to handle URL routing and manage SSL certificates. Ingress builds on these principles to offer an integrated solution.

## Deploying an Ingress Controller

To use Ingress, you must first deploy an Ingress controller. The controller continuously monitors the cluster for changes in Ingress resources and reconfigures the underlying load balancing solution accordingly.

>  A Kubernetes cluster does not include an Ingress controller by default. If you create Ingress resources without deploying an Ingress controller, they will have no effect.

There are several Ingress controllers available, including GCE, NGINX, Contour, HAProxy, Traefik, and Istio. In this guide, we will use NGINX as the example.

### Deploying the NGINX Ingress Controller

Below is an example of an NGINX Ingress controller deployment:

```yaml  theme={null}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
```

This Deployment creates one replica of the NGINX Ingress controller using a Kubernetes-optimized NGINX image.

To decouple configuration data from the image, you create a ConfigMap. This allows easy updates to log paths, keep-alive thresholds, SSL settings, session timeouts, and more without modifying the image.

Below is an example Service to expose the NGINX Ingress controller using NodePort:

```yaml  theme={null}
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
  selector:
    name: nginx-ingress
```

For a more comprehensive configuration, you can include environment variables to pass the Pod's name and namespace to the container. This enables the Ingress controller to load its configuration dynamically. Here is a complete configuration that includes the Deployment, Service, ConfigMap, and ServiceAccount:

```yaml  theme={null}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
  selector:
    name: nginx-ingress
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
```

In addition to the objects above, you must create the necessary Roles, ClusterRoles, and RoleBindings so that the Ingress controller has permissions to monitor and modify Ingress resources in the cluster.

## Creating Ingress Resources

Once the NGINX Ingress controller is deployed, you can begin creating Ingress resources that define rules and configurations for routing external traffic to backend Services.

### Simple Ingress for a Single Service

The following Ingress resource routes all incoming traffic to a single backend Service named "wear-service" on port 80:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  defaultBackend:
    service:
      name: wear-service
      port:
        number: 80
```

To create this Ingress resource, run:

```bash  theme={null}
kubectl create -f Ingress-wear.yaml
```

You should see output similar to:

```bash  theme={null}
ingress.extensions/ingress-wear created
```

Verify its creation with:

```bash  theme={null}
kubectl get ingress
```

Expected output:

```bash  theme={null}
NAME           HOSTS   ADDRESS   PORTS  AGE
ingress-wear   *       <none>    80     2s
```

This configuration directs all traffic to "wear-service".

### Ingress with Multiple URL Paths

For more complex routing such as directing traffic from different URL paths to different backend Services use Ingress rules. Suppose you want:

* Traffic to myonlinestore.com/wear to go to "wear-service"
* Traffic to myonlinestore.com/watch to go to "watch-service"

Define an Ingress resource as follows:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
    - http:
        paths:
          - path: /wear
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80
          - path: /watch
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```


After creating this resource, you can view its details by running:

```bash  theme={null}
kubectl describe ingress ingress-wear-watch
```

The output will display the rules and backend configurations similar to:

```bash  theme={null}
Name:             ingress-wear-watch
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host              Path    Backends
  ----              ----    --------
  *                 /wear   wear-service:80 (<none>)
                    /watch  watch-service:80 (<none>)
Annotations:
Events:
  Type    Reason      Age   From                      Message
  ----    ------      ----  ----                      -------
  Normal  CREATE      14s   nginx-ingress-controller  Ingress default/ingress-wear-watch
```

If a user visits an undefined URL (e.g., myonlinestore.com/listen), you can configure a default backend to serve a 404 page.

### Ingress Based on Host Names

Another common scenario involves routing traffic based on host names. For example, you might want:

* Traffic for myonlinestore.com to go to "primary-service"
* Traffic for [www.myonlinestore.com](http://www.myonlinestore.com) to go to "secondary-service"

Here’s how you can define the Ingress resource with host-specific rules:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-domain-routing
spec:
  rules:
    - host: myonlinestore.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: primary-service
                port:
                  number: 80
    - host: www.myonlinestore.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secondary-service
                port:
                  number: 80
```

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: critical-ingress
  namespace: critical-space
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /pay
        pathType: Prefix
        backend:
          service:
           name: pay-service
           port:
            number: 8282
```

manifest file to add TLS termination, Add an annotation to the web-app-ingress Ingress to redirect all HTTP requests to HTTPS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: webapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - app.example.local
    secretName: app-tls
  rules:
  - host: app.example.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app
            port:
              number: 80
```

In this example, each rule handles traffic for a specific domain by routing all requests ("/") to the designated backend Service.

> Splitting traffic by URL involves one rule with multiple paths, whereas splitting traffic by domain requires multiple rules with specific host fields. If you do not define a host, the Ingress rule will match all incoming traffic, regardless of the domain.

For additional information, consider reviewing:

* [NGINX Ingress Controller Repository](https://github.com/kubernetes/ingress-nginx)
