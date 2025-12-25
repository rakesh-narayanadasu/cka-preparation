# ETCD 

etcd is a distributed, reliable key-value store that is both simple and fast. 

What Is a Key-Value Store?

Traditional relational databases, such as SQL databases, store data in tables with rows and columns. For example, a table that contains information about individuals might look like this:

- Each row represents a single person.
- Each column holds a specific detail about that person (e.g., name, age).

If you need to include additional information (like salary data for employed individuals or grades for students), you must expand the table. This means adding columns that may not apply universally to every row.

In contrast, a key-value store organizes data as independent documents or files. Each document contains all relevant information for an individual, allowing flexible and dynamic data structures:

- Working individuals can have documents with salary details.
- Students can have documents with grade details.

For complex data transactions, structured formats like JSON or YAML are used. Here are some examples of key-value pairs stored as JSON documents:

```
{
  "name": "John Doe",
  "age": 45,
  "location": "New York",
  "salary": 5000
}
```

```
{
  "name": "Dave Smith",
  "age": 34,
  "location": "New York",
  "salary": 4000,
  "organization": "ACME"
}
```

## Installing etcd

you can download etcd using the following command:

```
curl -L https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz -o etcd-v3.3.11-linux-amd64.tar.gz
```

You can verify the API version by running:
```
./etcdctl --version # for API v2
```

or

```
export ETCDCTL_API=3
./etcdctl version   # for API v3
```

the command list includes options like:

```
put        # create or update a key
get        # retrieve a key or range of keys
del        # delete a key or range of keys
txn        # atomic transactions
watch      # watch keys or prefixes for changes
```

When API version is not set, it is assumed to be set to version 2. And version 3 commands listed above don't work. When API version is set to version 3, version 2 commands listed above don't work.

Apart from that, you must also specify path to certificate files so that ETCDCTL can authenticate to the ETCD API Server. The certificate files are available in the etcd-master at the following path. 

```
--cacert /etc/kubernetes/pki/etcd/ca.crt     
--cert /etc/kubernetes/pki/etcd/server.crt     
--key /etc/kubernetes/pki/etcd/server.key
```

```
kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl get / --prefix --keys-only --limit=10 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt  --key /etc/kubernetes/pki/etcd/server.key" 
```

# ETCD in Kubernetes

etcd is a distributed key-value store that maintains configuration data, state information, and metadata for your Kubernetes cluster. Every object - nodes, pods, configurations, secrets, accounts, roles, and role bindings is stored within etcd. When you run a command like kubectl get, the data is retrieved from this data store.

Any changes you make to the cluster whether adding nodes, deploying pods, or configuring ReplicaSets are first recorded in etcd. Only after etcd is updated are these changes considered to be complete.

*The etcd server typically listens on port 2379 for client requests. Ensuring that the advertised client URL (via the --advertise-client-urls option) is correctly configured is crucial for proper communication between the Kubernetes API Server and etcd.*

### Deployment Methods

Depending on your Kubernetes setup, you can deploy etcd in two primary ways: manually from scratch or automatically with kubeadm. Each method has its use cases, with manual setups providing a deeper understanding of etcd configurations and kubeadm streamlining the deployment process.

### Deploying etcd from Scratch

When setting up your cluster manually, you'll need to download the etcd binaries, install them, and configure etcd as a service on your master node. Manual deployment gives you more control over configuration options, particularly for setting up TLS certificates.

Below is an example of how you might download the etcd binaries and configure the etcd service:
```
wget -q --https-only \
"https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"


# Example etcd service configuration
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
```

### High Availability Considerations

In a production Kubernetes environment, high availability (HA) is paramount. By running multiple master nodes with corresponding etcd instances, you ensure that your cluster remains resilient even if one node fails.

To enable HA, each etcd instance must know about its peers. This is achieved by configuring the `--initial-cluster` parameter with the details of each member in the cluster. For example:

```
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls=https://${INTERNAL_IP}:2380 \
  --listen-peer-urls=https://${INTERNAL_IP}:2380 \
  --advertise-client-urls=https://${INTERNAL_IP}:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
```

In some deployments, you may use separate certificate files for peer communications (e.g., /etc/etcd/peer.pem and /etc/etcd/peer-key.pem). Always tailor these settings to match your desired security posture.

Deploying etcd with kubeadm
For many test environments and streamlined deployments, kubeadm automatically configures etcd. When you use kubeadm, the etcd server runs as a pod within the kube-system namespace, abstracting away the manual setup details.

To view all the pods running in the kube-system namespace, including etcd, run:
```
kubectl get pods -n kube-system
```

To examine the keys stored in etcd (organized under the registry directory), use the following command:
```
kubectl exec etcd-master -n kube-system -- etcdctl get / --prefix --keys-only
```

The etcd root directory, organized as the registry, contains subdirectories for various Kubernetes components such as nodes, pods, ReplicaSets, and Deployments.
