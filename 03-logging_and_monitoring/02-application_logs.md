# Managing Application Logs

Logging mechanisms help you monitor and troubleshoot your applications effectively.

### Logging in Docker

Docker containers typically log events to the standard output. If you run the container in detached mode using the -d flag, the logs will not appear on your terminal immediately. Instead, you can stream them later with:

```
docker logs -f <container_id>
```

### Logging in Kubernetes

View the live logs using:
```
kubectl logs -f <pod-name>
kubectl logs -f event-simulator-pod
```

Logging with Multiple Containers in a Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: event-simulator-pod
spec:
  containers:
    - name: event-simulator
      image: kodekloud/event-simulator
    - name: image-processor
      image: some-image-processor
```

```
kubectl logs -f <pod-name> <container-name>
kubectl logs -f event-simulator-pod event-simulator

or

kubectl logs -f <pod-name> -c <container-name>
kubectl logs -f event-simulator-pod -c image-processor
```
