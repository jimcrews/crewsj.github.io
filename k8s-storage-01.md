This guide walks you through how to manage and persist data using Docker volumes and Kubernetes Persistent Volumes (PVs) and Persistent Volume Claims (PVCs). It starts with creating and using Docker volumes, then shows how to mount volumes directly from your filesystem. We also cover setting up Persistent Volumes and Persistent Volume Claims in a KIND cluster.

Lets start by persisting data in Docker Containers, without Kubernetes.

### Docker Volumes

1. Create a Docker Volume
``` shell
docker volume create my-volume
```

2.  Create Docker Container using Volume
``` shell
docker run -it --rm --mount source=my-volume,destination=/my-data/ ubuntu:22.04
```

## Docker Volume Mounts

Instead of creating a Docker Volume, mount a folder directly from filesystem:

``` shell
docker run -it --rm --mount type=bind,source="${HOME}"/data,destination=/my-data ubuntu:22.04
```

## Kubernetes Volumes using HostPath

A **hostPath** volume mounts a file or directory from the host node's filesystem into your Pod. This Pod will generate a random number and append it to **/opt/number.txt**.
The below alpine pod's /opt directory is mounted to the nodes /tmp directory.


``` yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: alpine
  name: alpine
spec:
  containers:
  - image: alpine
    name: alpine
    resources: {}
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 1-100 -n 1 >> /opt/number.txt"]
    volumeMounts:
    - name: data-volume
      mountPath: /opt
  
  volumes:
  - name: data-volume
    hostPath:
      path: "/tmp"
      type: Directory

  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```
Because I am using KIND, I can check what node was assigned this pod, and confirm the file **number.txt** got created using `docker exec`:
``` shell
docker exec -it playground-worker3 /bin/bash
```

## Persistent Volumes

Instead of configuring volumes each time within a Pod manifest, Persistent Volumes are created by admins to provision storage resources ahead of time that can be used by pods.

To test PV's and PVC's using KIND, we need to update the KIND config as below. This creates a mapping from my PC to a path on the Control Plane node.

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
# map local folder to node path
  extraMounts:
  - hostPath: /home/jimc/data # local computer path
    containerPath: /data  # control plane path
- role: worker
- role: worker
- role: worker
```

Create cluster using: `kind create cluster --config kind-basic-config.yaml`

Now that the control plane has a folder called **/data**, we can create a PV to use it, then PVC to use the PV.

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  storageClassName: manual
  capacity:
    storage: 50Mi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data
```

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```

After you create the PersistentVolumeClaim, the Kubernetes control plane looks for a PersistentVolume that satisfies the claim's requirements. The control plane finds a suitable PersistentVolume with the same StorageClass, and binds the claim to the volume.

Next, create a new Pod on the control plane node as this is where KIND created the mapped folder

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  nodeName: playground-control-plane
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - mountPath: "/opt/my_share"
        name: my-volume
  volumes:
    - name: my-volume
      persistentVolumeClaim:
        claimName: example-pvc
```

Exec into the running pod to see the mapped folder in **/opt/my_share**

``` shell
k exec -it example-pod -- /bin/bash
```