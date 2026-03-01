# Prerequisite CNI

Network namespaces create isolated network environments on a single host. These namespaces are interconnected by a bridge network that establishes virtual interfaces (or virtual cables) for communication between namespaces. This involves assigning IP addresses, activating interfaces, and enabling NAT or IP masquerading for external connectivity. Although Docker configures its bridge networking using similar methods, it employs its own naming conventions. Other container platforms like Rocket, Mesos Containerizer, and Kubernetes address these networking challenges in a comparable way.

To standardize this process and avoid duplicating efforts across multiple platforms, a dedicated program known as "bridge" was developed. This program automates the tasks required to connect a container to a bridge network. For instance, you can run the program with the container ID and network namespace as shown below:

```bash  theme={null}
bridge add 2e34dcf34 /var/run/netns/2e34dcf34
```

The "bridge" program handles low-level networking configuration, freeing container runtime environments from such complexities. When container platforms like Rocket or Kubernetes spin up a new container, they invoke this bridge program passing the container ID and namespace to automatically set up the network.

> By offloading network configuration tasks to a standardized bridge program, container runtimes can focus on higher-level operations while ensuring consistent and reliable network setups via CNI-compliant plugins.

This brings us to an important question: if you want to develop a similar program for a different networking scenario, which commands and arguments should it support? How do you ensure compatibility with container runtimes like Kubernetes or Rocket? The solution lies in establishing a set of standards this is where the Container Networking Interface (CNI) comes into play.

CNI defines a standard for creating and integrating network plugins with container runtime environments. These plugins are responsible for:

* Creating a network namespace for each container.
* Identifying the networks to which the container should connect.
* Configuring the network when a container is created (using the "add" command) and cleaning up when it is deleted (using the "del" command).
* Setting up necessary network details via a JSON configuration file.

On the plugin side, CNI requires support for three command-line arguments: "add", "del", and "check". These commands must accept parameters such as the container ID and network namespace. The plugin then takes over to manage IP addresses and necessary routing, ensuring that containers can communicate effectively. The output of these operations must follow a strict format for consistency.

When both container runtimes and network plugins adhere to CNI standards, seamless interoperability is achieved. Any CNI-compliant plugin can work with any container runtime that supports these standards. The ecosystem already includes several CNI plugins such as bridge, VLAN, IP VLAN, MAC VLAN, and even one designed for Windows. IP address management (IPAM) plugins like host-local and DHCP are also available, along with third-party solutions like Weave, Flannel, Cilium, VMware NSX, Calico, and Infoblox.

>  Docker uses its own networking standard known as the Container Network Model (CNM), which differs from CNI. To use CNI with Docker, you must create a container without network configuration (using the “none” option) and then manually invoke the CNI plugin to set up networking.

Consider the following example that demonstrates how Kubernetes handles networking with Docker:

```bash  theme={null}
docker run --network=none nginx
bridge add 2e34dcf34 /var/run/netns/2e34dcf34
```

In this workflow, Kubernetes first creates a Docker container without any network configuration and then calls the CNI plugin to establish the network. This process highlights how Kubernetes efficiently leverages CNI standards to manage container networks.


# Cluster Networking

The networking configurations necessary for both the master and worker nodes within a Kubernetes cluster. Each node must be equipped with at least one network interface configured with an IP address. Additionally, every host should have a unique hostname and MAC address—this is especially important when creating virtual machines (VMs) by cloning existing ones.

## Required Ports for Kubernetes Components

Effective communication among the control plane components and worker nodes relies on specific port configurations. The following table summarizes the key ports that must be open:

| Port Range  | Component                        | Description                                                                                                                           |
| ----------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 6443        | Kubernetes API Server (master)   | Used by worker nodes, the kube-controller-manager, external users, and other control plane components to access the API Server.       |
| 10250       | Kubelet (master and worker)      | Monitors cluster activities and manages nodes.                                                                                        |
| 10259       | Kube-scheduler (master)          | Required for scheduling operations.                                                                                                   |
| 10257       | Kube-controller-manager (master) | Needed for managing cluster state and various controllers.                                                                            |
| 30000–32767 | Worker Nodes Services            | Exposes services for external access on worker nodes.                                                                                 |
| 2379 & 2380 | etcd Server (master)             | Port 2379 is used for client communication, and port 2380 is used for communication between etcd servers in multi-master deployments. |

## Verifying Network Configuration

To ensure that your cluster’s network environment is set up correctly, it is useful to run several common commands. These commands help you inspect interfaces, IP addresses, hostnames, routing tables, and active services:

```bash  theme={null}
ip link
ip addr
ip addr add 192.168.1.10/24 dev eth0
ip route
ip route add 192.168.1.0/24 via 192.168.2.1
cat /proc/sys/net/ipv4/ip_forward
arp
netstat -plnt
```

These commands are invaluable for gathering information about your network interfaces, IP configurations, and port usage. As you continue to explore your Kubernetes cluster setup, these tools will assist you in troubleshooting and ensuring network connectivity.


# Pod Networking

Kubernetes clusters consist of multiple master and worker nodes with pre-configured networking that allows full node-to-node communication. We assume that all Kubernetes control plane components (e.g., kube-apiserver, etcd, and kubelet) are already set up properly. With this foundation in place, the next step is deploying your applications with a focus on robust pod-level networking.

* Each pod gets a unique IP address.
* Every pod on a node can reach every other pod on that node using its IP address.
* Pods across different nodes can communicate using a consistent addressing scheme.

These requirements ensure that Kubernetes can support both local (node-level) and cross-node communications without relying on NAT rules.

## Understanding the Basics

Before deploying applications, it is crucial to address several questions:

* How are pods assigned IP addresses?
* How do these pods communicate both within a node and across nodes?
* How can services running in these pods be accessed from inside or outside the cluster?

Kubernetes leaves the implementation details for pod networking to the user, as long as the chosen solution meets the basic connectivity requirements. Many networking solutions are available; here, we demonstrate the core concepts using Linux network namespaces, bridge networks, and IP address management inspired by CNI concepts.

### Setting Up a Simple Bridge Network

Below is an example that sets up a simple bridge network and connects network namespaces:

```bash  theme={null}
# Create a Linux bridge interface named v-net-0
ip link add v-net-0 type bridge

# Bring (activate) the bridge interface up
ip link set dev v-net-0 up

# Assign IP address 192.168.15.5/24 to the bridge
# This makes the bridge act like a gateway inside 192.168.15.0/24
ip addr add 192.168.15.5/24 dev v-net-0


# Create a virtual Ethernet pair (like a virtual cable)
# veth-red <--> veth-red-br
# Packets entering one end come out the other
ip link add veth-red type veth peer name veth-red-br

# Move one end of the veth pair (veth-red) into the network namespace "red"
ip link set veth-red netns red

# Inside namespace "red", assign IP 192.168.15.1 to veth-red
ip -n red addr add 192.168.15.1 dev veth-red

# Bring up the interface inside the "red" namespace
ip -n red link set veth-red up

# Attach the other end of the veth pair (veth-red-br)
# to the bridge v-net-0 so it becomes part of the virtual switch
ip link set veth-red-br master v-net-0


# Inside namespace "blue", add a route:
# To reach 192.168.1.0/24, send traffic via 192.168.15.5
# (which is the bridge IP acting as gateway)
ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5


# Add a NAT rule:
# Any packets coming from 192.168.15.0/24
# will have their source IP rewritten (masqueraded)
# when leaving the host (typically to the internet)
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```

Imagine a cluster with three nodes where every node runs a combination of management and workload pods. Although node roles are flexible, the networking concepts remain consistent across nodes.

### Cluster Network Planning

Consider the following plan for a three-node cluster:

1. **External Network Configuration**: Each node has an IP in the 192.168.1.0 series (e.g., 192.168.1.11, 192.168.1.12, and 192.168.1.13).
2. **Container Network Namespaces**: Kubernetes creates unique network namespaces when containers start. These namespaces must connect to a network to allow inter-container communication.
3. **Bridge Networks on Each Node**: Create a bridge network on every node. Assign a distinct subnet to each bridge, such as:
   * Node one: 10.244.1.0/24
   * Node two: 10.244.2.0/24
   * Node three: 10.244.3.0/24

When a container is created, a virtual cable (veth pair) connects its network namespace to the node’s bridge network. One end is inserted into the container’s namespace and the other attaches to the node’s bridge. An IP address is assigned (for example, 10.244.1.2), and a route to the default gateway is configured before enabling the interface.

Below is a sample snippet from a script that connects a container to the network:

```bash  theme={null}
# Create veth pair
ip link add <veth-in-host> type veth peer name <veth-in-namespace>
# Attach one end to the container’s namespace
ip link set <veth-in-namespace> netns <namespace>
# Attach the other end to the bridge
ip link set <veth-in-host> master <bridge>
# Assign IP address to the container’s interface
ip -n <namespace> addr add <IP-address>/24 dev <veth-in-namespace>
# Add default route in the container’s namespace
ip -n <namespace> route add default via <bridge-IP>
# Bring up the container’s interface
ip -n <namespace> link set <veth-in-namespace> up
```

Executing this script on each node ensures that all containers receive an IP address and are connected to their respective internal networks.

### Cross-Node Communication

One of the challenges is enabling communication between pods on different nodes. For example, if a pod with IP 10.244.1.2 on node 1 attempts to ping a pod with IP 10.244.2.2 on node 2, the ping may initially fail due to unknown routes between subnets:

```bash  theme={null}
bluepod$ ping 10.244.2.2
Connect: Network is unreachable
```

To resolve this, add a route on node 1 that directs traffic for 10.244.2.2 via node 2’s external IP (e.g., 192.168.1.12):

```bash  theme={null}
node1$ ip route add 10.244.2.2 via 192.168.1.12
```

After configuring the routing, the ping command should succeed:

```bash  theme={null}
bluepod$ ping 10.244.2.2
64 bytes from 10.244.2.2: icmp_seq=1 ttl=63 time=0.587 ms
64 bytes from 10.244.2.2: icmp_seq=2 ttl=63 time=0.466 ms
```

Similarly, add these routes on all nodes to cover all pod subnets:

```bash  theme={null}
node1$ ip route add 10.244.2.2 via 192.168.1.12
node1$ ip route add 10.244.3.2 via 192.168.1.13
node2$ ip route add 10.244.1.2 via 192.168.1.11
node2$ ip route add 10.244.3.2 via 192.168.1.13
node3$ ip route add 10.244.1.2 via 192.168.1.11
node3$ ip route add 10.244.2.2 via 192.168.1.12
```

>  Manually configuring routes on every host is impractical for large-scale deployments. A more scalable solution involves configuring a centralized router to manage all subnet routes and setting each node's default gateway to this router.

## Integrating Container Network Interface (CNI)

In our lab setup, we executed scripts manually to configure pod networking. However, in a production Kubernetes environment where thousands of pods may be created per minute, this manual approach is not feasible.

This is where the Container Network Interface (CNI) becomes essential. CNI specifies how Kubernetes should invoke a networking script each time a pod is created. To conform with CNI standards, the networking script must have:

* An "add" section to connect the container to the network.
* A "delete" section to disconnect the container, remove interfaces, and free up the IP address.

When the container runtime launches a container, it uses the CNI configuration (provided as a command-line argument) to execute the relevant script with the command "add" and pass the container’s name and namespace identifier.

Below is an example snippet that illustrates the CNI execution process:

```bash  theme={null}
ip -n <namespace> link set <interface> up
ip link del <interface>
./net-script.sh add <container> <namespace>
```

# CNI in kubernetes

The CNI specifies the responsibilities of the container runtime. In Kubernetes, container runtimes such as Containerd or CRI-O create the container network namespaces and attach them to the correct network by invoking the appropriate network plugin. Although Docker was initially the primary container runtime, it has largely been replaced by Containerd as an abstraction layer.

## Configuring CNI Plugins in Kubernetes

When a container is created, the container runtime invokes the necessary CNI plugin to attach the container to the network. Two common runtimes that demonstrate how this process works are Containerd and CRI-O.

> Container runtimes look for CNI plugin executables in the `/opt/cni/bin` directory, while network configuration files are read from the `/etc/cni/net.d` directory.

### Directory Structure for CNI Plugins and Configuration

The network plugins reside in `/opt/cni/bin`, and the configuration files that dictate which plugin to use are stored in `/etc/cni/net.d`. Typically, the container runtime selects the configuration file that appears first in alphabetical order.

For example, you might see the following directories:

```bash
ls /opt/cni/bin
```

```bash
bridge  dhcp  flannel  host-local  ipvlan  loopback  macvlan  portmap  ptp  sample  tuning  vlan  weave-ipam  weave-net  weave-plugin-2.2.1
```

```bash
ls /etc/cni/net.d
```

```bash
10-bridge.conflist
```

In this case, the container runtime chooses the "bridge" configuration file.

### Understanding a CNI Bridge Configuration File

A typical CNI bridge configuration file, adhering to the CNI standard, might look like this:

```bash
cat /etc/cni/net.d/10-bridge.conf
```

```json
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.22.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

In this configuration:

* The `"name"` field (e.g., `"mynet"`) represents the network name.
* The `"type"` field set to `"bridge"` indicates the use of a bridge plugin.
* The `"bridge"` field (e.g., `"cni0"`) specifies the network bridge's name.
* The `"isGateway"` flag designates whether the bridge interface should have an IP address to function as a gateway.
* The `"ipMasq"` option enables network address translation (NAT) through IP masquerading.
* The `"ipam"` (IP Address Management) section uses `"host-local"` to allocate IP addresses from the specified subnet (`"10.22.0.0/16"`) and defines a default route.

> Understanding these configuration fields is crucial for troubleshooting and optimizing Kubernetes networking. The settings in this bridge configuration align with fundamental networking concepts such as bridging, routing, and NAT masquerading.


# CNI weave

Previously, we examined a custom CNI script that handled networking tasks through commands similar to the following:

```bash  theme={null}
./net-script.sh add <container> <namespace>
# Create veth pair
# Attach veth pair
# Assign IP Address
# Bring Up Interface
ip -n <namespace> link set ……
# Delete veth pair
ip link del ……
```

Instead of using our custom approach, the Weave plugin automates and streamlines the network setup. Let's dive in to understand how Weave functions.

## Manual Networking vs. Weave CNI

In traditional networking, the routing table on each host maps different networks. When a packet moves from one pod to another, it usually exits through the network and is directed by a router to the destination node hosting the target pod. Although this works well in small-scale networks, scaling to hundreds of nodes and pods makes managing numerous routing table entries extremely challenging.

Imagine a Kubernetes cluster as a company with different office sites (nodes). Each office has various departments (pods). Initially, a package (packet) may be delivered using a simple routing method. However, as the company expands across regions and countries, maintaining a comprehensive routing table becomes unmanageable.

>  Think of Weave as a specialized shipping service. It deploys dedicated agents (pods) at each node that collectively form a peer-to-peer network, ensuring efficient communication and accurate routing in a large-scale environment.

## How Weave Works

The Weave CNI plugin deploys an agent on each Kubernetes node. These agents exchange information about nodes, networks, and pods to maintain a complete topology of the cluster. Each node runs a Weave bridge, allowing dynamic IP address assignment. In the upcoming practice lesson, you will determine the exact range of IP addresses assigned by Weave.

Keep in mind that a pod may be connected to multiple bridge networks (e.g., both the Weave bridge and the Docker bridge). The container's routing configuration controls the path a packet follows, and Weave ensures that each pod has the correct route through its assigned agent. When sending a packet to a pod on another node, Weave intercepts, encapsulates, and routes it using updated source and destination details. At the destination node, the corresponding Weave agent decapsulates the packet and delivers it to the intended pod.

## Deploying Weave on a Kubernetes Cluster

Deploying Weave on a Kubernetes cluster is straightforward. Once you have set up your base Kubernetes environment with nodes, inter-node networking, and control plane components you can deploy the Weave plugin using a single command. This command deploys the necessary components (Weave peers) as pods on every node, often configured via a DaemonSet.

Here's an example command to inspect the routing settings within a running pod:

```bash  theme={null}
kubectl exec busybox -- ip route
default via 10.244.1.1 dev eth0
```

To deploy the Weave components, run:

```bash  theme={null}
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

The output should confirm successful deployment:

```bash  theme={null}
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.extensions/weave-net created
```

If you are using kubeadm along with the Weave plugin, you will see the Weave peers as pods on each node. For troubleshooting or further verification, list the pods in the kube-system namespace:

```bash  theme={null}
kubectl get pods -n kube-system
```

For more detailed troubleshooting steps and configuration options, refer to the [Weaveworks documentation](https://www.weave.works/docs/net/latest/).


# ipam weave

We explore how Kubernetes assigns IP addresses within virtual bridge networks on nodes, how pods receive their IPs, where this information is stored, and the mechanisms in place to prevent duplicate IP assignments.

## Understanding IP Address Management in Kubernetes

Kubernetes distinguishes between node IPs and pod IPs. This article focuses on the allocation of an IP subnet to the virtual bridge networks on each node and the subsequent assignment of IPs to pods. Node IP addresses are managed separately, often using external IP management solutions.

IP assignment is governed by the Container Network Interface (CNI) standards. The CNI plugin—the network solution provider is responsible for assigning IPs to containers. 

Recall the basic plugin developed earlier, which handled IP assignment within the container network namespace. To ensure smooth network operations, the assigned IPs must be unique. Kubernetes does not enforce a specific IP management method, leaving that responsibility to the CNI solution’s design.

## Manual IP Management Example

One straightforward approach to IP management is to store the list of assigned IPs in a file. A script then manages this file and assigns IP addresses and routes within a specific network namespace. For example:

```bash  theme={null}
# Assign IP Address in a specific namespace
ip -n <namespace> addr add <ip_address>/<subnet_mask> dev <interface>
ip -n <namespace> route add <destination> via <gateway>
```

>  This manual method is best suited for simple environments or testing purposes. For larger deployments, consider using built-in IPAM solutions.

## Using Built-in CNI Plugins

Rather than implementing custom IP assignment logic, you can leverage built-in IPAM plugins provided by the CNI. The host-local plugin, for instance, manages IP addresses locally on each node. A typical workflow might include retrieving a free IP from a maintained file:

```bash  theme={null}
# Retrieve free IP from file
ip=$(get_free_ip_from_file)

# Assign IP Address in a specific namespace
ip -n <namespace> addr add <ip_address>/<subnet_mask> dev <interface>
ip -n <namespace> route add <destination> via <gateway>
```

Alternatively, you can directly invoke the host-local plugin within your script:

```bash  theme={null}
# Invoke IPAM host-local plugin to retrieve a free IP
ip=$(get_free_ip_from_host_local)

# Assign IP Address in a specific namespace
ip -n <namespace> addr add <ip_address>/<subnet_mask> dev <interface>
ip -n <namespace> route add <destination> via <gateway>
```

>  Leveraging CNI plugins such as host-local reduces custom code maintenance and aligns with Kubernetes best practices for network management.

## CNI Configuration

The IPAM configuration is specified in the CNI configuration file. This file includes details such as the plugin type, subnet, and routing rules. Your script can read these parameters at runtime and invoke the appropriate plugin without hard-coding settings. Below is an example configuration using the host-local plugin:

```json 
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": "net-script",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      {
        "dst": "0.0.0.0/0"
      }
    ]
  }
}
```

For more detailed information on CNI configuration options, refer to the [Kubernetes Documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/).

## How Weaveworks Manages IP Addresses

Different network solution providers offer varied mechanisms for IP address management. Weaveworks, for instance, automatically allocates an IP range of 10.32.0.0/12 for the entire cluster. This range, covering IP addresses from 10.32.0.1 to 10.47.255.254, provides approximately one million IPs for pods. The total range is equally divided among the cluster nodes, with each node assigning its share to its pods. These IP ranges are configurable through additional options during the deployment of the Weave plugin.

By understanding both manual and plugin-based IP management strategies, you can implement a robust and scalable network configuration in your Kubernetes clusters. Explore further with practice tests and configuration exercises to deepen your knowledge of IPAM in Kubernetes.

