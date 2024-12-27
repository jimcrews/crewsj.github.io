# Setting Up a Local Kubernetes Cluster with kind

In this guide, we will set up a local Kubernetes cluster using kind (Kubernetes IN Docker). kind is a tool for running local Kubernetes clusters using Docker container nodes. It is primarily designed for testing Kubernetes itself, but it can also be used for local development or CI.

## Prerequisites

 - [Docker](https://docs.docker.com)
 - [Kubectl](https://kubernetes.io/docs/)
 - [kind](https://kind.sigs.k8s.io/)

## Creating a Cluster

Once Kind is installed, create a Kind config file to specfiy cluster options. We'll start with a 4 node cluster.

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: playground
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

Create the cluser: 

``` shell
kind create cluster --config ./kind-basic-config.yaml
```


### Verify the Cluster

 - List the clusters: `kind get clusters`
 - Use `kubectl` to get nodes: `k get nodes`
 - List docker containers: `docker ps`

### Delete the Cluster

`kind delete cluster --name playground`