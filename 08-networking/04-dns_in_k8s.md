# DNS in kubernetes

The fundamentals of DNS, including common tools such as `host`, `nslookup`, and `dig` alongside various DNS record types (A, CNAME, etc.) and the domain name hierarchy. We even demonstrated how to set up your own DNS server using CoreDNS. Now, we shift our focus to the DNS names assigned to various Kubernetes objects like services and pods and the different methods of accessing one pod from another.

Imagine a three-node Kubernetes cluster with multiple pods and services distributed across them. Each node typically has a unique name and IP address registered in your organization's DNS server. However, our focus here is on the internal DNS resolution among the cluster’s pods and services. By default, when you create a cluster, Kubernetes deploys a built-in DNS server (unless manually configured otherwise), which facilitates name resolution for pods and services.

Consider a simple scenario with two pods and a service in your cluster:

* A **test pod** with IP `10.244.1.5`.
* A **web pod** with IP `10.244.2.5`.

Even if these pods reside on different nodes (as indicated by their IP addresses), Kubernetes DNS assumes that all pods and services can be reached via their IP addresses. To allow the test pod to communicate with the web pod, a service named **web-service** is created. This service is assigned its own IP address (e.g., `10.107.37.188`) and automatically gets a DNS record mapping the service name to its IP.

Within the cluster, any pod can resolve and access the web service using its service name. For example, to access the web-service from the test pod, you could use:

```bash  theme={null}
curl http://web-service
# Output: Welcome to NGINX!
```

Pods within the same namespace (default namespace is usually "default") can communicate using just their short names. 

In our scenario, because the test pod, web pod, and web-service are all in the **default** namespace, the test pod can simply refer to the service as "web-service." However, if the web-service were deployed in another namespace (for example, "apps"), you would need to access it using "web-service.apps." Here, "apps" becomes part of the fully qualified service name.

To illustrate DNS resolution with namespaces, consider the following examples:

```bash  theme={null}
# When the service is in the default namespace
curl http://web-service
# When the service resides in the 'apps' namespace
curl http://web-service.apps
# Using the fully qualified domain name (FQDN)
curl http://web-service.apps.svc.cluster.local
# Output: Welcome to NGINX!
```

Each namespace in Kubernetes gets its own subdomain. All services within that namespace are grouped under a subdomain called "svc." Additionally, the entire cluster is associated with a root domain (by default, `cluster.local`). Thus, the fully qualified domain name for a service in the "apps" namespace is:

  web-service.apps.svc.cluster.local

Now, let’s discuss pod DNS records. By default, DNS records for pods are not created. However, this behavior can be explicitly enabled. When pod DNS records are activated, Kubernetes generates a DNS record for each pod by converting the pod’s IP address into a hostname replacing dots (`.`) with dashes (`-`). The record includes the pod's namespace, is set to type "pod," and utilizes the cluster's root domain.

For example, if a test pod in the default namespace has the IP `10.244.2.5`, the corresponding DNS record becomes:

  10-244-2-5.apps.pod.cluster.local

This DNS entry resolves to the pod's IP address. You can test the resolution with the command below:

```bash  theme={null}
curl http://10-244-2-5.apps.pod.cluster.local
# Output: Welcome to NGINX!
```

# CoreDNS in Kubernetes

We explored how to address a service or pod from another pod. Now, we will explain how Kubernetes leverages a centralized DNS server to achieve that functionality.

Imagine you have two pods with different IP addresses. One approach to enable communication between them is to add an entry into each pod’s hosts file. For instance, on the first pod, you might map the second pod (named "web") to IP 10.244.2.5, and on the second pod, map the first pod (named "test") to IP 10.244.1.5. However, when dealing with thousands of pods that are frequently created and removed, manually managing these entries becomes impractical.

> Instead of manually editing hosts files, Kubernetes deploys a central DNS server. Each pod is pre-configured via its `/etc/resolv.conf` file to use this centralized server (typically at 10.96.0.10), which automatically updates DNS records for new pods and services.

Kubernetes does not create DNS entries for individual pods manually. Instead, it sets up DNS records for services, and for pods, it converts IP addresses into hostnames by replacing dots with dashes.

Before Kubernetes version 1.12, this service was known as Kube-DNS. Starting with version 1.12, however, the recommended DNS server is CoreDNS, which brings enhanced flexibility and performance. Below is a conceptual illustration showing how pods configure their `/etc/resolv.conf` to point to the DNS server:

```bash  theme={null}
cat /etc/resolv.conf
nameserver 10.96.0.10
```

## CoreDNS Setup in the Cluster

CoreDNS is deployed as a pod within the kube-system namespace. To ensure high availability, Kubernetes runs two replicas of CoreDNS pods managed by a ReplicaSet (now part of a Deployment). Each pod runs the CoreDNS executable, which you could also run manually if deploying CoreDNS independently.

CoreDNS requires a configuration file commonly named "Corefile" and located at `/etc/coredns/Corefile` that outlines various plugins used to process DNS queries. An example configuration is shown below:

```bash  theme={null}
cat /etc/coredns/Corefile
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        upstream
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    proxy . /etc/resolv.conf
    cache 30
    reload
}
```

This configuration performs the following functions:

* Logs and handles errors.
* Provides health check endpoints.
* Integrates with Kubernetes via the Kubernetes plugin, configuring the primary domain to cluster.local and transforming pod IP addresses into a dashed hostname format.
* Exposes Prometheus metrics for monitoring.
* Forwards unresolved DNS queries (such as [www.google.com](http://www.google.com)) to the nameserver specified in the pod’s /etc/resolv.conf.
* Caches DNS responses and supports dynamic reloads of the configuration upon changes.

Note that this configuration is stored in a ConfigMap. If adjustments are needed, simply update the ConfigMap:

```bash  theme={null}
kubectl get configmap -n kube-system
NAME      DATA   AGE
coredns   1      168d
```

Once the CoreDNS pod is running with the correct configuration, it continuously monitors the Kubernetes API for new pods and services, allowing DNS records to be updated dynamically.

## DNS Service and Pod Configuration

To enable pods to communicate with the CoreDNS server, Kubernetes creates a service (named kube-dns by default) with the IP address 10.96.0.10. This IP is automatically set as the primary nameserver in all pod /etc/resolv.conf files. The service details are as follows:

```bash  theme={null}
kubectl get service -n kube-system
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
kube-dns     ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP     1d
```

A typical pod’s /etc/resolv.conf file will appear as:

```bash  theme={null}
cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

This configuration is automatically managed by the Kubelet. If you inspect the Kubelet configuration file, you will find entries for both the cluster DNS and the cluster domain:

```bash  theme={null}
cat /var/lib/kubelet/config.yaml
...
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
```

## Resolving Services and Pods

With the correct DNS configuration, pods can resolve services using different domain name formats. For example, if you have a web service deployed in your cluster, you can access it using any of the following names:

* web-service
* web-service.default
* web-service.default.svc
* web-service.default.svc.cluster.local

These examples are demonstrated in the commands below:

```bash  theme={null}
cat /etc/resolv.conf
nameserver 10.96.0.10
curl http://web-service
curl http://web-service.default
curl http://web-service.default.svc
curl http://web-service.default.svc.cluster.local
```

You can also use tools like nslookup or host to check the fully qualified domain name:

```bash  theme={null}
host web-service
web-service.default.svc.cluster.local has address 10.107.37.188
```

The search entries in /etc/resolv.conf allow the resolver to append the listed domains when you use partial names. However, individual pod names must always be resolved with their complete fully qualified domain name (FQDN).

For example:

```bash  theme={null}
cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local

host web-service
web-service.default.svc.cluster.local has address 10.107.37.188

host 10-244-2-5
Host 10-244-2-5 not found: 3(NXDOMAIN)

host 10-244-2-5.default.pod.cluster.local
10-244-2-5.default.pod.cluster.local has address 10.244.2.5
```

> The search entries in the `/etc/resolv.conf` file simplify service resolution by allowing shorter names. However, pod-specific DNS records require their full FQDN for proper resolution.

