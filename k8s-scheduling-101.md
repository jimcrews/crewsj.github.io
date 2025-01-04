I'm revisiting Kubernetes scheduling to cement my learning. Kubernetes has some powerful scheduling capabilities to make sure your pods land on the correct nodes in your cluster (given you want to have control over this placement). In this guide, I'll walk through the basics of manual scheduling, node selectors, taints and tolerations, and node affinity.

## Manual Scheduling

For new pods joining clusters without a Scheduler:

`nodeName` can be added manually to the Pod definition file which will assign the pod to the specified node.
`nodeName` is a child of `spec`:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    tier: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  nodeName: playground-worker3
```

## Node Selectors

### Add label to node

``` shell
k label nodes playground-worker2 disktype=ssd

# test selector

k get nodes --selector disktype=ssd
```


### Create a new Pod definition that includes `nodeSelector`

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    tier: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  nodeSelector:
    disktype: ssd
```


## Taints and Tolerations

Taints allow a node to repel a set of pods. Tolerations allow the scheduler to schedule pods with matching taints.

### Add Taints to nodes

``` shell
k taint nodes playground-worker env=dev:NoSchedule
k taint nodes playground-worker2 env=test:NoSchedule
k taint nodes playground-worker3 env=prod:NoSchedule
```

### Create Pods

Each node in the cluster has a taint that will prevent any pod from being scheduled on it without a toleration..
Create a pod without any tolerations and confirm it remains ina Pending state.

Tolerations need to be added to a Pod definition file

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    tier: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "test"
    effect: "NoSchedule"
```

Taints and Tolerations does not tell a Pod to go to a particular Node. For example, tolerations on a Pod does not stop it from being placed on an untainted node.

### Remove Taints from nodes

``` shell
k taint nodes playground-worker env=dev:NoSchedule-
k taint nodes playground-worker2 env=test:NoSchedule-
k taint nodes playground-worker3 env=prod:NoSchedule-
```

## Node Affinity

Node affinity is conceptually similar to nodeSelector, allowing you to constrain which nodes your Pod can be scheduled on based on node labels. There are two types of node affinity:

`requiredDuringSchedulingIgnoredDuringExecution`: The scheduler can't schedule the Pod unless the rule is met. This functions like nodeSelector, but with a more expressive syntax.
`preferredDuringSchedulingIgnoredDuringExecution`: The scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler still schedules the Pod.

We still have a label on our worker2 node

``` shell
k get nodes --selector disktype=ssd
```

We can use Node Affinity to assign a pod to this node:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    tier: frontend
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd

```