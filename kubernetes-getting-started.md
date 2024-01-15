Kubernetes is an open source platform for managing containerized workloads. This post goes over setting up a local cluster, and then looking at how the scheduler places pods on particular nodes. *A pod is a group of 1 or more containers, and a node is a worker machine*.

Minikube is popular. Due to its age there's a lot of support for it, however it can only run a single node cluster. In the "real world", clusters will typically spread across multiple nodes. Enter K3D. Like Minikube, K3D runs Kubernetes in Docker, but has support for multiple nodes, so that's what I'm using.

## Local Cluster Setup

Follow the [docs](https://k3d.io/v5.6.0/#installation) to get K3D installed.
[kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) and [docker](https://docs.docker.com/get-docker/) are required. I'll also use the alias of 'k' for 'kubectl'

I'm following the [K3D Ingress Guide](https://k3d.io/v5.4.6/usage/exposing_services/) to get a cluster running, and expanding on each step.

### K3D Create Multi-Node Cluster

Create a local cluster with 1 Control Plane, and 3 Worker Nodes and make it accessible via ingress (map the ingress port 80 to localhost:8081).
```
k3d cluster create my-cluster -p "8081:80@loadbalancer"  --servers 1 --agents 3
```

Lets take a look at what nodes are available using `k get nodes`

| NAME                    | STATUS | ROLES                | AGE | VERSION      |
|-------------------------|--------|----------------------|-----|--------------|
| k3d-my-cluster-server-0 | Ready  | control-plane,master | 40s | v1.27.5+k3s1 |
| k3d-my-cluster-agent-0  | Ready  |                      | 37s | v1.27.5+k3s1 |
| k3d-my-cluster-agent-1  | Ready  |                      | 36s | v1.27.5+k3s1 |
| k3d-my-cluster-agent-2  | Ready  |                      | 35s | v1.27.5+k3s1 |


I heard its bad practice to use the default namespace, so let's create a new namespace for all things going forward.
```
k create namespace test
```

We'll create config files for each component so we can see how they join together at the end.

### Deploy 3 nginx Containers into Cluster

Let's now create a deployment of 5 pods. By default, these pods will be placed on any random node, including the control-plane node.

```
k create deployment nginx-deploy \
--image=nginx \
--dry-run=client \
--replicas=5 \
--output yaml \
> nginx-deploy.yaml
```

apply config into namespace 'test'
```
k apply -f nginx-deploy.yaml -n test
``` 

check the new pods in the 'test' namespace using `k get pods -n test -o wide`.

| NAME                          | READY | STATUS  | AGE   | NODE                    |
|-------------------------------|-------|---------|-------|-------------------------|
| nginx-deploy-846d6f46b7-8m9nx | 1/1   | Running | 2m17s | k3d-my-cluster-agent-2  |
| nginx-deploy-846d6f46b7-hmzhr | 1/1   | Running | 2m17s | k3d-my-cluster-server-0 |
| nginx-deploy-846d6f46b7-f8547 | 1/1   | Running | 2m17s | k3d-my-cluster-agent-0  |
| nginx-deploy-846d6f46b7-nzjck | 1/1   | Running | 2m17s | k3d-my-cluster-agent-0  |
| nginx-deploy-846d6f46b7-fk9dj | 1/1   | Running | 2m17s | k3d-my-cluster-agent-1  |


### Create a Service

A service is an abstraction that exposes a group of pods as a network service.

```
k create service clusterip nginx --tcp=80:80 --dry-run=client -o yaml > nginx-service.yaml
```
update the yaml selector app to point to our deployment: 'nginx-deploy'
```
k apply -f nginx-service.yaml -n test
```

### Create Ingress
Ingress lets us route traffic from outside the cluster to one or more services inside the cluster

create the ingress object (copy config from [docs](https://kubernetes.io/docs/concepts/services-networking/ingress/)). Update the namespace.
```
k apply -f my_ingress.yaml -n test
```

We can now browse to http://localhost:8081/ and see the 'Welcome to nginx' page.

These components hook up together using the following selectors:

![nginx_ingress_config](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/nginx_ingress_config.png "nginx_ingress_config")

## Kubernetes Scheduler

How to ensure pods get scheduled on the correct nodes ?

### Taints and Tolerations

I want new pods to get scheduled on the 'agent' (or worker) nodes and not the control-plane node. To ensure this happens, I'll 'Taint' the control-plane node.

Taints and Tolerations restrict nodes from accepting certain pods. it does not tell the pod to go to a particular node.
 - Taints are set on Nodes
 - Tolerations are set on Pods

Apply Taint
```
k taint nodes \
k3d-my-cluster-server-0 \
node-role.kubernetes.io/control-plane:NoSchedule
```

Check Taint
```
k describe node k3d-my-cluster-server-0 | grep -i taint
```

Re-deploy Deployment
```
k delete deployment nginx-deploy -n test
k apply -f nginx-deploy.yaml -n test
```

No Pod has a toleration to 'node-role.kubernetes.io/control-plane', so no pod was sheduled on that node.

| NAME                          | READY | STATUS  | AGE | NODE                   |
|-------------------------------|-------|---------|-----|------------------------|
| nginx-deploy-846d6f46b7-82rlb | 1/1   | Running | 5s  | k3d-my-cluster-agent-0 |
| nginx-deploy-846d6f46b7-g5crs | 1/1   | Running | 5s  | k3d-my-cluster-agent-2 |
| nginx-deploy-846d6f46b7-wwzxr | 1/1   | Running | 5s  | k3d-my-cluster-agent-2 |
| nginx-deploy-846d6f46b7-4zb2v | 1/1   | Running | 5s  | k3d-my-cluster-agent-0 |
| nginx-deploy-846d6f46b7-dwkh9 | 1/1   | Running | 5s  | k3d-my-cluster-agent-1 |


### Node Affinity

Let's add labels in order to group the agent nodes into 'Large' and 'Small'

```
k label nodes k3d-my-cluster-agent-0 size=Small
k label nodes k3d-my-cluster-agent-1 size=Small
k label nodes k3d-my-cluster-agent-2 size=Large
```

Update the deployment config by including an 'Affinity' section:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-deploy
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}

      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: size
                operator: In
                values:
                - Small

status: {}

```


Now let's re-deploy the pods to the 'Small' nodes by adding 'affinity' rules into the config
```
k delete deployment nginx-deploy -n test
k apply -f nginx-deploy.yaml -n test
```

| NAME                         | READY | STATUS  | AGE | NODE                   |
|------------------------------|-------|---------|-----|------------------------|
| nginx-deploy-d864d4448-6hzr7 | 1/1   | Running | 44s | k3d-my-cluster-agent-1 |
| nginx-deploy-d864d4448-7r92b | 1/1   | Running | 44s | k3d-my-cluster-agent-1 |
| nginx-deploy-d864d4448-qmq2r | 1/1   | Running | 44s | k3d-my-cluster-agent-0 |
| nginx-deploy-d864d4448-zh4qk | 1/1   | Running | 44s | k3d-my-cluster-agent-0 |
| nginx-deploy-d864d4448-6l4v5 | 1/1   | Running | 44s | k3d-my-cluster-agent-0 |


A taint is basically a node saying "I'm filthy, go away", a toleration is a pod saying "I don't care if you're dirty, i wouldn't mind running on you" and a node affinity is a pod saying "i love to run on nodes with label so-and-so".

## Cleanup

```
k delete -f my_ingress.yaml -n test
k delete service nginx-service -n test
k delete deployment nginx-deploy -n test
k3d cluster delete my-cluster
```