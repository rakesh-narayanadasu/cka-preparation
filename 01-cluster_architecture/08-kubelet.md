# Kubelet

The Kubelet is often oversees node activities by managing container operations such as starting and stopping containers based on instructions from the master scheduler. Additionally, the Kubelet registers the node with the Kubernetes cluster and continuously monitors the state of pods and their containers. It regularly reports the status of the node and its workloads to the Kubernetes API server.

When the Kubelet receives instructions to run a container or pod, it communicates with the container runtime (e.g., Docker) to download the required image and initiate the container. It then maintains the health of these containers and ensures they operate as expected.

### Installing the Kubelet

Unlike other Kubernetes components, the Kubelet is not automatically deployed when you set up your cluster using tools like kubeadm. It must be installed manually on each worker node. Follow the steps below to install the Kubelet:

Step 1: Download the Kubelet Binary

Execute the following command to download the Kubelet binary:
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
```
Step 2: Configure and Run the Kubelet as a Service

Set up the Kubelet with the required configuration by running it as a service. Use the command below to start the Kubelet with the necessary parameters:
```
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --network-plugin=cni \
  --register-node=true \
  --v=2
```
Step 3: Verify the Kubelet Process

After installation, verify that the Kubelet is running by checking its process status on the worker node. Run the following command:
```
ps -aux | grep kubelet
```

