# Kustomize Problem Statement idealogy

Before exploring what Kustomize is and how to use it, lets examines the challenges it addresses and explains the motivation behind its creation.

## The Traditional Approach

Consider a simple example of a single NGINX deployment defined in a YAML file. This deployment creates one NGINX pod:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

Imagine you have multiple environments like development, staging, and production. You might require the same deployment to behave differently in each environment. For example, on a local development machine you need only one replica, staging might require two or three, and production could need five or more.

A common solution is to duplicate the YAML file into individual directories for each environment and then modify environment-specific parameters (such as replica counts). For instance:

```yaml  
# dev/nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

```yaml  
# stg/nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

```yaml  
# prod/nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

To apply these configurations, you would run a command like:

```bash  
$ kubectl apply -f dev/
deployment.apps/nginx-deployment created
```

And similarly for the staging and production directories. While this approach can work for a handful of resources, it becomes both tedious and error-prone as more resources are added. Every new resource (like a service defined in `service.yaml`) must be copied across every environment directory. Over time, this leads to mismatched configurations and unnecessary maintenance overhead.

>  To maintain consistency and reduce redundancy, it is essential to preserve a single source of truth for configurations, modifying only what is needed per environment.

## The Need for a Better Approach

The key challenge is to reuse Kubernetes configurations while only modifying what differs by environment. Instead of duplicating elaborate configuration files for each environment, a solution is needed to treat configurations like code with a central base configuration and specific layers of changes applied on top.

## Enter Kustomize

Kustomize provides an elegant solution by introducing two key components: Base configuration and Overlays.

### Base Configuration

The Base configuration contains resources common to all environments. It represents default values that every environment uses unless explicitly overridden. For example, consider the following base NGINX deployment with one replica by default:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
```

### Overlays

Overlays allow customization of the base configuration for each environment. Each overlay—whether for development, staging, or production—defines changes specific to that environment. In a development overlay, the default configuration might remain unchanged, while staging and production overlays could modify the replica count to suit their requirements.

### Recommended Folder Structure

Kustomize recommends the following folder structure to organize configurations:

```text  
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── nginx-depl.yaml
│   ├── service.yaml
│   └── redis-depl.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── config-map.yaml
    ├── stg/
    │   ├── kustomization.yaml
    │   └── config-map.yaml
    └── prod/
        ├── kustomization.yaml
        └── config-map.yaml
```

In this structure, the `base` directory holds the common configuration, while the `overlays` directory contains subdirectories for each environment with their respective configuration adjustments.

### How Kustomize Works

Kustomize takes the Base configuration and the overlays to produce a final, environment-specific Kubernetes manifest that you can apply to your cluster. One major benefit is that Kustomize is built into kubectl, eliminating the need for additional installation (though you can opt to install a newer version if desired).

Unlike templating systems such as Helm, Kustomize operates directly on standard YAML files no templating language to learn. This results in configuration files that are simple, easy to validate, and maintain.

### Simplicity and Scalability

By separating the common configuration (Base) from environment-specific modifications (Overlays), Kustomize streamlines application deployment. Each overlay contains only the necessary changes for its environment, reducing duplication and minimizing errors during updates.

Kustomize embraces simplicity by keeping every artifact as plain YAML no extra templating syntax or processing overhead is required. This advantage is pivotal for efficiently managing Kubernetes deployments across various environments.

Kustomize offers a scalable and maintainable way to manage environment-specific configuration changes without duplicating entire sets of configuration files. Its use of base configurations and overlays minimizes errors and streamlines the deployment process.

# Kustomize vs Helm


## Helm Project Structure

A well-organized Helm project typically separates configuration files based on the target environment. Below is an example directory structure that demonstrates how to arrange your Helm charts and values files:

```plaintext  
k8s/
└── Deployment.yaml
└── environments/
    ├── values.dev.yaml
    ├── values.stg.yaml
    └── values.prod.yaml
└── templates/
    ├── nginx-deployment.yaml
    ├── nginx-service.yaml
    ├── db-deployment.yaml
    └── db-service.yaml
```

* The **templates** directory contains Kubernetes manifest files that include Go templating syntax.
* The **environments** directory includes various `values.yaml` files tailored for development, staging, and production.

When deploying your application, you select the appropriate values file based on the target environment, and Helm injects these values into the templates accordingly.

## Additional Helm Features

Helm is more than just a templating tool—it is a powerful package manager for Kubernetes applications, offering capabilities similar to those found in Linux package managers like yum or apt. Key advanced features include:

* Conditionals and loops within templates
* Built-in functions to manipulate and format data
* Lifecycle hooks to manage application deployment events

> Helm charts are rendered using Go templating syntax, meaning they are not valid YAML until processed. This can make the charts more challenging to read compared to plain Kubernetes YAML files.

In contrast, Kustomize uses straightforward YAML overlays, making it simpler and often more readable. However, the simplicity of Kustomize comes with less flexibility than Helm's advanced feature set.

Choosing between Kustomize and Helm depends on your project's specific requirements, your team's expertise, and the level of complexity you're prepared to manage. Helm’s powerful templating, advanced features, and environment-specific configuration options make it an excellent choice for complex deployments, while Kustomize offers a simpler, more transparent approach using plain YAML.



# Kustomize Output

## Deploying Configurations Using Pipes

To deploy the generated configurations, you can use the Linux pipe utility. This allows you to feed the output of the Kustomize build directly into the Kubernetes API using the `kubectl apply` command. Here’s an example:

```bash  
kustomize build k8s/ | kubectl apply -f -
```

In this command, the pipe (`|`) directs the output from `kustomize build k8s/` into `kubectl apply -f -`, thus creating the nginx deployment and its corresponding service.

Alternatively, you can accomplish the same deployment natively with kubectl using the `-k` flag. This flag tells kubectl to locate and process the `kustomization.yaml` file in the specified directory:

```bash  
kubectl apply -k k8s/
```

Delete the resources using the pipe method by running:

```bash  
kustomize build k8s/ | kubectl delete -f -
```

Or, delete the resources natively with kubectl:

```bash  
kubectl delete -k k8s/
```

>  Both methods are effective, so you can choose the one that best fits your workflow. The native kubectl approach with the `-k` flag simplifies the process by eliminating the need to use a pipe.

# Managing Directories

We explore how to efficiently manage Kubernetes manifests spread across multiple directories using Kustomize. Up to now, you have seen only a basic example with a simple kustomization.yaml file. However, even with limited knowledge, you can leverage powerful features in Kustomize to better organize your configurations.

Consider a scenario where you have a directory named "k8s" containing four YAML files:

* API deployment
* API service
* Database deployment
* Database service

Initially, you might deploy these configurations with the standard Kubernetes command:

```bash
kubectl apply -f k8s/
```

As your project grows and the number of YAML files increases to 20, 30, or even 50, you might decide to organize them into subdirectories. For example, you could move the API deployment and service YAML files into an "api" subdirectory and the database configurations into a "db" subdirectory. After reorganizing, you would deploy each set of configurations separately:

```bash
kubectl apply -f k8s/api/
kubectl apply -f k8s/db/
```

> With multiple subdirectories, managing deployments can become cumbersome. Every change might require separate commands for each subdirectory and even adjustments to your CI/CD pipeline.

This is where Kustomize proves its worth. You can create a root kustomization.yaml file in your "k8s" directory that lists every resource by its relative path:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Kubernetes resources to be managed by Kustomize
resources:
  - api/api-depl.yaml
  - api/api-service.yaml
  - db/db-depl.yaml
  - db/db-service.yaml
```

With this configuration, you can deploy all resources with a single command:

```bash
kustomize build k8s/ | kubectl apply -f -
```

Alternatively, you can leverage kubectl's native support for Kustomize:

```bash
kubectl apply -k k8s/
```

> While the above approach works, it can get unwieldy if the number of directories increases. A modular strategy with separate kustomization.yaml files in each subdirectory helps to keep the configuration clean and maintainable.

Imagine expanding your project to include additional components, such as a cache and Kafka. Your root kustomization.yaml file might then list resources from API, database, cache, and Kafka directories as follows:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Kubernetes resources to be managed by Kustomize
resources:
  - api api/api-service.yaml
  - db/db-depl.yaml
  - db/db-service.yaml
  - cache/redis-depl.yaml
  - cache/redis-service.yaml
  - cache/redis-config.yaml
  - kafka/kafka-depl.yaml
  - kafka/kafka-service.yaml
  - kafka/kafka-config.yaml
```

This file can quickly become cluttered and hard to maintain. A better solution is to add a separate kustomization.yaml file within each subdirectory. For example, in the "db" directory, the kustomization.yaml might look like this:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - db-depl.yaml
  - db-
Similarly, you should create kustomization.yaml files in the "api", "cache", and "kafka" directories. Then, update your root kustomization.yaml file to only reference these subdirectories:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - api/
  - db/ kafka/
```

When you run either of the following commands, Kustomize will automatically locate each subdirectory's own kustomization.yaml file and build a combined configuration:

```bash
kustomize build k8s/ | kubectl apply -f -
```

or

```bash
kubectl apply -k k8s/
```

This modular approach keeps your root kustomization.yaml file clean and simplifies the deployment process across multiple directories.


# Common Transformers

We explore how to efficiently modify your Kubernetes configurations using Kustomize's built-in transformers. Kustomize allows you to apply common configuration changes across multiple YAML files whether by adding labels, modifying resource names, setting namespaces, or applying annotations—without editing each file manually.

Below, we outline the challenges these transformers solve and provide detailed examples of several common transformations.

***

## The Problem

Imagine working with multiple Kubernetes resource files, such as a Deployment and a Service. Consider the following initial configuration files:

```yaml  
# db-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

```yaml  
# db-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  selector:
    component: db-depl
  ports:
    - protocol: "TCP"
      port: 27017
      targetPort: 27017
  type: LoadBalancer
```

Suppose you want to apply a common configuration change like adding a label (for example, `org: example`) or appending a suffix (such as `-dev`) to resource names—across all these files. Manually updating each file in a production environment with numerous YAML files can be time-consuming and error-prone. This is where Kustomize's transformers become extremely useful.

### 1. Common Label Transformation

Instead of manually updating each resource, you can add a common label to all Kubernetes resources via your kustomization file. 

```yaml  
# Kustomization.yaml
commonLabels:
  org: example
```

This change automatically appends the label to all imported resources.

### 2. Name Prefix/Suffix Transformation

If you need to append a suffix (like `-dev`) to the names of all your resources, you can adjust your resource file names manually or delegate the change to Kustomize.

```yaml  
# Kustomization.yaml
namePrefix: example-
nameSuffix: -dev
```

Kustomize will automatically modify the names of all imported resources to include the specified prefix and suffix.

### 3. Namespace Transformation

The namespace transformer enables you to group all your Kubernetes resources under a designated namespace. Instead of individually editing each file, simply modify the metadata within your YAML.

This method ensures that the Service is deployed in the specified namespace (`lab` in this case).

```yaml
# Kustomization.yaml
namespace: lab
```

### 4. Common Annotations Transformation

When you need to apply common annotations across all your Kubernetes resources, Kustomize makes it easy. For example, to add an annotation like `branch: master`, declare it in your kustomization file:

```yaml  
# Kustomization.yaml
commonAnnotations:
  branch: master
```

With this setup, every resource managed by this kustomization automatically receives the annotation.

#### 5. Using an Image Transformer

The image transformer is a powerful feature that enables you to modify container image references directly in your configuration files. Suppose the current database deployment uses the Mongo image and you want to replace it with Postgres. Open the kustomization.yaml file in the database folder and add the following image transformer section:

```yaml
# Kustomization.yaml
images:
  - name: mongo
    newName: postgres
    newTag: "4.2"
```

After applying the image transformation, the final rendered output for the database deployment will display the updated container image:

```yaml
...
containers:
  - name: mongo
    image: postgres:4.2
...
```

# Patches 

Patches offer a granular approach to updating specific sections of a Kubernetes resource, which is especially useful when you want to target one or a few objects rather than applying a broad change. For instance, if you need to update the replica count in a Deployment, a tailored patch that specifically addresses that object is ideal.

## Patch Parameters

When creating a patch, you must specify three key parameters:

1. **Operation Type:**\
   This parameter defines the type of change to perform. The primary operations include:

   * **add:** Adds an element to a list (e.g., adding a container to a Deployment’s container list).
   * **remove:** Deletes an element from the configuration (e.g., removing a container or label).
   * **replace:** Substitutes an existing value with a new one (e.g., updating a Deployment's replica count from 5 to 10).

2. **Target:**\
   The target parameter specifies which Kubernetes resource or resources will be patched. You can match resources based on properties such as kind, version, name, namespace, label selectors, or annotation selectors. Multiple properties can be combined to precisely target the intended objects.

3. **Value:**\
   For **add** or **replace** operations, this parameter defines the value to be added or used as a replacement. The **remove** operation does not require a value since it deletes the specified element.

## Inline Patch Example

Consider the following example where we have a Deployment defined in `deployment.yaml`. We aim to change the deployment name from "api-deployment" to "web-deployment". Below is the original Deployment configuration:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
      - name: nginx
        image: nginx
```

To perform the update, define the patch in your `kustomization.yaml` file under the `patches` property. The patch below specifies the target object (a Deployment with the name "api-deployment") and provides an inline patch to replace the `metadata/name` field:

```yaml  
# kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /metadata/name
        value: web-deployment
```

After applying this patch, the rendered Deployment configuration becomes:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
      - name: nginx
        image: nginx
```

## Replacing the Replica Count

Next, let’s update the number of replicas. Assume the original Deployment configuration has one replica and you want to change it to five. 

```yaml  
patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
```

## Patch Strategies: JSON 6902 vs. Strategic Merge Patch

There are two primary patch strategies in Kustomize: JSON 6902 patches and strategic merge patches.

> JSON 6902 patches require you to specify both the target resource and detailed patch operations, ensuring precise modifications.

### JSON 6902 Patch

A JSON 6902 patch involves providing the target object and a list of operations including the patch details such as the operation (e.g., replace), the path of the property, and the new value. Here’s an example:

```yaml
customization:
  patches:
    - target:
        kind: Deployment
        name: api-deployment
      patch:
        - op: replace
          path: /spec/replicas
          value: 5
```

### Strategic Merge Patch

A strategic merge patch resembles a standard Kubernetes configuration file. It merges your partial configuration into the existing one, updating only the specified fields. Below is the equivalent strategic merge patch for the replica count change:

```yaml
customization:
  patches:
    - patch: |-
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: api-deployment
        spec:
          replicas: 5
```

Both methods are effective; the choice depends on your preference. Strategic merge patches tend to be more readable and familiar for those used to standard Kubernetes configurations, while JSON 6902 patches offer higher precision for targeted modifications.

## Key Takeaways

| Feature     | JSON 6902 Patch                            | Strategic Merge Patch                             |
| ----------- | ------------------------------------------ | ------------------------------------------------- |
| Usage       | Detailed operations on a specific resource | Merges a partial config into an object            |
| Format      | List of operations with precise paths      | Partial YAML structure matching original resource |
| Readability | Less intuitive                             | More intuitive and resembles a standard config    |

By understanding these patching techniques, you can tailor your Kubernetes configurations to meet specific requirements with the appropriate level of granularity. This flexibility in patching ensures that your deployments are both maintainable and scalable.

# Different Types of Patches

Two methods to define patches within your kustomization configuration. By using either JSON 6902 patches or strategic merge patches, you can choose to embed your patches inline in your kustomization.yaml file or reference an external file that contains the patch definitions.

## Inline Patching

The inline method involves embedding the patch directly into the kustomization.yaml file. This approach is ideal when you have a small number of patches, keeping your configuration straightforward. For instance, an inline patch to modify the replica count for a Deployment might look like this:

```yaml
# Kustomization (inline method)
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

> Inline patching is best suited for simple and minimal modifications, ensuring quick adjustments without the need for managing multiple files.

## Separate File Patching

If your kustomization.yaml file becomes overly cluttered with many patch definitions, consider storing your patches in a separate file. This approach enhances maintainability and keeps your main configuration file clean. In your primary kustomization.yaml file, reference the external patch file as shown below:

```yaml
# Kustomization (separate file reference)
patches:
  - path: replica-patch.yaml
    target:
      kind: Deployment
      name: api-deployment
```

```yaml
# replica-patch.yaml
- op: replace
  path: /spec/replicas
  value: 5
```

Within the external file (replica-patch.yaml), you can list all the necessary modifications for your deployment. This method is especially useful for larger projects where organizing patches into dedicated files improves clarity.

## Strategic Merge Patch Example

Strategic merge patches are an alternative to JSON 6902 patches and can also be defined inline or referenced via an external file. Below are examples demonstrating both methods.

### Inline Strategic Merge Patch

```yaml  
# Inline strategic merge patch
patches: |
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: api-deployment
      spec:
        replicas: 5
```

### Separate File Reference for Strategic Merge Patch

```yaml  
# Separate file reference for strategic merge patch
patches:
  - replica-patch.yaml
```

```yaml
# replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: api-deployment
spec:
    replicas: 5
```

>  For both patching methods, choose inline patches when dealing with a few simple modifications. For extensive configurations, using separate files can greatly enhance manageability and reduce complexity.


# Patches Dictionary


Let's learn how to update, add, and remove keys in a Kubernetes deployment configuration using both JSON 6902 patches and strategic merge patches. The step-by-step examples provided below will help you understand the process and decide which patching method best fits your configuration management needs.

## Updating a Key with a JSON 6902 Patch

Assume you have a deployment configuration with a label `component: api` that needs to be updated to `component: web`. First, consider the original deployment configuration:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

To update this label using a JSON 6902 patch, add the following patch to your `kustomization.yaml`:

```yaml  
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/template/metadata/labels/component
        value: web
```

>  The patch uses the "replace" operation to update the existing key at the specified path (`/spec/template/metadata/labels/component`) with the new value `web`.

## Updating a Key with a Strategic Merge Patch

You can achieve the same update using a strategic merge patch. Instead of including the patch inline, store it in a separate file named `label-patch.yaml` and reference it in your `kustomization.yaml`.

In your `kustomization.yaml`, include:

```yaml  
patches:
  - label-patch.yaml
```

Then, create `label-patch.yaml` with only the fields that need to be updated:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        component: web
```

Kustomize will merge this patch with the original configuration, updating only the `component` label value from `api` to `web`.

## Adding a New Key Using a JSON 6902 Patch

Suppose you want to add a new label `org: Example` to the existing deployment configuration. Consider the following original configuration:

```yaml  
# api-depl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

Now, add the following JSON 6902 patch in your `kustomization.yaml`:

```yaml  
# kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: add
        path: /spec/template/metadata/labels/org
        value: Example
```

In this patch:

• The **add** operation instructs Kubernetes to introduce a new key.\
• The **path** defines where the new label should be inserted (`/spec/template/metadata/labels/org`).\
• The **value** `Example` is the new label value.

After applying the patch, the final configuration will include both labels—`component: api` and `org: Example`:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
        org: Example
    spec:
      containers:
        - name: nginx
          image: nginx
```

## Adding a New Key with a Strategic Merge Patch

To add a new label using a strategic merge patch, reference an external patch file. In your `kustomization.yaml`, include:

```yaml  
patches:
  - label-patch.yaml
```

Then, in the `label-patch.yaml` file, specify the new key:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: Example
```

Kustomize will merge this patch with the original configuration, resulting in the addition of the new `org` label.


## Removing a Key Using a JSON 6902 Patch

Consider a deployment that currently has two labels: `org: Example` and `component: api`. Here is the existing configuration:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        org: Example
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

To remove the `org` label, apply the following JSON 6902 patch:

```yaml  
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: remove
        path: /spec/template/metadata/labels/org
```

This patch uses the "remove" operation to delete the `org` label. After merging, only the `component: api` label remains.


## Removing a Key with a Strategic Merge Patch

With a strategic merge patch, you can remove a key by setting its value to `null`. Starting with the same configuration as above, reference an external patch file in your `kustomization.yaml`:

```yaml  
patches:
  - label-patch.yaml
```

Then, in `label-patch.yaml`, mark the `org` label for removal:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    metadata:
      labels:
        org: null
```

Kustomize merges this patch to remove the `org` label, leaving only the `component: api` label in the configuration.


# Patches list

Let's learn how to modify containers within a Kubernetes Deployment by performing operations on list items. We cover how to replace, add, and delete items from the container list using both JSON 6902 patches and strategic merge patches. Every example maintains the correct YAML hierarchy and list indexing to ensure a seamless configuration update.

Below is the base deployment configuration used in all examples:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

## 1. Replacing a Container Using a JSON 6902 Patch

In this first example, we update the container’s name and image from "nginx" to "haproxy". Since the container section is defined as a list, we specifically target the first container (index 0) in the patch path. Remember that list indexes start at zero.

The following JSON 6902 patch replaces the container at index 0:

```yaml  
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0
        value:
          name: haproxy
          image: haproxy
```

## 2. Replacing a Container Using a Strategic Merge Patch

An alternative approach is to update the container using a strategic merge patch. In this example, we update the image of an existing container. Verify that the Deployment configuration and the patch file refer to the same container name. Note that a minor typo ("ngin" instead of "nginx") has been corrected for clarity.

The strategic merge patch, defined in a file (e.g., `label-patch.yaml`), looks like this:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
        - name: nginx
          image: haproxy
```

In your `kustomization.yaml`, reference the patch file:

```yaml  
patches:
  - label-patch.yaml
```

## 3. Adding an Item to a List Using a JSON 6902 Patch

To add another container to your Deployment, use the JSON patch with an "add" operation. The patch path ends with a dash (-) to signal that the new container should be appended to the end of the list. Although indexing (e.g., 0 for the beginning) is possible, appending is achieved easily using the dash notation.

The base deployment configuration is as follows:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: nginx
          image: nginx
```

The JSON 6902 patch to add a new container is:

```yaml  
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: add
        path: /spec/template/spec/containers/-
        value:
          name: haproxy
          image: haproxy
```

## 4. Adding an Item to a List Using a Strategic Merge Patch

You can also add a container by merging configuration files. In this approach, the original Deployment configuration is combined with a patch file. In this example, the initial Deployment includes a placeholder container named "web" with the image "nginx":

The patch file (e.g., `label-patch.yaml`) adds an additional container:

```yaml  
# label-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
        - name: haproxy
          image: haproxy
```

Reference the patch in your `kustomization.yaml`:

```yaml  
# kustomization.yaml
patches:
  - label-patch.yaml
```

After applying the merge, the Deployment will feature two containers: one named "web" using the image "nginx" and another named "haproxy" using the image "haproxy".

>  Using strategic merge patches lets you combine multiple configuration files effortlessly while keeping your Deployment definitions clean and modular.

## 5. Deleting an Item from a List Using a JSON 6902 Patch

If you need to remove a container from your Deployment, such as one named "database", a JSON 6902 patch can be used to delete it by specifying its index in the list. In the example below, the "web" container is at index 0, and the "database" container is at index 1.

The original configuration:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: api
  template:
    metadata:
      labels:
        component: api
    spec:
      containers:
        - name: web
          image: nginx
        - name: database
          image: mongo
```

The JSON 6902 patch to remove the container at index 1 is:

```yaml  
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: remove
        path: /spec/template/spec/containers/1
```

## 6. Deleting an Item from a List Using a Strategic Merge Patch

Removing a container using a strategic merge patch is achieved by adding a delete directive in your patch file. In this example, we remove the container with the name "database".

The strategic merge patch file (e.g., `label-patch.yaml`) includes the `$patch: delete` directive:

```yaml  
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
        - $patch: delete
          name: database
```

Add this patch to your `kustomization.yaml`:

```yaml  
patches:
  - label-patch.yaml
```

>  Double-check list indexes and container names when applying deletion patches to avoid removing the wrong configuration.


# Overlays

Let's see how Kustomize can be used to tailor a base Kubernetes configuration for different environments such as development, staging, and production. In this guide, we'll walk through the primary use case of Kustomize and illustrate how to implement environment-specific customizations using overlays.

A typical Kustomize project is organized into two main sections:

1. **Base Configuration** – Contains all shared configurations applicable across environments.
2. **Overlays** – Contains environment-specific customizations that are applied as patches or additional resources.

Consider the following directory structure:

```text
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── nginx-depl.yaml
│   ├── service.yaml
│   └── redis-depl.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── config-map.yaml
    ├── stg/
    │   ├── kustomization.yaml
    │   └── config-map.yaml
    └── prod/
        ├── kustomization.yaml
        └── config-map.yaml
```

In this example, the base folder holds all the default Kubernetes configurations, while each overlay folder (dev, stg, prod) includes custom configurations specific to their respective environments.

## How Overlays Work

To customize your base configuration for a specific environment, you define patches within the overlay. For instance, suppose you have a base deployment file for nginx (`nginx-depl.yaml`) that specifies a replica count of 1. To adjust the replica count for the development environment, you can define a kustomization.yaml file in the dev overlay folder as follows:

```yaml
bases:
  - ../../base
patch: |-
  - op: replace
    path: /spec/replicas
    value: 2
```

The corresponding base deployment configuration remains:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
```

Here, the `bases` property points to the relative path of the base configurations. The provided patch updates the `replicas` value to 2 for the development environment. The notation `../../base` indicates that from the dev folder, you move up two levels (from dev to overlays, and from overlays to k8s), then enter the base directory.

For the production environment, you might require a higher replica count. The kustomization.yaml in the production overlay could look like this:

```yaml
bases:
  - ../../base
patch: |-
  - op: replace
    path: /spec/replicas
    value: 3
```

> Overlays can be used to both modify existing configurations and to add new resources that are unique to an environment. This makes it easy to manage environment-specific differences without duplicating configuration files.

## Adding Unique Resources

Overlays are not limited to patching the base configuration. In your overlay folders, you can also introduce additional resources that do not exist in the base. For example, if the production environment requires a Grafana deployment, you can add it using the following kustomization.yaml:

```yaml
bases:
  - ../../base
resources:
  - grafana-depl.yaml
patch: |-
  - op: replace
    path: /spec/replicas
    value: 2
```

In this configuration, not only is the replica count updated for an existing deployment, but an additional resource (`grafana-depl.yaml`) is also imported specifically for the production environment.

> Ensure that all relative paths in your kustomization.yaml files are correctly defined. An incorrect path can lead to configuration import errors which can impact your deployment.

## Flexible Directory Structure

Kustomize offers significant flexibility in organizing your Kubernetes configurations. You are not limited to a fixed directory layout; you can structure your base and overlay directories into subfolders by features or any other scheme that suits your project. The key is ensuring that the resources are correctly referenced in the kustomization.yaml files.


# Components

Components allow you to define a reusable block of configuration logic that can be included in multiple overlays. This approach is particularly beneficial when an application supports optional features that should only be enabled in certain overlays instead of globally. By centralizing feature-specific configurations—such as Kubernetes resources, patches, config maps, and secrets—you reduce duplication and prevent configuration drift.

> Using components provides a single source of truth for your features, ensuring that updates or changes propagate consistently across all overlays where the feature is enabled.

## When to Use Components

Imagine a scenario where you have a set of configurations to enable a specific feature. If this feature were meant for all overlays, including the configuration in the Base might be appropriate. However, when the feature should only be activated in a subset of overlays, manually duplicating the configuration leads to scalability issues and potential errors. Components solve this problem efficiently by maintaining a centralized configuration that you can import wherever necessary.

Consider an application that can be deployed in three variations: development, premium, and self-hosted. The Base configuration holds common settings, while each variation is represented by its own overlay folder. Suppose the application has two optional features:

1. **Caching**: Only the premium and self-hosted versions require caching, which involves deploying a Redis instance along with its configurations and secrets.
2. **External Database**: A PostgreSQL database is enabled only for the development and premium versions.

Including the caching configuration directly in the Base would apply it to all overlays unintentionally, and duplicating the configuration across multiple overlays is error-prone. Components allow you to write the configuration once and reuse it across the specific overlays where it's needed.

## Organizing Components

A well-organized project structure simplifies management. Consider the following directory layout:

```text  theme={null}
k8s/
├── base/
│   ├── kustomization.yaml
│   └── api-depl.yaml
├── components/
│   ├── caching/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── redis-depl.yaml
│   └── db/
│       ├── kustomization.yaml
│       ├── deployment-patch.yaml
│       └── postgres-depl.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── premium/
    │   └── kustomization.yaml
    └── standalone/
        └── kustomization.yaml
```

In this setup:

* **Base** contains shared configurations.
* **Overlays** (dev, premium, standalone) inherit from the Base.
* **Components** store isolated configurations for individual features, such as caching and external database (db). Each component directory includes all the necessary Kubernetes resources to enable the feature.

Once organized into components, overlays simply import the desired component by referencing its path within the kustomization configuration. This method minimizes duplication and streamlines updates across all relevant overlays.

## Implementing a Kustomize Component

Let's walk through an example by implementing the database component. In the **db** folder, you will find the following files:

* **postgres-depl.yaml**: Defines the deployment resource for a PostgreSQL database.
* **deployment-patch.yaml**: Contains a strategic merge patch to update the base deployment with database-specific configurations.
* **kustomization.yaml**: Declares the component by importing resources, generating secrets, and applying patches.

### Postgres Deployment Configuration

The `postgres-depl.yaml` file creates a PostgreSQL deployment:

```yaml  theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      containers:
        - name: postgres
          image: postgres
```

### Component kustomization.yaml

In the component’s `kustomization.yaml`, a different API version and kind indicate that this is a component. It imports the PostgreSQL deployment, generates a secret for the database password, and applies patches:

```yaml  theme={null}
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - postgres-depl.yaml
secretGenerator:
  - name: postgres-cred
    literals:
      - password=postgres123
patches:
  - deployment-patch.yaml
```

### Deployment Patch

The `deployment-patch.yaml` file is a strategic merge patch that updates the Base deployment by adding an environment variable for the database password. This integration ensures that the Base application can securely access the database credentials when the feature is enabled:

```yaml  theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: api
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-cred
              key: password
```

### Overlay Configuration

To integrate the database component into an overlay such as the development overlay—the overlay’s `kustomization.yaml` file should reference both the Base configuration and the specific component:

```yaml
bases:
  - ../../base
components:
  - ../../components/db
```

By listing the component path under the `components` section, the overlay effortlessly incorporates the external database feature without duplicating any configuration.

> Always ensure that the paths in your overlay configurations accurately reference the component directories to prevent any configuration errors during deployment.
