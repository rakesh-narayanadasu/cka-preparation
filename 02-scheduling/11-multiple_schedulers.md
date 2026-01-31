# Multiple Schedulers

Kubernetes default scheduler distributes pods across nodes evenly while considering factors such as taints, tolerations, and node affinity. However, certain use cases may require a custom scheduling algorithm. For instance, when an application needs to perform extra verification before placing its components on specific nodes, a custom scheduler becomes essential. By writing your own scheduler, packaging it, and deploying it alongside the default scheduler, you can tailor pod placement to your specific needs.

Ensure that every additional scheduler has a unique name. The default scheduler is conventionally named `default-scheduler` and any custom scheduler must be registered with its own distinct name in the configuration files.

### Configuring Schedulers with YAML

```
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
```

```
# my-scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler
```

### Deploying the Custom Scheduler as a Pod

In addition to running the scheduler as a service, you can deploy it as a pod inside the Kubernetes cluster. This method involves creating a pod definition that references the scheduler’s configuration file.

```
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
    - name: kube-scheduler
      image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
      command:
        - kube-scheduler
        - --address=127.0.0.1
        - --kubeconfig=/etc/kubernetes/scheduler.conf
        - --config=/etc/kubernetes/my-scheduler-config.yaml
```

The corresponding custom scheduler configuration file might look like:

```
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler
```

**Note: Leader election is an important configuration for high-availability environments. It ensures that while multiple scheduler instances are running, only one actively schedules the pods.**

### Configuring Workloads to Use the Custom Scheduler

To have specific pods or deployments use your custom scheduler, add the "schedulerName" field in the pod's specification. For example:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
  schedulerName: my-custom-scheduler
```

