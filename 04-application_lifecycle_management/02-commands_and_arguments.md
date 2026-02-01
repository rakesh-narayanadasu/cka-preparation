# Configure Applications

Configuring applications comprises of understanding the following concepts:

- Configuring Command and Arguments on applications
- Configuring Environment Variables
- Configuring Secrets

# Commands and Arguments in Docker

You can specify the command in either of two formats:

1) Shell form:
```
CMD sleep 5
```

2) JSON array format:
```
CMD ["sleep", "5"]
```

### Configuring ENTRYPOINT for Runtime Arguments

Sometimes, you may want to specify only runtime arguments without changing the default command. In such cases, the ENTRYPOINT instruction is useful. This instruction sets the executable to run when the container starts, and any command-line arguments provided at runtime are appended to it.

Consider the following Dockerfile:
```
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

### Overriding ENTRYPOINT at Runtime

At times, you might want to completely override the ENTRYPOINT. For example, if you wish to use a different command (like switching from sleep to sleep2.0), you can do so using the `--entrypoint` flag in the docker run command.

Given the Dockerfile:
```
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

```
docker run --entrypoint sleep2.0 ubuntu-sleeper 10
```

# Commands and Arguments in Kubernetes

### Overriding Default Behavior with Arguments
By specifying the args field in the pod definition file, the CMD instruction is overridden, which effectively changes the sleep duration from 5 to 10 seconds.

Docker commands:
```
docker run --name ubuntu-sleeper ubuntu-sleeper     # sleeps 5 seconds
docker run --name ubuntu-sleeper ubuntu-sleeper 10
```

Pod definition YAML:

```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]
```

### Overriding the ENTRYPOINT

Now, consider a scenario where you want to override the ENTRYPOINT itself (for example, switching from "sleep" to an alternative command like "sleep2.0"). In Docker, you would use the `--entrypoint` option in the `docker run` command. In Kubernetes, this is achieved by providing the `command` field in the pod definition. Here, the `command` field corresponds to the Dockerfile’s ENTRYPOINT, while the `args` field continues to override the CMD instruction.

Docker command example with overridden ENTRYPOINT:
```
docker run --name ubuntu-sleeper --entrypoint sleep2.0 ubuntu-sleeper 10
```

Pod definition YAML with overridden ENTRYPOINT:
```
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]
    args: ["10"]
```

**Remember that specifying the `command` in a pod definition replaces the Dockerfile's `ENTRYPOINT` entirely, while the `args` field only overrides the default parameters defined by `CMD`.**

```
---
apiVersion: v1 
kind: Pod 
metadata:
  name: ubuntu-sleeper-2 
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command:
      - "sleep"
      - "5000"
```

```
apiVersion: v1 
kind: Pod 
metadata:
  name: ubuntu-sleeper-2 
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "5000"]
```

```
containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
      - |
        echo "Starting...";
        apt update && apt install -y curl;
        curl https://example.com;
        echo "Done"
```

- `command` → replaces `ENTRYPOINT`
- `args` → replaces `CMD`
- `/bin/sh -c` lets you run multiple commands in sequence

```
apiVersion: v1
kind: Pod
metadata:
  name: webapp-green
spec:
  containers:
    - name: color
      image: kodekloud/webapp-color
      args:
        - "color"
        - "green"
```

If the app expects flag-style arguments `python app.py --color=green` then use below `args` format:
```
apiVersion: v1 
kind: Pod 
metadata:
  name: webapp-green
  labels:
      name: webapp-green 
spec:
  containers:
  - name: simple-webapp
    image: kodekloud/webapp-color
    args: ["--color=green"]
```


| Pod Definition Field | Dockerfile Instruction | Functionality Description |
|---------------------|------------------------|---------------------------|
| command             | ENTRYPOINT             | Specifies the command to run when the container starts (completely replacing the Dockerfile ENTRYPOINT). |
| args                | CMD                    | Provides default parameters passed to the command (overriding the Dockerfile CMD). |
