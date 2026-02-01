# Configure Environment Variables

### Setting Environment Variables Using Docker

When running a Docker container, you can set environment variables using the `-e` flag. For example, the command below sets the `APP_COLOR` environment variable to "pink":

```
docker run -e APP_COLOR=pink simple-webapp-color
```

This command assigns the value "pink" to `APP_COLOR` while launching the `simple-webapp-color` container.

### Configuring Environment Variables in Kubernetes Pods

Kubernetes allows you to define environment variables within your pod definitions. In the pod manifest, environment variables are listed under the env property, which is an array. Each entry in the array should specify:

- `name`: The name of the environment variable.
- `value`: The corresponding value assigned to that environment variable.

Below is an example pod definition that explicitly sets the environment variable:

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 8080
      env:
        - name: APP_COLOR
          value: pink
```

### Leveraging ConfigMaps and Secrets for Environment Variables

Instead of hardcoding values into your pod manifest, you can enhance flexibility and security by referencing external configuration sources such as ConfigMaps or Secrets. This approach simplifies maintenance and helps protect sensitive information.

To define an environment variable directly, use:
```
env:
  - name: APP_COLOR
    value: pink
```

To reference a ConfigMap for the environment variable:

```
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: color
```

To source the environment variable from a Secret

```
env:
  - name: APP_COLOR
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: color
```

### Note

- Using ConfigMaps and Secrets promotes better security practices and easier management of configuration drift. Ensure these objects are updated consistently with your application requirements.
- Avoid hardcoding sensitive data directly into your manifests. Always use Secrets when dealing with sensitive information such as passwords or API keys.


### Configure ConfigMaps in Applications

There are two main approaches to create ConfigMaps in Kubernetes: the imperative method and the declarative method.

#### Imperative Approach

If you prefer using the command line without a definition file, you can create a ConfigMap directly by specifying key–value pairs. For example, create a ConfigMap named `app-config` with specific environment variables:

```
kubectl create configmap app-config \
--from-literal=APP_COLOR=blue \
--from-literal=APP_MOD=prod
```

You can also generate a ConfigMap from a file using the `--from-file` option. For instance, if you have a file named `app_config.properties` containing configuration data:
```
kubectl create configmap app-config --from-file=app_config.properties
```

#### Declarative Approach

With a declarative approach, you define your ConfigMap in a YAML file and apply it with kubectl. Here is an example ConfigMap definition:
```
apiVersion: v1
kind: ConfigMap
metadata:
    name: app-config
data:
    APP_COLOR: blue
    APP_MODE: prod
```

Traditionally, environment variables are set directly within the pod specification. For example:

```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
    - name: simple-webapp-color
      image: simple-webapp-color
      ports:
        - containerPort: 80
  env:
    - name: APP_COLOR
      value: blue
    - name: APP_MODE
      value: prod
```

Below is an example pod definition that injects the `app-config` ConfigMap into the container as environment variables:

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
    - configMapRef:
        name: app-config
```

Below is an example pod definition that injects specific env `APP_COLOR` from the `app-config` ConfigMap into the container as environment variables:
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
      env:
        - name: APP_COLOR
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_COLOR
```

### Injection Methods

In addition to using `envFrom`, there are other methods to inject configuration data from ConfigMaps into your pods. You can inject a single environment variable using the `valueFrom` property or mount the entire ConfigMap as a volume. For example:

```
env:
  - name: APP_COLOR
    value: blue
  - name: APP_MODE
    value: prod

# You want ALL keys in the ConfigMap injected as environment variables
envFrom:
  - configMapRef:
      name: app-config

# You want specific keys only
env:
  - name: APP_COLOR
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_COLOR

# Your app expects config files
volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```

**Always validate the names and key references in your ConfigMaps to ensure that your pods load the correct configuration data.**

