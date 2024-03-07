Use Taints and Toleration's with Node Affinity rules to ensure Pods get scheduled on Specific Nodes.

**Taints and tolerations** work together to ensure that pods are not scheduled onto inappropriate nodes. When you taint a node, it will repel all the pods except those that have a toleration for that taint. A node can have one or many taints associated with it.

**Node Affinity** allows you to constrain which nodes your Pod can be scheduled on based on node labels

## Demonstration using local Cluster

Our demo cluster consists of 1 control-plane (`kind-control-plane`) node and 3 worker nodes (`kind-worker`, `kind-worker2`, `kind-worker3`).

The Control Plane node is tainted by default. We'll add taints to 2 worker nodes (kind-worker, and kind-worker2). 1 worker will be left untainted (kind-worker3).

```
k taint node kind-worker spray=mortein:NoSchedule
k taint node kind-worker2 spray=brute:NoSchedule
```

### Current state - Taints added to Nodes:

| NAME               | Taints                                |
|--------------------|---------------------------------------|
| kind-control-plane | node-role.kubernetes.io/control-plane |
| kind-worker        | spray=mortein                         |
| kind-worker2       | spray=brute                           |
| kind-worker3       |                                       |


### Create Pods

```
k run mortein-pod --image=nginx
k run brute-pod --image=nginx
```

Pods run on untainted nodes only, because no Toleration's have been applied to either Pod. (remember, kind-worker3 is untainted)

| NAME        | READY | STATUS  | NODE         |
|-------------|-------|---------|--------------|
| brute-pod   | 1/1   | Running | kind-worker3 |
| mortein-pod | 1/1   | Running | kind-worker3 |

### update both pods to include toleration's. (mortein example below).

```
  tolerations:
  - key: "spray"
    operator: "Equal"
    value: "mortein"
    effect: "NoSchedule"
```

This time, the _brute_ Pod is allowed to run on the _brute_ Node and does so, but even though the _mortein_ Pod had the option to run on tainted node `kind-worker`, chose the untainted `kind-worker3`

| NAME        | READY | STATUS  | NODE         |
|-------------|-------|---------|--------------|
| brute-pod   | 1/1   | Running | kind-worker2 |
| mortein-pod | 1/1   | Running | kind-worker3 |

So Taints and Tolerations are not enough to ensure Pods land where we want them to. In order to make sure _brute_ runs on `kind-worker2`, and _mortein_ runs on `kind-worker`, we need to add labels to the Nodes, and Affinity rules to the Pods. 

### Label Nodes

```
k label node kind-worker spray-type=mortein
k label node kind-worker2 spray-type=brute
```

### Current state - Taints and Labels added to Nodes:

| NAME               | Taints                                | Label notes |
|--------------------|---------------------------------------|-------------|
| kind-control-plane | node-role.kubernetes.io/control-plane |             |
| kind-worker        | spray=mortein                         | spray-type=mortein  |
| kind-worker2       | spray=brute                           | spray-type=brute  |
| kind-worker3       |                                       |             |


### Add Node Affinity rules to Pods. (mortein example below).

```
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: spray-type
            operator: In
            values:
            - mortein
```
After re-creating the pods, the scheduler places them exactly where we want them:

| NAME        | READY | STATUS  | NODE         |
|-------------|-------|---------|--------------|
| brute-pod   | 1/1   | Running | kind-worker2 |
| mortein-pod | 1/1   | Running | kind-worker  |


Below are the complete pod templates:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: brute-pod
  name: brute-pod
spec:
  containers:
  - image: nginx
    name: brute-pod
    resources: {}

  tolerations:
  - key: "spray"
    operator: "Equal"
    value: "brute"
    effect: "NoSchedule"

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: spray-type
            operator: In
            values:
            - brute

  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mortein-pod
  name: mortein-pod
spec:
  containers:
  - image: nginx
    name: mortein-pod
    resources: {}

  tolerations:
  - key: "spray"
    operator: "Equal"
    value: "mortein"
    effect: "NoSchedule"

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: spray-type
            operator: In
            values:
            - mortein

  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```


