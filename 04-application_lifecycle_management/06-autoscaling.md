# Autoscaling

Kubernetes is specifically designed for hosting containerized applications and incorporates scaling based on current demands. It supports two main scaling types:

1) Workload Scaling: Adjusting the number of containers or Pods running in the cluster.

2) Cluster (Infrastructure) Scaling: Adding or removing nodes (servers) from the cluster.

When scaling in a Kubernetes cluster, consider the following:

1) Cluster Infrastructure Scaling:
- Horizontal Scaling: Add more nodes.
- Vertical Scaling: Enhance the resources (CPU, memory) of existing nodes.

2) Workload Scaling:
- Horizontal Scaling: Create additional Pods.
- Vertical Scaling: Modify the resource limits and requests for existing Pods.

### Approaches to Scaling in Kubernetes

Kubernetes supports both manual and automated scaling methods.

#### Manual scaling:

Manual scaling requires intervention from the user:

- Infrastructure Scaling (Horizontal): Provision new nodes and join them to the cluster using:

```
kubectl join ...
```

- Workload Scaling (Horizontal): Adjust the number of Pods manually with:
```
kubectl scale ...
```

- Pod Resource Adjustment (Vertical): Edit the deployment, stateful set, or replica set to modify resource limits and requests:
```
kubectl edit ...
```

#### Automated Scaling

Automation in Kubernetes simplifies scaling and ensures efficient resource management:

- **Kubernetes Cluster Autoscaler**: Automatically adjusts the number of nodes in the cluster by adding or removing nodes when needed.
- **Horizontal Pod Autoscaler (HPA)**: Monitors metrics and adjusts the number of Pods dynamically.
- **Vertical Pod Autoscaler (VPA)**: Automatically changes resource allocations for running Pods based on observed usage.



## Manual Horizontal Scaling

To monitor the pod’s resource consumption manually, you might run:
```
kubectl top pod my-app-pod
NAME         CPU(cores)   MEMORY(bytes)
my-app-pod   450m         350Mi
```

When resource usage reaches a predefined threshold (e.g., 450 mCPU), you must manually scale the deployment:

```
kubectl scale deployment my-app --replicas=3
```

Below is a sample deployment configuration for this scenario:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx
        resources:
          requests:
            cpu: "250m"
          limits:
            cpu: "500m"
```

Manually scaling pods requires continuous monitoring, which can be resource-intensive and error-prone during traffic surges.

## Automated Scaling with Horizontal Pod Autoscaler

Kubernetes simplifies scaling with the Horizontal Pod Autoscaler. HPA monitors resource metrics including CPU, memory, and custom metrics using the metrics server. When usage exceeds a defined threshold, it automatically adjusts the number of pod replicas in deployments, stateful sets, or replica sets.

When CPU or memory usage is high, HPA scales up the number of pods; when usage drops, it scales them down to conserve system resources. HPA can even track multiple metrics concurrently.

#### Imperative Creation of an HPA
For an existing Nginx deployment, you can configure an HPA with the following command. This command sets the autoscaler to maintain CPU utilization at 50% with a replica count that can vary between 1 and 10:
```
kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10
```

To check the status of your HPA, run:
```
kubectl get hpa
```

If you need to remove the autoscaler later, simply run:
```
kubectl delete hpa my-app
```

#### Declarative HPA Configuration
Alternatively, you can define the HPA using a declarative configuration file. The example below uses the autoscaling/v2 API:

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Metrics Server and Custom/External Metrics

The HPA relies on the internal metrics server for real-time CPU and memory usage data. Kubernetes also supports custom metrics adapters, which allow HPA to fetch metrics from internal cluster workloads. Additionally, external metrics adapters can integrate with tools like Datadog or Dynatrace to supply metrics from outside the cluster.


# In-Place Pod Resource

In Kubernetes version 1.32, modifying a pod’s resource requirements in a Deployment causes the **existing pod to be deleted and replaced with a new one that has the updated resource definitions**. This approach does not perform changes in place, and the termination and replacement process can be disruptive especially for **stateful** workloads.

This feature is currently in alpha (as of Kubernetes 1.27) and is not enabled by default. To use it, you must explicitly enable the feature flag “in-place Pod vertical scaling.” When enabled, additional resize policy parameters become available. For example, you can specify that changing the CPU resource does not require a pod restart, while adjusting the memory allocation does.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx
        resizePolicy:
          - resourceName: cpu
            restartPolicy: NotRequired  # change in cpu will not restart pod
          - resourceName: memory
            restartPolicy: RestartContainer
        resources:
          requests:
            cpu: "1"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

### Limitations of In-Place Resizing

While in-place resizing offers benefits, it also comes with certain limitations:

- Supported Resources: Only CPU and memory can be updated in place.
- QoS Class: The Pod Quality of Service (QoS) class cannot be modified through in-place resizing.
- Container Eligibility: Init containers and ephemeral containers are not eligible for resizing.
- Immutable Resource Positions: Once set, a container’s resource requests and limits cannot be repositioned.
- Memory Limit Constraints: A container’s memory limit cannot be reduced below its current usage. If an update attempts to lower the memory limit too far, the resize operation will remain in progress until a permissible limit is reached.
- Platform Support: Windows Pods are not supported by the in-place resizing feature at this time.


# Vertical Pod Autoscaler (VPA)

Manually updating resources can be error-prone and time-consuming. The VPA automates this process, continuously monitoring metrics and adjusting CPU and memory allocations for pods. Unlike the Horizontal Pod Autoscaler (HPA), which scales the number of pods based on demand, the VPA fine-tunes resource specifications for existing pods.

Note that the VPA is not included by default in Kubernetes clusters. You must deploy it separately from its GitHub repository. After deployment, verify its three components the recommender, updater, and admission controller using the commands below:

```
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
```
```
kubectl get pods -n kube-system | grep vpa
vpa-admission-controller-xxxx   Running
vpa-recommender-xxxx            Running
vpa-updater-xxxx                Running
```

OR

```
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

The components work as follows:

- **VPA Recommender**: Continuously monitors resource usage via the Kubernetes metrics API, aggregates historical and live data, and provides recommendations for optimal CPU and memory allocations.
- **VPA Updater**: Evaluates running pods against recommendations, evicting those with suboptimal resource requests.
- **VPA Admission Controller**: Intercepts the pod creation process and mutates pod specifications based on the recommender’s suggestions, ensuring new pods start with proper resource allocations.

### Configuring the VPA

To create a VPA for your deployment, use a YAML configuration file rather than running imperative commands. Below is an example configuration for a VPA targeting the `my-app` deployment. In this configuration, the VPA monitors CPU usage for the container named `my-app` and operates in “Auto” update mode:

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: "my-app"
        minAllowed:
          cpu: "250m"
        maxAllowed:
          cpu: "2"
        controlledResources: ["cpu"]
```

The VPA supports multiple update modes:
- Off: Only provides recommendations without any modifications.
- Initial: Applies recommendations only to newly created pods.
- Recreate: Evicts pods running with suboptimal resource allocations, leading to pod restarts.
- Auto: Currently behaves like “recreate” by evicting pods to apply updated values. In the future, auto mode may support in-place updates without restarting pods.

To view the VPA recommendations, run:
```
kubectl describe vpa my-app-vpa
Recommendations:
  Target:
    Cpu: 1.5
```

In this example, the VPA recommends increasing the CPU allocation to 1.5 cores.

```
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: flask-app
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: flask-app
  updatePolicy:
    updateMode: "Recreate"
    evictionRequirements:
      - resources: ["cpu", "memory"]
        changeRequirement: TargetHigherThanRequests
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 100Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
```

#### VPA vs HPA
VPA and HPA both aim to optimize resource utilization, but they address different aspects of scaling:

| Feature | Vertical Pod Autoscaler (VPA) | Horizontal Pod Autoscaler (HPA) |
|--------|------------------------------|----------------------------------|
| **Scaling Method** | Adjusts CPU and memory configurations for individual pods (may require pod restarts) | Scales the number of pods based on demand |
| **Pod Behavior** | Might cause downtime due to pod restarts during resource updates | Typically maintains availability by adding or removing pods on the fly |
| **Traffic Spikes** | Less effective during sudden spikes, as pod restarts are needed | More responsive to rapid traffic changes by dynamically adjusting pod count |
| **Cost Optimization** | Prevents over-provisioning by matching resource allocations with actual usage | Reduces costs by eliminating the overhead of running idle pods |
| **Ideal Use Cases** | Best suited for stateful workloads and resource-intensive applications (e.g., databases, JVM apps, AI workloads) | Ideal for stateless services like web applications and microservices where quick scaling is crucial |
