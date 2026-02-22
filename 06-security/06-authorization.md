# Authorization

We explore how authentication allows individuals or machines to gain access to a cluster and how authorization subsequently defines what actions they can perform within that cluster. Once a user gains access, authorization ensures they only have the appropriate permissions for their role. For example, a cluster administrator can view various objects such as Pods, Nodes, and Deployments:

```bash  theme={null}
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          53s

kubectl get nodes
NAME        STATUS   ROLES     AGE     VERSION
worker-1    Ready    <none>    5d21h   v1.13.0
worker-2    Ready    <none>    5d21h   v1.13.0

kubec
```

Administrators have full control, allowing them to create or delete objects like Pods or Nodes. As the cluster scales and more users including administrators, developers, testers, or external applications like monitoring tools and Jenkins access the system, it is critical to provide only the access level necessary for each user’s role. For instance, developers might be limited to deploying applications without the ability to modify the overall cluster configuration.

Below is an example demonstrating operations executed with limited permissions:

```plaintext  theme={null}
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          53s

kubectl get nodes
NAME       STATUS   ROLES     AGE     VERSION
worker-1   Ready    <none>    5d21h   v1.13.0
worker-2   Ready    <none>    5d21h   v1.13.0

kubectl delete node worker-2
Node worker-2 Deleted!
```

In contrast, attempting similar operations without sufficient privileges results in the following responses:

```bash  theme={null}
kubectl get pods
Error from server (Forbidden): pods is forbidden: User "Bot-1" cannot list "pods"

kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "Bot-1" cannot get "nodes"

kubectl delete node worker-2
Error from server (Forbidden): nodes "worker-2" is forbidden: User "developer" cannot delete resource "nodes"
```

When sharing a cluster across different organizations or teams using namespaces, authorization restricts users to their designated namespaces. Kubernetes supports multiple authorization mechanisms, including:

* Node Authorization
* Attribute-Based Authorization (ABAC)
* Role-Based Access Control (RBAC)
* Webhook Authorization

The Kubernetes API Server is the central component accessed by both management users and internal components, such as kubelets, which retrieve and report metadata about services, endpoints, nodes, and pods. 

Requests from kubelets typically using certificates with names prefixed by "system:node" as part of the system:nodes group are authorized by a special component known as the node authorizer.

>  Kubernetes supports several authorization strategies to meet diverse security requirements. Always select the most appropriate mechanism for your cluster’s needs.

## Attribute-Based Authorization

Attribute-based authorization associates specific users or groups with a defined set of permissions. For example, you can grant a user called "dev-user" permissions to view, create, and delete pods. This is achieved by creating a policy file in JSON format and passing it to the API server. Consider the following example policy file:

```json  theme={null}
{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "dev-user-2", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"group": "dev-users", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "security-1", "namespace": "*", "resource": "csr", "apiGroup": "*"}}
```

Each time security requirements change, you must manually update this policy file and restart the Kube API Server. This manual process can be tedious and set the stage for more streamlined methods such as Role-Based Access Control (RBAC).

## Role-Based Access Control (RBAC)

RBAC simplifies user permission management by defining roles instead of directly associating permissions with individual users. For example, you can create a "developer" role that encompasses only the necessary permissions for application deployment. Developers are then associated with this role, and modifications in user access can be handled by updating the role, affecting all associated users immediately.

RBAC is considered the standard method for managing access within a Kubernetes cluster.

## External Authorization Mechanisms

If you prefer managing authorization externally rather than with built-in Kubernetes mechanisms, third-party tools like [Open Policy Agent (OPA)](https://www.openpolicyagent.org/) are an excellent choice. OPA can handle both admission control and authorization by processing user details and access requirements sent via API calls from Kubernetes. Based on OPA’s response, access is either granted or denied.

## AlwaysAllow and AlwaysDeny Modes

Kubernetes also supports two basic authorization modes:

* **AlwaysAllow:** Permits all requests without performing any authorization checks.
* **AlwaysDeny:** Denies all requests.

These modes are configured using the authorization-mode option on the Kube API Server and are crucial when determining which authorization mechanism is active. In cases where no mode is specified, AlwaysAllow is used by default.

Below is an example configuration using AlwaysAllow:

```bash  theme={null}
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=AlwaysAllow \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --kubelet-client-key=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --service-node-port-range=30000-32767 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \\
  -v=2
```

You can also specify a comma-separated list of multiple authorization modes. For example, to configure node authorization, RBAC, and webhook authorization, set the parameter as follows:

```bash  theme={null}
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --authorization-mode=Node,RBAC,Webhook \\
  --bind-address=0.0.0.0 \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/apiserver-etcd-client.crt \\
  --etcd-keyfile=/var/lib/kubernetes/apiserver-etcd-client.key \\
  --etcd-servers=https://127.0.0.1:2379 \\
  --event-ttl=1h \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \\
  --tls-cert-file=/var/lib/kubernetes/apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/apiserver.key \\
  --v=2
```

When multiple modes are configured, each request is processed sequentially in the order specified. For example, a user’s request is first evaluated by the node authorizer. If the request does not pertain to node-specific actions and is consequently denied, it is then passed to the next module, such as RBAC. Once a module approves the request, further checks are bypassed and the user is granted access.


# Role Based Access Controls

In this we dive into Kubernetes Role-Based Access Controls (RBAC) to help you manage permissions effectively. You'll learn how to create roles, bind them to users, and verify permissions within a namespace.

## Creating a Role

To define a role, create a YAML file that sets the API version to `rbac.authorization.k8s.io/v1` and the kind to `Role`. In this example, we create a role named **developer** to grant developers specific permissions. The role includes a list of rules where each rule specifies the API groups, resources, and allowed verbs. For resources in the core API group, provide an empty string (`""`) for the `apiGroups` field.

For instance, the following YAML definition grants developers permissions on pods (with various actions) and allows them to create ConfigMaps:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```

Create the role by running:

```bash  theme={null}
kubectl create -f developer-role.yaml
```
>  Both roles and role bindings are namespace-scoped. This example assumes usage within the default namespace. To manage access in a different namespace, update the YAML metadata accordingly.

## Creating a Role Binding

After defining a role, you need to bind it to a user. A role binding links a user to a role within a specific namespace. In this example, we create a role binding named **devuser-developer-binding** that grants the user `dev-user` the **developer** role.

Below is the combined YAML definition for both creating the role and its corresponding binding:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

Create the role binding using the command:

```bash  theme={null}
kubectl create -f devuser-developer-binding.yaml
```

## Verifying Roles and Role Bindings

After applying your configurations, it's important to verify that the roles and role bindings have been created correctly.

To list all roles in the current namespace, execute:

```bash  theme={null}
kubectl get roles
```

Example output:

```bash  theme={null}
NAME        AGE
developer   4s
```

Next, list all role bindings:

```bash  theme={null}
kubectl get rolebindings
```

Example output:

```bash  theme={null}
NAME                      AGE
devuser-developer-binding 24s
```

For detailed information about the **developer** role, run:

```bash  theme={null}
kubectl describe role developer
```

Sample output:

```bash  theme={null}
Name:         developer
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources           Non-Resource URLs   Resource Names   Verbs
  -----------         ------------------   --------------   ----
  ConfigMap           []                   []               [create]
  pods                []                   []               [get watch list create delete]
```

To view the specifics of the role binding:

```bash  theme={null}
kubectl describe rolebinding devuser-developer-binding
```

Example output:

```bash  theme={null}
Name:         devuser-developer-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:    Role
  Name:    developer
Subjects:
  Kind     Name      Namespace
  ----     ----      ---------
  User     dev-user
```

## Testing Permissions with kubectl auth

You can test whether you have the necessary permissions to perform specific actions by using the `kubectl auth can-i` command. For example, to check if you can create deployments, run:

```bash  theme={null}
kubectl auth can-i create deployments
```

This command might return:

```bash  theme={null}
yes
```

Similarly, to verify if you can delete nodes:

```bash  theme={null}
kubectl auth can-i delete nodes
```

Expected output:

```bash  theme={null}
no
```

To test permissions for a specific user without switching user contexts, use the `--as` flag. Although the **developer** role does not permit creating deployments, it does allow creating pods:

```bash  theme={null}
kubectl auth can-i create deployments
# Output: yes
kubectl auth can-i delete nodes
# Output: no
kubectl auth can-i create deployments --as dev-user
# Output: no
kubectl auth can-i create pods --as dev-user
# Output: yes
```

You can also specify a namespace in your commands to verify permissions scoped to that particular namespace.

## Limiting Access to Specific Resources

In some scenarios, you may want to restrict user access to a select group of resources. For example, if you have multiple pods in a namespace but only intend to provide access to pods named "blue" and "orange," you can utilize the `resourceNames` field in the role rule.

Start with a basic role definition without any resource-specific restrictions:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
```

Then, update the rule to restrict access solely to the "blue" and "orange" pods:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
  resourceNames: ["blue", "orange"]
```

# Cluster Roles

We covered roles and role bindings, which are namespace-specific. In this article, we extend that concept by introducing cluster roles and cluster role bindings, allowing you to manage permissions across your entire Kubernetes cluster.

When you create roles and role bindings without specifying a namespace, they are added to the default namespace and only authorize access within that scope. This works well for namespaced resources such as pods, deployments, and services but not for cluster-scoped resources. For example, nodes cannot be assigned to a specific namespace (e.g., "node01" cannot belong to the "dev" namespace). Cluster-scoped resources like nodes and persistent volumes are managed at the cluster level, so they require a different approach.

Most resources (such as pods, replica sets, jobs, deployments, services, and secrets) are namespaced. In contrast, cluster-scoped resources, including nodes and persistent volumes, do not belong to any namespace. The following image clearly illustrates the difference between namespaced resources (e.g., pods, services) and cluster-scoped resources (e.g., nodes, cluster roles):

To list namespaced and non-namespaced resources, you can use these commands:

```bash  theme={null}
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

> Remember: Roles and role bindings are ideal for namespace-level access, while cluster roles and cluster role bindings extend permissions across the cluster.

## Cluster Roles and Cluster Role Bindings

To authorize cluster-scoped resources, such as nodes and persistent volumes, you need to create cluster roles and cluster role bindings. Cluster roles function similarly to roles, but they are tailored for actions that span the entire cluster.

For example, you can define a cluster administrator role that grants the ability to list, retrieve, create, and delete nodes. Alternatively, you might establish a storage administrator role to manage persistent volumes and persistent volume claims.

Below is an example of a cluster role definition file named `cluster-admin-role.yaml`. This YAML file defines a ClusterRole that grants administrative permissions on nodes:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

Once the ClusterRole is created, you bind it to a user through a ClusterRoleBinding object. The following example binds the `cluster-administrator` ClusterRole to a user named `cluster-admin`:

```yaml  theme={null}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

Apply these configurations using the `kubectl create` command.

It's important to note that while cluster roles and role bindings are primarily used for cluster-scoped resources, they can also manage access to namespace-scoped resources. When you bind a cluster role that grants permissions on pods, for instance, the user will have access to pods in every namespace—unlike a namespaced role which restricts access to a single namespace.

> Kubernetes provides several default cluster roles when the cluster is initially set up. Be sure to review these defaults to understand the baseline permissions before creating custom roles.

