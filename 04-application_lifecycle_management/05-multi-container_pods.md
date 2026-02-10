# Multi Container Pods

Multi-container pods are designed to group containers that **share the same lifecycle**. This means they are created and terminated together, share a common network namespace (allowing seamless communication via localhost), and have access to shared storage volumes. This design simplifies configurations by eliminating the complexities of volume sharing and networking between separate pods.

For example, a web server might need to be paired with a dedicated logging agent.

The following YAML snippet demonstrates how to configure a pod that contains both a web application and its corresponding logging agent:
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
    - name: log-agent
      image: log-agent
```

This configuration ensures that both containers share the same lifecycle, network, and storage resources, allowing them to work together seamlessly.


## Multi Container Pod Design Patterns

1) Co-located Containers
Both the containers are meant to **continue to run throughout the entire pod lifecycle**. These are usually used when two services are dependent on each other.

2) Init Containers
This is used when there are initialization steps to be performed when a **pod starts before the main application itself**. For example, this could be an init container that waits for a database to be ready before starting the main application. The init container does its job and ends its job, and then the main application starts.

The init container is differentiated well because we know that in terms of the init container, it starts and then stops, and then the main app runs.

3) SideCar Containers
A sidecar container is set up like an init container where the **sidecar starts first, does its job, but instead of ending its run, it continues to run throughout the life cycle of the pod**. 

The main application starts after the sidecar container starts. This is useful when you have a log shipper of sorts that needs to be run, along with the main application that needs to start before the main app starts, but continue to run as long as the main app runs and then end after the main app ends.


- In case of Both starts together and there is no guarantee that one would start before the other. However, the Sidecar Containers option provides the ability to specify an order of startup and then continue to run throughout the pod lifecycle.

```
# colocated-containers.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: web-app
      image: web-app
      ports:
        - containerPort: 8080
    - name: main-app
      image: main-app
```
If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.
```
#init-container.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: web-app   # third the main contianer runs
      image: web-app
      ports:
        - containerPort: 8080
  initContainers:
    - name: db-checker    # first db-checker container runs and ends
      image: busybox
      command: 'wait-for-db-to-start.sh'
    - name: api-checker   # second api-checker container runs and ends
      image: busybox
      command: 'wait-for-another-api.sh'
```

```
# sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: web-app
      image: web-app
      ports:
        - containerPort: 8080
  initContainers:
    - name: log-shipper
      image: busybox
      command: 'setup-log-shipper.sh'
      restartPolicy: Always        
```

The init container continues to run as it has a restart policy set to always. So this will also ensure the init container is terminated after the main application stops. That way the log shipper can catch the startup and termination logs of the main container.


# Self Healing Applications

Kubernetes supports self-healing applications through ReplicaSets and Replication Controllers. The replication controller helps in ensuring that a POD is re-created automatically when the application within the POD crashes. It helps in ensuring enough replicas of the application are running at all times.

Kubernetes provides additional support to check the health of applications running within PODs and take necessary actions through Liveness and Readiness Probes. 

## Readiness Probes

When a pod is first created, it enters a Pending state while the scheduler selects an appropriate node for placement. If no node is immediately available, the pod stays in the Pending state. Use kubectl describe pod to investigate any delays. After scheduling, the pod transitions to the ContainerCreating state as images are pulled and containers begin their startup process. Once all containers launch successfully, the pod moves into the Running state and remains there until the application completes execution or is terminated.

For more detailed insights, inspect the pod’s conditions. Initially, when a pod is scheduled, the “PodScheduled” condition becomes true. Following initialization, the “Initialized” condition is set to true, and finally, once all containers are ready, the “ContainersReady” condition is affirmed. When all these conditions are true, the pod is officially considered ready.

Examine these conditions by running:
```
kubectl describe pod <pod-name>
```

For example:

- A minimal script might be ready within milliseconds.
- A database service may take several seconds to become responsive.
Complex web servers could require minutes to fully initialize.
- A Jenkins server, for instance, might take 10 to 15 seconds to initialize followed by a brief warm-up period. Without suitable readiness checks, traffic might be erroneously routed to an unready pod.

Kubernetes relies on the pod’s ready condition to determine if it should receive traffic. By default, it assumes that container creation equates to readiness. This is where readiness probes are critically important—they help signal the true readiness of the application within the container.

As an application developer, you can define tests (probes) that confirm the application has reached a ready state. Examples include:
- An HTTP GET request to check API responsiveness for a web application.
- A TCP port check for database services.
- Executing a custom command that confirms the application is ready.

To configure a readiness probe, include the readinessProbe field in your pod specification. For an HTTP-based readiness probe, specify the endpoint path and port. For example:
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 8
```

There are three primary types of readiness probes you can use:

1) HTTP GET Probe:

```
readinessProbe:
  httpGet:
    path: /api/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 8
```

- initialDelaySeconds: The delay before the first probe.
- periodSeconds: How frequently the probe is executed.
- failureThreshold: The number of consecutive failures that trigger the pod to be marked as not ready.

2) TCP Socket Probe:

```
readinessProbe:
  tcpSocket:
    port: 3306
```

3) Exec Command Probe:

```
readinessProbe:
  exec:
    command:
      - cat
      - /app/is_ready
```


## Liveness Probe

When you run an Nginx container using Docker the container starts serving users immediately. However, if the Nginx process crashes, the container will exit. Since Docker is not designed for orchestration, the container remains stopped until you manually restart it.

In contrast, running the same web application in Kubernetes provides automated resilience. If the application crashes, Kubernetes will detect the issue and automatically restart the container.

This is where liveness probes become essential. A liveness probe periodically checks the health of your application running inside the container. If the probe’s test fails, Kubernetes considers the container unhealthy and recreates it to restore service. As a developer, you define what “healthy” means for your specific application. For a web service, it might mean that the API is responsive; for a database, ensuring the TCP socket is open could be the test; or you may choose to execute a custom command for more complex scenarios.

Liveness probes are defined within your pod’s container specification in a manner similar to readiness probes but in a distinct configuration section. The available methods include:

- HTTP GET: Perform an HTTP request to check an API endpoint.
- TCP Socket: Validate that a specific TCP port is accepting connections.
- Exec Command: Run a command inside the container to verify application health.

Additionally, you can customize the behavior of the probe with parameters such as the initial delay, probe frequency, and failure thresholds.

### Configuring a Liveness Probe
Below is an example of a complete pod definition that uses an HTTP GET liveness probe for a simple web application:
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    name: simple-webapp
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /api/healthy
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 8
```

1) HTTP GET Probe:
```
livenessProbe:
  httpGet:
    path: /api/healthy
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 8
```

2) Using a TCP Socket

```
livenessProbe:
  tcpSocket:
    port: 3306
```

3) Using an Exec Command

```
livenessProbe:
  exec:
    command:
      - cat
      - /app/is_healthy
```

# Jobs

Containerized workloads are typically classified into two categories:
- Long-running workloads: For example, web servers and database applications that continue to run until they are manually stopped.
- Batch processing tasks: These execute a specific operation such as computation, image processing, data analysis, or report generation—and then terminate.

### Jobs for Batch Processing

For batch processing or large-scale data tasks, you might need multiple pods working together concurrently. Unlike ReplicaSets, which ensure a certain number of pods remain running, Kubernetes Jobs are designed to run pods until the specified task is completed successfully.

To create a Job, start with a job definition file that uses the API version `batch/v1` and kind `Job`. In the job specification, a template holds the pod definition. Here’s an example job definition that performs our math addition:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  template:
    spec:
      containers:
      - name: math-add
        image: ubuntu
        command: ['expr', '3', '+', '2']
      restartPolicy: Never
```

### Running Multiple Pods with Job Completions

In many real-world scenarios, you might require a job to run multiple pods simultaneously to process data in parallel or handle retries for failed operations. To run three pods for a single job, set the `completions` field to `3`:
```
apiVersion: batch/v1
kind: Job
metadata:
  name: math-add-job
spec:
  completions: 3
  template:
    spec:
      containers:
      - name: math-add
        image: ubuntu
        command: ['expr', '3', '+', '2']
      restartPolicy: Never
```

Check the job status:
```
kubectl get jobs

NAME                DESIRED   SUCCESSFUL   AGE
random-error-job    3         2            38s
```
checking the pod status:
```
kubectl get pods

NAME                     READY   STATUS      RESTARTS   AGE
math-add-job-25j9p       0/1     Completed   0          2m
math-add-job-87g4m       0/1     Completed   0          2m
math-add-job-ds295       0/1     Completed   0          2m
```
By default, pods for a job are created sequentially each pod starts only after the previous one completes.

### Handling Failures with Jobs
Consider a scenario where you use an image like kodekloud/random-error that randomly either completes successfully or fails. In such cases, if one pod fails, Kubernetes will create another pod until it achieves the specified number of successful completions. **Kubernetes continues to create new pods until three pods complete successfully**.

## Running Pods in Parallel with Jobs
For scenarios where you want the pods to run concurrently, you can set the `parallelism` property. This allows multiple pods to be created simultaneously. For example, to run up to three pods in parallel:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: random-error-job
spec:
  completions: 3
  parallelism: 3
  template:
    spec:
      containers:
      - name: random-error
        image: kodekloud/random-error
      restartPolicy: Never
```

Kubernetes will intelligently create new pods as necessary until all three completions are successful.

# Cron Jobs

A CronJob allows you to run tasks on a recurring schedule. For example, you might have a job that generates reports and sends emails. Rather than triggering the job immediately with the “kubectl create” command, you can schedule it with a CronJob to occur periodically.

### Converting a Basic Job to a CronJob
Consider the following basic Job template:
```
apiVersion: batch/v1
kind: Job
metadata:
  name: reporting-job
spec:
  completions: 3
  parallelism: 3
  template:
    spec:
      containers:
      - name: reporting-tool
        image: reporting-tool
      restartPolicy: Never
```

Below is the complete YAML definition for a CronJob that runs every minute:

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: reporting-cron-job
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      completions: 3
      parallelism: 3
      template:
        spec:
          containers:
          - name: reporting-tool
            image: reporting-tool
          restartPolicy: Never
```

To verify that your CronJob has been created correctly, run:
```
kubectl get cronjob

NAME                  SCHEDULE      SUSPEND   ACTIVE
reporting-cron-job    */1 * * * *   False     0
```

