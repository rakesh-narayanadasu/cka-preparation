# Secrets

While storing non-sensitive details like hostnames or usernames in a ConfigMap is acceptable, placing a password in such a resource is not secure. Kubernetes Secrets provide a mechanism to safely store sensitive information by encoding the data (note: this is not encryption by default).

**Secrets encode data using Base64. Although it provides obfuscation, it is not a substitute for encryption.**

There are two primary approaches to creating a Secret:

- Imperative Creation: Using the command line to create Secrets on the fly.
- Declarative Creation: Defining Secrets in YAML files.

```
DB Host: mysql
DB User: root
DB Password: paswrd
```

### Imperative Creation of a Secret

With the imperative method, you can supply key-value pairs directly via the command line. For example, to create a Secret named "app-secret" with the key-value pair `DB_Host=mysql`:

```
kubectl create secret generic app-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=paswd
```

Alternatively, create a Secret from a file with the `--from-file` option:

```
kubectl create secret generic app-secret --from-file=app_secret.properties
```

### Declarative Creation of a Secret

Below is a sample YAML definition for a Secret:

```
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```

#### Converting Plaintext to Base64

On Linux hosts, you can convert plaintext values to Base64-encoded strings using the `echo -n` command piped to `base64`. For example:

```
echo -n 'mysql' | base64
echo -n 'root' | base64
echo -n 'paswrd' | base64
```

If you need to decode an encoded value, use the base64 --decode command:

```
echo -n 'bXlzcWw=' | base64 --decode
echo -n 'cm9vdA==' | base64 --decode
echo -n 'cGFzd3Jk' | base64 --decode
```

After creating a Secret, you can list and inspect it with the following commands:

```
kubectl get secrets
```

Describe a Secret (without showing sensitive data):

```
kubectl describe secret app-secret
```

View the encoded data in YAML format:

```
kubectl get secret app-secret -o yaml
```

#### Injecting Secrets into a Pod

Once the Secret is created, you can inject it into a Pod using environment variables or by mounting them as files in a volume.

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    envFrom:
    - secretRef:
        name: app-secret
```

```
envFrom:
    - secretRef:
        name: app-secret

env:
    - name: DB_Password
      valueFrom:
        secretKeyRef:
            name: app-secret
            key: DB_Password

volumes:
    - name: app-secret-volume
      secret:
        secretName: app-secret
```
