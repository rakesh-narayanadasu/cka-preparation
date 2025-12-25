# Kube API Server

The Kube API Server acts as the central management component in a Kubernetes cluster by handling requests from kubectl, validating and authenticating them, interfacing with the etcd datastore, and coordinating with other system components.

When you execute a command like:
```
kubectl get nodes
```
The utility sends a request to the API Server. The server processes this request by authenticating the user, validating the request, fetching data from the etcd cluster, and replying with the desired information. For example, the output of the command might be:
```
NAME      STATUS   ROLES    AGE   VERSION
master    Ready    master   20m   v1.11.3
node01    Ready    <none>   20m   v1.11.3
```
### API Server Request Lifecycle

When a direct API POST request is made to create a pod, the API Server:

- Authenticates and validates the request.
- Constructs a pod object (initially without a node assignment) and updates the etcd store.
- Notifies the requester that the pod has been created.

For instance, using a curl command:
```
curl -X POST /api/v1/namespaces/default/pods ...[other]
Pod created!
```
The scheduler continuously monitors the API Server for pods that need node assignments. Once a new pod is detected, the scheduler selects an appropriate node and informs the API Server. The API Server then updates the etcd datastore with the new assignment and passes this information to the Kubelet on the worker node. The Kubelet deploys the pod via the container runtime and later updates the pod status back to the API Server for synchronization with etcd.

### Deployment and Setup

If your cluster is bootstrapped with a kube admin tool, most of these intricate details are abstracted. However, when setting up a cluster on your own hardware, you need to download the Kube API Server binary from the Kubernetes release page, configure it, and run it as a service on the Kubernetes master node.

Typical Service Configuration:

The Kube API Server is launched with a variety of parameters to secure communication and manage the cluster effectively. Below is an example of a typical service configuration file:
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-apiserver

# kube-apiserver.service
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --kubelet-https=true \
  --runtime-config=api/all \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --v=2
```
The configuration includes several certificate-related options, securing communication channels between various Kubernetes components.

### Verifying the Deployment

For clusters set up with kube-admin tools, the Kube API Server is deployed as a pod in the kube-system namespace. To inspect these pods, run:
```
kubectl get pods -n kube-system
```

```
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.17.0.32
    - --allow-privileged=true
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --disable-admission-plugins=PersistentVolumeLabel
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
```

Another way to review the active API Server configuration is by checking the systemd service file on the master node:
```
cat /etc/systemd/system/kube-apiserver.service
```

