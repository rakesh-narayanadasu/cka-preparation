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

