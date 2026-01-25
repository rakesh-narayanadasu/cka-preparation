# Deployments

Kubernetes deployments an abstraction that simplifies managing your applications in a production environment. Rather than interacting directly with pods and ReplicaSets, deployments offer advanced features that enable you to:

- Deploy multiple instances of your application (like a web server) to ensure high availability and load balancing.
- Seamlessly perform rolling updates for Docker images so that instances update gradually, reducing downtime.
- Quickly roll back to a previous version if an upgrade fails unexpectedly.
- Pause and resume deployments, allowing you to implement coordinated changes such as scaling, version updates, or resource modifications.

ReplicaSets maintain multiple pod copies. A deployment, however, sits at a higher level, automatically managing ReplicaSets and pods while providing enhanced features like rolling updates and rollbacks.

#### Creating a Deployment

To create a deployment, start by writing a deployment definition file. This file is similar to a ReplicaSet definition, with the key difference being that the kind is set to Deployment instead of ReplicaSet. Below is an example of a correct deployment definition file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
        - name: nginx-container
          image: nginx
```
Once your deployment definition file (for example, named `deployment-definition.yml`) is ready, create the deployment with the following command:
```
kubectl create -f deployment-definition.yml
```
To verify the deployment, run:
```
kubectl get deployments

NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment    3         3         3            3           21s
```

When you create a deployment, Kubernetes automatically creates an associated ReplicaSet. To see this in action, run:
```
kubectl get replicasets
```
You'll notice a new ReplicaSet with a name derived from your deployment. This ReplicaSet oversees the creation and management of pods. To view the pods managed by the ReplicaSet, run:
```
kubectl get pods
```
While deployments and ReplicaSets work together seamlessly, deployments provide additional functionalities such as rolling updates, rollbacks, and the ability to pause/resume changes.

To view all the created Kubernetes objects—deployments, ReplicaSets, pods, and more—use the following command:
```
kubectl get all

NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/myapp-deployment         3         3         3            3           9h

NAME                                        DESIRED   CURRENT   READY   AGE
rs/myapp-deployment-6795844b58              3         3         3       9h

NAME                                      READY   STATUS    RESTARTS   AGE
po/myapp-deployment-6795844b58-5rbjl        1/1     Running   0          9h
po/myapp-deployment-6795844b58-h4w55         1/1     Running   0          9h
po/myapp-deployment-6795844b58-1fjhv         1/1     Running   0          9h
```
This gives you a comprehensive overview of your deployment's components.
