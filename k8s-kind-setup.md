# Table of Contents

- [Prerequisites](#prerequisites)
- [KIND Cluster Pods and Services](#creating-a-cluster)
- [Build / Deploy / Expose Custom Django Container](#build--deploy--expose-custom-django-container)
- [Cleanup](#cleanup)

## Prerequisites

 - [Docker](https://docs.docker.com)
 - [Kubectl](https://kubernetes.io/docs/)
 - [kind](https://kind.sigs.k8s.io/)

## Creating a Cluster

Once Kind is installed, create a Kind config file to specify cluster options. We'll start with a 4 node cluster.
Because we are using KIND, we need to take additional steps to expose the Cluster to the local computer. To do this, we need to use port mappings. The kind `containerPort` and the service `nodePort` needs to be equal.

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
kind create cluster --config ./kind-ports-config.yaml
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

Test connectivity from your local machine. You should receive the nginx welcome page:

```
curl localhost:30001
```

## Internal Communication

Now we need to expose PODs for internal communication. We will create a new Pod called `backend` that needs to communicate with Pod `nginx`.
Create the Pod:

`k run backend --image=nginx --labels="tier=backend"`

Test comms out of the box by trying to telnet `backend` from inside `nginx`:

``` bash
k exec -it nginx -- bash
apt-get update && apt-get install telnet -y

telnet backend 80 # FAIL
```

A service of type ClusterIP needs to be created to expose the backend application to nginx.

Create Service ClusterIP

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    tier: backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

``` bash
k exec -it nginx -- bash

telnet backend-svc 80 # SUCCEEDS
```

The **Service Name** is now the link to the backend Pod


## Build / Deploy / Expose Custom Django Container

Create sample app docker image (using the Dockerfile below):

``` dockerfile
FROM python:3.6-slim

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1
RUN mkdir /app
WORKDIR /app
RUN pip install --upgrade pip

COPY requirements.txt /app
COPY devops /app

RUN pip install -r requirements.txt && cd devops

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

Build image and push to dockerhub:

``` shell
docker build -t jimcr/django-sample-app-demo:v1 .

# Push to dockerhub
docker login
docker push jimcr/django-sample-app-demo:v1
```

Create Deployment (the **containerPort** is the port exposed in the Container):

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: django-deploy
  name: django-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-deploy
  template:
    metadata:
      labels:
        app: django-deploy
    spec:
      containers:
      - image: jimcr/django-sample-app-demo:v1
        name: django-sample-app-demo
        ports:
        - containerPort: 8000
```

I can now access the website from within the cluster only. If I docker exec into the KIND control plane node, I can curl the IP address of the worker node the app is currently hosted on:

``` shell
docker exec -it 73216bf1c0c9 /bin/bash

curl -L http://10.244.1.3:8000/demo
```

Create NodePort to expose service to localhost:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: django
spec:
  type: NodePort
  selector:
    app: django-deploy
  ports:
    - port: 80
      targetPort: 8000
      nodePort: 30001
```

Browse to http://localhost:30001/demo/ to test

## Cleanup

``` shell
kind delete cluster --name playground
```