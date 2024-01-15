Kubernetes is an open source platform for managing containerized workloads. This post goes over setting up a local cluster, and then looking at how the scheduler places pods on particular nodes. *A pod is a group of 1 or more containers, and a node is a worker machine*.

Minikube is marketed as 'local Kubernetes', focusing on making it easy to learn and develop for Kubernetes.. Due to its age there's a lot of support for it, however it can only run a single node cluster. In the "real world", clusters will typically spread across multiple nodes. Enter K3D. Like Minikube, K3D runs Kubernetes in Docker, but has support for multiple nodes, so that's what I'm using.

## Local Cluster Setup

Follow the [docs](https://k3d.io/v5.6.0/#installation) to get K3D installed.
[kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) and [docker](https://docs.docker.com/get-docker/) are required. **I'll also be using the alias of 'k' for 'kubectl'**

### K3D Create Multi-Node Cluster

Create a local cluster with 1 Control Plane, and 3 Worker Nodes and make it accessible via ingress (map the ingress port 80 to localhost:8081):
```
k3d cluster create my-cluster -p "8081:80@loadbalancer"  --servers 1 --agents 3
```

Lets take a look at what nodes are available using `k get nodes`

| NAME                    | STATUS | ROLES                | AGE |
|-------------------------|--------|----------------------|-----|
| k3d-my-cluster-server-0 | Ready  | control-plane,master | 40s |
| k3d-my-cluster-agent-0  | Ready  |                      | 37s |
| k3d-my-cluster-agent-1  | Ready  |                      | 36s |
| k3d-my-cluster-agent-2  | Ready  |                      | 35s |

**A note on namespaces**: I heard its bad practice to use the default namespace, so let's create a new namespace for all components going forward.
```
k create namespace test
```

### Create 5 nginx Containers

Let's now **create a deployment** of 5 pods. By default, these pods will be placed on any random node, including the control-plane node.

```
k create deployment nginx-deploy \
--image=nginx \
--dry-run=client \
--replicas=5 \
--output yaml \
> nginx-deploy.yaml
```

Apply this config into the namespace 'test':
```
k apply -f nginx-deploy.yaml -n test
``` 

Verify the new pods in the 'test' namespace using `k get pods -n test -o wide`:

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

We need to point the service to our containers. Update the config 'selector' section before applying it:
```
selector:
    app: nginx-deploy
```

**Apply** this config into the namespace 'test':
```
k apply -f nginx-service.yaml -n test
```

### Create Ingress
Ingress lets us route traffic from outside the cluster to one or more services inside the cluster.

Create the ingress object by copying config from the kubernetes [docs](https://kubernetes.io/docs/concepts/services-networking/ingress/) to a new file called my_ingress.yaml.

Apply this config into the namespace 'test':
```
k apply -f my_ingress.yaml -n test
```

**We can now browse to http://localhost:8081/ and see the 'Welcome to nginx' page.**

This is what we have so far. Our cluster components are aware of eachother because of the selectors:

[![nginx_ingress_config](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/nginx_ingress_config.png)](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/nginx_ingress_config.png)

## Kubernetes Scheduler

How to ensure pods get scheduled on the correct nodes ?

### Taints and Tolerations

I want new pods to get scheduled on the 'agent' (or worker) nodes and not the control-plane node. To ensure this happens, I'll 'Taint' the control-plane node.

Taints and Tolerations restrict nodes from accepting certain pods. A taint is basically a node saying "I'm filthy, go away", a toleration is a pod saying "I don't care if you're dirty, i wouldn't mind running on you".

 - Taints are set on Nodes
 - Tolerations are set on Pods

Apply a Taint to the control-plane:
```
k taint nodes \
k3d-my-cluster-server-0 \
node-role.kubernetes.io/control-plane:NoSchedule
```

Check the Taint was created:
```
k describe node k3d-my-cluster-server-0 | grep -i taint
```

Re-deploy Deployment to see what happens:
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

A pod would require this toleration to run on the control-plane node:

``` yaml
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
```

### Node Affinity

Node affinity is a pod saying "i love to run on nodes with label so-and-so". Let's add labels to our Nodes - 'Large' and 'Small'

```
k label nodes k3d-my-cluster-agent-0 size=Small
k label nodes k3d-my-cluster-agent-1 size=Small
k label nodes k3d-my-cluster-agent-2 size=Large
```

Update the deployment config (nginx-deploy.yaml) by including an 'Affinity' section to mandate pods only run on 'Small' nodes.

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


Re-create the pods to see the result:
```
k delete deployment nginx-deploy -n test
k apply -f nginx-deploy.yaml -n test
```

No Pods are running on agent 2 because it's 'size' label value was set to 'Large'.

| NAME                         | READY | STATUS  | AGE | NODE                   |
|------------------------------|-------|---------|-----|------------------------|
| nginx-deploy-d864d4448-6hzr7 | 1/1   | Running | 44s | k3d-my-cluster-agent-1 |
| nginx-deploy-d864d4448-7r92b | 1/1   | Running | 44s | k3d-my-cluster-agent-1 |
| nginx-deploy-d864d4448-qmq2r | 1/1   | Running | 44s | k3d-my-cluster-agent-0 |
| nginx-deploy-d864d4448-zh4qk | 1/1   | Running | 44s | k3d-my-cluster-agent-0 |
| nginx-deploy-d864d4448-6l4v5 | 1/1   | Running | 44s | k3d-my-cluster-agent-0 |


## Cleanup

Delete all the cluster components and the cluster itself by running these commands:

```
k delete -f my_ingress.yaml -n test
k delete service nginx-service -n test
k delete deployment nginx-deploy -n test
k3d cluster delete my-cluster
```