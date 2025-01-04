# Setting Up a Local Kubernetes Cluster with kind

In this guide, we will set up a local Kubernetes cluster using Kind (Kubernetes IN Docker). Kind is a tool for running local Kubernetes clusters using Docker container nodes. It is primarily designed for testing Kubernetes itself, but it can also be used for local development or CI. We will also demonstrate how to test using NodePort for port mappings.

## Prerequisites

 - [Docker](https://docs.docker.com)
 - [Kubectl](https://kubernetes.io/docs/)
 - [kind](https://kind.sigs.k8s.io/)

## Creating a Cluster

Once Kind is installed, create a Kind config file to specify cluster options. We'll start with a 4 node cluster.
To use port mappings with NodePort, the kind node containerPort and the service nodePort needs to be equal.

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: playground
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30001
    hostPort: 30001
    listenAddress: "0.0.0.0"
    protocol: tcp
- role: worker
- role: worker
- role: worker
```

Create the cluster: 

``` shell
kind create cluster --config ./kind-basic-config.yaml
```

### Verify the Cluster

 - List the clusters: `kind get clusters`
 - Use `kubectl` to get nodes: `k get nodes`
 - List docker containers: `docker ps`



## Test NodePort Connection

Create an nginx Pod for testing web connectivity from outside the cluster nodes network:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - image: nginx
    name: nginx

```

Create a NodePort service that matches the Kind cluster NodePort config:

``` shell
k create svc nodeport my-np --tcp=30001:80 --dry-run=client -o yaml
```

Update the selectors to point to the pod created above. Result should look like:

``` yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-np
  name: my-np
spec:
  ports:
  - name: http
    nodePort: 30001
    port: 80
    targetPort: 80
  selector:
    app: nginx
    tier: frontend
  type: NodePort

```

Test connectivity from your local machine:

```
curl localhost:30001
```

You should receive the nginx welcome page

``` shell
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

## Cleanup

``` shell
kind delete cluster --name playground
```