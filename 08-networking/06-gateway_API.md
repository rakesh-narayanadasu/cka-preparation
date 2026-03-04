# Introduction to Gateway API

Previously, when using Ingress, multiple teams or organizations sharing a single Ingress resource faced coordination challenges. For instance, if team A manages a web service and team B manages a video service, they would need to coordinate their changes on one Ingress resource. This multi-tenancy issue is a significant limitation of Ingress, which can only be managed by one team at a time.

## Limitations of Ingress

Consider the basic Ingress configuration below, which routes traffic based on host names:

```yaml  theme={null}
# ingress-wear-watch.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
    - host: wear.my-online-store.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80
    - host: watch.my-online-store.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```

This configuration supports HTTP-based rules such as host and path matching. However, it does not natively support other routing protocols like TCP, UDP, or advanced features such as traffic splitting, header manipulation, authentication, or rate limiting. To implement these capabilities, controller-specific annotations are used. For example, the following configuration adds NGINX-specific SSL redirection via annotations:

```yaml  theme={null}
# ingress-with-annotations.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: wear.my-online-store.com
      http:
        paths:
          - path: /foo
            pathType: Prefix
            backend:
              service:
                name: wear-service
                port:
                  number: 80
    - host: watch.my-online-store.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: watch-service
                port:
                  number: 80
```

> Because each controller implements its own set of annotations, these configurations are tightly coupled to specific controllers and cannot be validated by Kubernetes itself.

Similarly, advanced scenarios like configuring Cross-Origin Resource Sharing (CORS) require different annotations depending on the controller. For example, a CORS configuration for NGINX might look like this:

```yaml  theme={null}
# Ingress-cors.yaml (for NGINX)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cors-ingress
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, PUT, POST"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://allowed-origin.com"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
```

And for a traffic controller like Traefik, a similar configuration would be:

```yaml  theme={null}
# Ingress-traefik.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: traefik-ingress
  annotations:
    # CORS Configuration
    traefik.ingress.kubernetes.io/headers.customresponseheaders: |
      Access-Control-Allow-Origin: '*'
      Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
      Access-Control-Allow-Headers: Content-Type,Authorization
      Access-Control-Allow-Credentials: true
      Access-Control-Max-Age: 3600
```

These examples clearly demonstrate that the same use case leads to different configurations based solely on the chosen controller.

## Introducing the Gateway API

The Gateway API was created as an official Kubernetes project to overcome the limitations of Ingress. It supports both layer 4 (transport) and layer 7 (application) routing, representing the next generation of load balancing and service mesh APIs. By decoupling responsibilities, the Gateway API introduces three distinct objects:

1. **Gateway Class:** Configured by infrastructure providers to define the underlying network infrastructure (e.g., NGINX, Traefik, or other load balancers).
2. **Gateway:** Managed by cluster operators; these are instances built from a Gateway Class.
3. **HTTPRoute (and other route types):** Managed by application developers; these routes support various protocols such as HTTP, TCP, and gRPC.

Unlike Ingress, the Gateway API offers a unified and declarative configuration that is independent of controller-specific annotations.

### Installing Gateway API with NGINX

The Gateway API defines custom resources, but a controller is needed to implement them. We’ll use the NGINX Gateway Controller, which supports all standard Gateway API resources.

To install the NGINX Gateway Controller, run the following commands:

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -

kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/experimental?ref=v1.6.2" | kubectl apply -f -

helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --create-namespace -n nginx-gateway
```

> What this does: Installs the NGINX Gateway Controller, along with the Gateway API Custom Resource Definitions (CRDs) and related resources.

### Gateway API Configuration Example

The following example illustrates how to define a Gateway Class, a Gateway, and an HTTPRoute using the Gateway API:

```yaml
# gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
  namespace: default
spec:
  controllerName: example.com/gateway-controller
```

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: default
spec:
  gatewayClassName: example-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

```yaml
# http-route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
      namespace: default
  hostnames:
    - "www.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /login
      backendRefs:
        - name: example-svc
          port: 8080
```

In this configuration, the HTTPRoute routes requests with a path prefix of "/login" arriving at http://www.example.com to the backend service "example-svc" on port 8080.

## TLS Configuration with Gateway API

Traditional Ingress configurations handle TLS by using the `spec.tls` section, often accompanied by additional annotations to enforce HTTPS redirection. Consider the following Ingress example for a secure application:

```yaml
# ingress-secure-app.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - secure.example.com
      secretName: tls-secret
```

In contrast, the Gateway API allows for a more structured TLS configuration as part of the listener definition:

```yaml
# gateway-secure.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: secure-gateway
spec:
  gatewayClassName: example-gc
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: tls-secret
  allowedRoutes:
    kinds:
      - kind: HTTPRoute
```

Here, the HTTPS listener on port 443 is explicitly configured with TLS termination, referencing the necessary TLS secret. The `allowedRoutes` field ensures that only HTTPRoute objects can attach to the listener.

## Traffic Splitting and Canary Deployments

Ingress relies on annotations for complex use cases like canary deployments. For example, you might use NGINX annotations to route 20% of the traffic to a new application version:

```yaml
# canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2
            port:
              number: 80
```

The Gateway API simplifies this process by natively defining traffic splitting within an HTTPRoute:

```yaml
# app-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: split-traffic
spec:
  parentRefs:
    - name: app-gateway
  rules:
    - backendRefs:
        - name: app-v1
          port: 80
          weight: 80
        - name: app-v2
          port: 80
          weight: 20
```

In this example, "app-v1" receives 80% of the traffic while "app-v2" handles the remaining 20%. This declarative and controller-agnostic approach greatly simplifies advanced traffic management scenarios.

## Centralized Advanced Configuration

For features like Cross-Origin Resource Sharing (CORS), the Gateway API centralizes configurations without relying on controller-specific annotations. Consider the following Gateway API-based CORS configuration:

```yaml
# http-route-cors.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cors-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: ResponseHeaderModifier
          responseHeaderModifier:
            add:
              - name: Access-Control-Allow-Origin
                value: "*"
              - name: Access-Control-Allow-Methods
                value: "GET, POST, PUT, DELETE, OPTIONS"
              - name: Access-Control-Allow-Headers
                value: "Content-Type,Authorization"
              - name: Access-Control-Allow-Credentials
                value: "true"
              - name: Access-Control-Max-Age
                value: "3600"
  backendRefs:
    - name: api-service
```

This self-contained configuration is consistent and works seamlessly across different Gateway API controllers.

## Gateway API Controller Support

A range of major controllers now support, or are actively implementing, the Gateway API. This industry support includes platforms such as Amazon EKS, Azure Application Gateway for Containers, Contour, Envoy, Google Kubernetes Engine, HAProxy, Istio, Kong, Kuma, NGINX, and many others. Such broad adoption underlines the growing momentum and reliability of the Gateway API.

The Gateway API overcomes the limitations of Ingress by providing a more declarative, structured, and multi-protocol solution for traffic management. By segregating responsibilities among GatewayClass, Gateway, and HTTPRoute, it simplifies multi-tenant management and advanced routing scenarios such as TLS termination, traffic splitting, and CORS configuration.
