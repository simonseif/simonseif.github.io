+++
author = "Simon Seif"
title = "Debug Go Processes in Kubernetes"
date = "2021-12-22"
description = "Sample article showcasing basic Markdown syntax and formatting for HTML elements."
tags = [
    "go",
    "kubernetes",
]
categories = [
    "themes",
    "syntax",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
+++

This post describes a procedure to attach the [Delve](https://github.com/go-delve/delve) debugger to a Go application running in a Kubernetes pod and connect the debugger with your local development environment. 
<!--more-->

The procedure uses Kubernetes' [Ephemeral Containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) to launch the debugger.
This comes with the advantage that the container image of the debug target, i.e. the Go application, does not need to bring a rich user land (bash, delve, netcat, etc.).
Instead, this approach works even when the target image is [distroless](https://github.com/GoogleContainerTools/distroless) or built `FROM SCRATCH` (see [Create the smallest and secured golang docker image based on scratch](https://chemidy.medium.com/create-the-smallest-and-secured-golang-docker-image-based-on-scratch-4752223b7324)).
The second feature of this approach is that the debugger is exposed to your local machine via network plumbing and pipes.
It is therefore possible to connect your local development environment, for example GoLand, to the process running in Kubernetes.
The approach is illustrated in the figure below.

![debug setup](/images/delve-kubernetes/setup.png "Debug Setup")

# tl;dr
```bash
EXPORT DEBUGGER_IMAGE=my-debug-toolkit
EXPORT TARGET_POD=my-pod
EXPORT TARGET_CONTAINER=app-container

kubectl debug --image=$DEBUGGER_IMAGE -c debugger $TARGET_POD --target=$TARGET_CONTAINER -- dlv attach 1 --listen=:2345 --headless=true --api-version=2 --accept-multiclient --continue

tcpserver 127.0.0.1 2345 kubectl exec -i -c debugger $TARGET_POD -- nc 127.0.0.1 2345 &

dlv connect 127.0.0.1:2345
```

# Requirements
Ephemeral Containers are a still relatively new feature in Kubernetes.
The feature is included since version [1.16 in Alpha stage](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/).
Alpha stage features are disabled by default and need to be enabled explicitly (start the API server with `--feature-gates=EphemeralContainers=true`).
With the release of Kubernetes 1.23, the feature reached Beta stage and is therefore enabled by default.

Another minor complication is that this approach requires a shared PID namespace between the ephemeral container and the target container.
It is generally possible to [share the PID namespace in pods](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/).
Ephemeral containers allow to target the namespaces of a specific container.
However, the `--target` parameter must be [supported by the Container Runtime](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#ephemeral-container).
For [CRI-O](https://github.com/cri-o/cri-o), full support just landed with the [1.23 release](https://github.com/cri-o/cri-o/releases/tag/v1.23.0). 

# Debug image
```Docker
FROM registry.access.redhat.com/ubi8/ubi
RUN dnf install -y delve nc
ENTRYPOINT /bin/bash
```
The image for the debug container needs to contain Delve, the actual debugger, and netcat (nc).
There are no other requirements to this image.
If you want to include other tools as well or replace the base image, feel free to do so.
Build and push the image to your registry.

# Ephemeral Debug Container
`kubectl debug` creates an ephemeral container in the pod running the debug target.
The ephemeral container runs a headless delve debug server that attaches to the Go process in the target container.
The debug server listens on a local TCP port.

`kubectl debug --image=k8s-0:443/delve -c debugger demo-app --target=app -- dlv attach 1 --listen=:2345 --headless=true --api-version=2 --accept-multiclient --continue`

Breakdown of the command arguments and parameters:

| Argument/Parameter | Description |
|---------------------------|---
| `--image=k8s-0:443/delve` | Name of our debug image (the one built in the step before).
| `-c debugger`             | Name of the debug container. This name cannot be re-used during the lifetime of a pod. In other words, you have to use a new name if you start a second instance of this debug container (including restarts).
| `demo-app`                | Name of the target pod.
| `--target=app`            | Name of the target application container.
| `-- dlv attach 1`         | Attach the debugger the running process with PID 1. If there are multiple processes in the application container the target PID might be different.
| `--listen=:2345`          | Listen locally on port 2345. No other container in the pod must claim this port.
| `--headless=true`         | Run debug server only, in headless mode.
| `--api-version=2`         | Delve API version when headless. New clients should use v2.
| `--accept-multiclient`    | Allows a headless server to accept multiple client connections.
| `--continue`              | Don't stop the execution of the process right after attaching. If the process' execution is suspended, we might trigger liveness probe failures.

# Plumbing
Accessing this port from the outside is now slightly trickier than accessing ports of regular containers.
The API for ephemeral containers does not allow ports.
Therefore, one cannot just declare an ad-hoc service that exposes the debug server.
An ugly work-around is to declare the port in a regular container and just bind to it from the ephemeral container (it is a shared network namespace after all).

Instead, this procedure will take a different path.
On the developer machine, we will start a `tcpserver` process (install from `ucspi-tcp` package).
`tcpserver` waits for connections from TCP clients.
For each connection, `tcpserver` spawns a subprocess and pipes the subprocess' stdin/stdout to the network connection.
We will use `kubectl exec` as this subprocess.


So far we have the following: 
```
tcpserver <--stdin/stdout--> kubectl
```

When run in interactive mode, `kubectl exec` will forward the remote process' stdin/stdout.
The remote process started by `kubectl exec` will be `netcat`.

Adding `netcat` gives us: 
```
tcpserver <--stdin/stdout--> kubectl <--stdin/stdout (tunneled)--> netcat
```

`netcat` running in the debug container will then connect to the (pod) local Delve debug server and forward this TCP connection to stdin/stdout.

This leaves us with the final plumbing: 
```
tcpserver <--stdin/stdout--> kubectl <--stdin/stdout (tunneled)--> netcat <--TCP--> dlv
```
Effectively, the connection between tcpserver (developer machine) and Delve debug server (debug container) is tunneled through kubectl and the Kubernetes API server.

The command to set this up is:

`tcpserver 127.0.0.1 2345 kubectl exec -i -c debugger demo-app -- nc 127.0.0.1 2345`

The command consists of three segments that correspond to the three different processes that are started.

`tcpserver 127.0.0.1 2345`: locally (developer machine) bind to port 2345.
The debug client (IDE) will later connect to this port to initiate a debug session.

`kubectl exec -i -c debugger demo-app --`: the program that is run when a connection to tcpserver is established.
tcpserver pipes stdin and stdout of that process to the network connection. 
The program, `kubectl exec`, runs a command in interactive mode inside the container `-c debugger` of the `demo-app` pod.
Interactive mode means stdin/stdout are forwarded. 

`-- nc 127.0.0.1 2345`: the command executed in the target container.
netcat connects to the local tcp port 2345 and forwards that connection to stdin/stdout.
The port must match the `--listen=:2345` parameter of the debug server.

# Debug Session
At this point, the Delve debug server is exposed locally on port 2345.
Delve can be started in client mode and connect to a server.

`dlv connect 127.0.0.1:2345`

Goland supports this particular setup as run configuration.
`Add configuration...` >> `Go Remote`

![GoLand Go Remote Run Configuration](/images/delve-kubernetes/goland.png "GoLand Go Remote Run Configuration")

# Demo
[![asciicast](https://asciinema.org/a/VSFFQqOFdBOl3viUyrrjPwBRJ.svg)](https://asciinema.org/a/VSFFQqOFdBOl3viUyrrjPwBRJ)

# Caveats
- Ephemeral containers resources are not accounted for.
- Stopping the execution on a breakpoint might cause liveness probes to fail.
- Every time the debug container is re-created (`kubectl debug`), a new name for it must be chosen.

