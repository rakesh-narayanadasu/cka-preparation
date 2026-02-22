# Network Policies

## Traffic Flow Example

Consider a simple application configuration consisting of a web server, an API server, and a database server. The traffic flow is as follows:

1. A user sends a request to the web server on port 80.
2. The web server forwards this request to the API server on port 5000.
3. The API server retrieves data from the database server on port 3306, then sends the response back to the user.

There are two main types of network traffic involved:

* **Ingress traffic:** Incoming traffic. For example, user requests arriving at the web server on port 80.
* **Egress traffic:** Outgoing traffic. For example, requests sent from the web server to the API server.

In our diagrams, a solid arrow indicates the direction of the originating traffic (either ingress or egress), while a dotted arrow represents the response flow, which is typically not controlled by network policies.

For clarity:

* The **API server** receives ingress traffic from the web server (port 5000) and sends out egress traffic to the database server (port 3306).
* The **database server** receives ingress traffic on port 3306 from the API server.

To support this traffic flow, the following rules must be established:

* An ingress rule for the web server to allow HTTP traffic on port 80.
* An egress rule for the web server to permit traffic to port 5000 on the API server.
* An ingress rule for the API server to accept traffic on port 5000.
* An egress rule for the API server to allow traffic to port 3306 on the database server.
* An ingress rule for the database server to allow traffic on port 3306.

## Network Security in Kubernetes

In a Kubernetes cluster, nodes host pods and services, each assigned a unique IP address. A crucial capability of Kubernetes is that pods can communicate with one another without extra configuration—such as setting up custom routes. Typically, all pods reside in a virtual private network (VPN) that spans the entire cluster, allowing them to interact using pod IPs, pod names, or configured services.

By default, Kubernetes employs an "all-allow" rule permitting any pod to communicate with every other pod or service within the cluster.

Now, consider the earlier scenario with three pods: one for the front-end web server, one for the API server, and one for the database server. Services facilitate communication between these pods and external users, while the default configuration allows free communication across the cluster.

### Restricting Communication with Network Policies

If your security requirements dictate that the front-end web server should not communicate directly with the database server, you can enforce this by implementing a network policy. For example, you might create a policy that only permits the API server to interact with the database server.

A network policy in Kubernetes is defined as an object, which you attach to one or more pods using labels and selectors. In this scenario, the policy would only allow ingress traffic from the API pod on port 3306 while blocking all other sources from accessing the database pod.

### Implementing a Network Policy

To apply a network policy, you assign labels to pods and define matching selectors in the network policy object. For example, consider this snippet used to select the database pod:

```yaml  theme={null}
podSelector:
  matchLabels:
    role: db
```

This configuration ensures the network policy only applies to pods labeled with `role: db`. Next, you define policy rules to allow only ingress traffic from the API pod on port 3306.

Below is the complete network policy configuration:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```

In this configuration:
  * The `podSelector` targets the database pod through its label.
  * The `policyTypes` field specifies that only ingress traffic is affected.
  * The ingress rule allows traffic specifically from pods with the label `name: api-pod` on TCP port 3306.

Keep in mind that isolation only applies to the traffic explicitly defined under `policyTypes`. Unspecified traffic is automatically allowed by default.

Below are two Pod YAMLs that match your NetworkPolicy:
- One DB pod with label role: db
- One API pod with label name: api-pod

These labels must match exactly, or the NetworkPolicy won’t work.

Below is the DB pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    role: db
spec:
  containers:
  - name: mysql
    image: mysql:8
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: password
```

Below is the API pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    name: api-pod
spec:
  containers:
  - name: api-container
    image: busybox
    command: ["sleep", "3600"]
```

## Enabling Network Policies in Kubernetes

To enforce the network policy, execute the following command:

```bash  theme={null}
kubectl create -f db-policy.yaml
```
>  Network policies are enforced by the cluster's networking solution. While solutions like Kube-router, Calico, Romana, and Weave Net support network policies, Flannel does not enforce them. Creating policies with Flannel will not produce an error, but they won’t be applied. Always consult your network solution’s documentation to verify its support for network policies.


# Developing network policies

We explore advanced network policy configurations within Kubernetes. We will use our familiar web API and database pods to demonstrate how to secure your database pod by allowing only authorized access. Specifically, only the API pod will access the database pod on port 3306, while other pods (such as the web pod) remain unrestricted. By default, Kubernetes permits all traffic between pods, so explicit network policies are necessary to enforce these security measures.

## Blocking All Ingress Traffic to the Database Pod

To begin, we block all incoming traffic to the database pod. This is achieved by creating a network policy that targets the database pod using labels and selectors. In our example, the database pod is labeled `role: db`. The following YAML file defines a policy that denies all ingress traffic by default since no specific ingress rules are provided:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
```

>  If no ingress rules are specified, Kubernetes treats the policy as a complete block for incoming traffic.

## Allowing Ingress Traffic from the API Pod

The next step is to allow ingress traffic from the API pod on port 3306. Because responses to permitted traffic are automatically allowed, configuring an ingress rule is sufficient. In this rule, the source is defined by pod selectors (and optionally namespace selectors) while the destination port is specified.

For instance, to allow traffic only from pods labeled `name: api-pod` within namespaces labeled `prod` and also permit traffic from a backup server with IP `192.168.5.10` update the network policy as follows:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:        # API pods with api-pod labels
          name: api-pod
      namespaceSelector:
        matchLabels:        # allow only from specific prod namespace
          name: prod
    - ipBlock:              # allow from speific IP block 
        cidr: 192.168.5.10/32
  ports:
  - protocol: TCP
    port: 3306
```

In this configuration, the first entry under the `from` section restricts traffic to API pods in the production namespace (an AND condition). The second entry allows an external backup server by specifying its IP block.

>  Combining pod selectors with namespace selectors ensures that the rule applies only to the intended pods within the correct namespace.

## Configuring Egress Traffic

In scenarios where the database pod must initiate outbound connections (for example, sending backups to an external server), an egress rule becomes necessary. To support both ingress and egress traffic, include `Egress` in the `policyTypes` and specify an egress rule.

The revised policy below permits the database pod to send traffic to a backup server at IP `192.168.5.10` on port `80` while still restricting all other outbound connections:

```yaml  theme={null}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
  ports:
  - protocol: TCP
    port: 3306
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```

>  Ensure that your egress rules cover all required outbound connections. Missing an egress rule may inadvertently block critical communication between your services.

## Summary of Network Policy Configuration

| Configuration Aspect | Description                                                                                                     | YAML Reference                                                |
| -------------------- | --------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Blocking Traffic     | Deny all ingress traffic to the database pod by default.                                                        | Initial policy with `podSelector` for `role: db`.             |
| Allowing Ingress     | Permit API pod access on port 3306 by combining pod and namespace selectors. Also allow a specific external IP. | Ingress rule with pod and namespace selectors plus `ipBlock`. |
| Configuring Egress   | Enable the database pod to send outbound traffic to an external backup server on port 80.                       | Egress rule addition with `ipBlock` for backup server.        |


* [Kubernetes Networking Concepts](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
