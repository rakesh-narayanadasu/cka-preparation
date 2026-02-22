# Service Accounts

There are two main types of accounts in Kubernetes:

* **User Accounts:** Designed for human users like administrators or developers.
* **Service Accounts:** Intended for machine-to-machine interactions or application-specific tasks. For instance, monitoring tools like Prometheus use a service account to query the Kubernetes API for performance metrics, while Jenkins uses one for deploying applications.

## Example: A Kubernetes Dashboard Application

Consider an example: "my Kubernetes dashboard," a basic dashboard application built with Python. This application retrieves a list of Pods from a Kubernetes cluster by sending API requests and subsequently displays the results on a web page. To authenticate its API requests, the application uses a dedicated service account.

### Creating a Service Account

To create a service account named `dashboard-sa`, run:

```bash  theme={null}
kubectl create serviceaccount dashboard-sa
```

To view all service accounts, use:

```bash  theme={null}
kubectl get serviceaccount
```

The output will appear similar to:

```bash  theme={null}
NAME           SECRETS   AGE
default        1         218d
dashboard-sa   1         4d
```

Upon creation, Kubernetes automatically generates a service account token stored as a Secret and links it to the account. To inspect the details of your service account and its token, execute:

```bash  theme={null}
kubectl describe serviceaccount dashboard-sa
```

Expected output:

```bash  theme={null}
Name:                dashboard-sa
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   dashboard-sa-token-kbbdm
Tokens:              dashboard-sa-token-kbbdm
Events:              <none>
```

To examine the token itself, view the corresponding Secret:

```bash  theme={null}
kubectl describe secret dashboard-sa-token-kbbdm
```

Sample output:

```bash  theme={null}
Name:                dashboard-sa-token-kbbdm
Namespace:           default
Labels:              <none>
Type:                kubernetes.io/service-account-token
Data
  token: eyJhbGciOiJSUzI1NiIsImtpZCI6Ij...  (truncated for privacy)
```

This token serves as the authentication bearer token for accessing the Kubernetes API. For example, using curl:

```bash  theme={null}
curl https://192.168.56.70:6443/api -k \
--header "Authorization: Bearer eyJhbgG…"
```
> In your custom dashboard application, you would typically place the token into the appropriate configuration field to enable API authentication.

## Automatic Mounting of Service Account Tokens

When deploying third-party applications (such as a custom dashboard or Prometheus) on a Kubernetes cluster, you can have Kubernetes automatically mount the service account token as a volume into the Pod. This token is typically available at the path: `/var/run/secrets/kubernetes.io/serviceaccount`.

Every namespace includes a default service account that is automatically injected into Pods. For example, consider the following simple Pod manifest using a custom dashboard image:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```

After creating this Pod, running:

```bash  theme={null}
kubectl describe pod my-kubernetes-dashboard
```

will reveal a volume mounted from a Secret (usually named something like `default-token-xxxx`). You might see an excerpt similar to:

```bash  theme={null}
Name:           my-kubernetes-dashboard
Namespace:      default
Status:         Running
IP:             10.244.0.15
Containers:
  nginx:
    Image:        my-kubernetes-dashboard
    Mounts:       /var/run/secrets/kubernetes.io/serviceaccount from default-token-j4hkv (ro)
Volumes:
  default-token-j4hkv:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-j4hkv
    Optional:    false
```

Inside the Pod, listing the contents of the service account directory shows files such as the `token` file containing the bearer token:

```bash  theme={null}
kubectl exec -it my-kubernetes-dashboard -- ls /var/run/secrets/kubernetes.io/serviceaccount
kubectl exec -it my-kubernetes-dashboard -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Using a Different Service Account

By default, Pods use the `default` service account. To assign a different service account—like the previously created `dashboard-sa`—update your Pod definition to include the `serviceAccountName` field:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
    - name: my-kubernetes-dashboard
      image: my-kubernetes-dashboard
```
> Remember that you cannot modify the service account of an existing Pod. To use a new service account, delete and recreate the Pod. Deployments will automatically roll out new Pods when changes are made to the Pod template.

After deploying the updated manifest, running:

```bash  theme={null}
kubectl describe pod my-kubernetes-dashboard
```

will show that the new service account is now in effect, with volume mounting information reflecting the token for `dashboard-sa` (e.g., `dashboard-sa-token-kbbdm`).

If you wish to disable the automatic mounting of the service account token, set `automountServiceAccountToken` to `false` in the Pod specification:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  automountServiceAccountToken: false
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```

## Changes in Kubernetes Versions 1.22 and 1.24

Prior to Kubernetes v1.22, service account tokens were automatically mounted from Secrets without an expiration date. Starting with v1.22, the TokenRequest API (KEP-1205) was introduced to generate tokens that are audience-bound, time-bound, and object-bound—enhancing security significantly.

Below is an example Pod definition using a projected volume sourced from the TokenRequest API:

```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
    - image: nginx
      name: nginx
      volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: kube-api-access-6mtg8
          readOnly: true
  volumes:
    - name: kube-api-access-6mtg8
      projected:
        defaultMode: 420
        sources:
          - serviceAccountToken:
              expirationSeconds: 3607
            path: token
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
          - downwardAPI:
              items:
                - fieldRef:
                    apiVersion: v1
                    fieldPath: metadata.namespace
```

Starting with Kubernetes v1.24, Kubernetes no longer automatically creates non-expiring service account tokens stored as Secrets. Instead, after creating a new service account, you must generate a token explicitly with:

```bash  theme={null}
kubectl create token dashboard-sa
```

This command produces a token with an expiry (by default, one hour from creation). You can verify and decode this token using tools like jq or [jwt.io](https://jwt.io):

```bash  theme={null}
jq -R 'split(".") | select(length > 0) | .[0] | @base64 | fromjson' <<< <TOKEN>
```

If necessary (though not recommended), you can still create a non-expiring token by manually creating a Secret. Ensure the service account exists first:

```yaml  theme={null}
apiVersion: v1
kind: Secret
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: dashboard-sa
type: kubernetes.io/service-account-token
```
> It is highly recommended to use the TokenRequest API to generate tokens, as API-generated tokens provide additional security features such as expiry, audience restrictions, and improved manageability.

# Create a Service Account
```yaml
kubectl create sa dashboard-sa
```

# Role
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - watch
  - list
```

# RoleBinding
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: dashboard-sa # Name is case sensitive
  namespace: default
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

# Pod defination
```yaml  theme={null}
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```

## Summary

* **Service Accounts vs. User Accounts:** Service accounts are meant for applications (or machines), whereas user accounts are for human users.
* **Token Generation:** Creating a service account automatically generates a token stored in a Secret, which is used for API authentication.
* **Automatic Token Mounting:** Pods can automatically mount the service account token at `/var/run/secrets/kubernetes.io/serviceaccount`, though this behavior can be modified.
* **Enhanced Security:** Since Kubernetes v1.22, tokens are generated using the TokenRequest API, making them audience-bound, time-bound, and more secure.
* **Kubernetes v1.24 Changes:** With v1.24, Kubernetes no longer provisions non-expiring tokens automatically via Secrets; use the `kubectl create token` command to generate tokens as needed.

