In this guide I'm going to use KIND to demonstrate how Network Policy restricts traffic between Pods.

## Setup KIND

KIND's default Network plugin doesn't support Network Policies, so we need to disable it in the KIND config, and install another one.

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
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
```

Create the cluster using `kind create cluster --config kind-networking-config.yaml`

## Install Calico

Calico is a popular third party networking solution that supports Network Policies. Documentation on installation in KIND is found [here](https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind)

Install calico for KIND:

``` shell
k apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/calico.yaml
```

## Create Application

We'll use a 3-tier application to demonstrate how network security rules can control traffic between pods.

Create the front-end Pod and Service:
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  labels:
    role: frontend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - name: http
      containerPort: 80
      protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    role: frontend
spec:
  selector:
    role: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
```

Create the back-end Pod and Service:
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
  labels:
    role: backend
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - name: http
      containerPort: 80
      protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    role: backend
spec:
  selector:
    role: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Create the database Pod and Service:
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: mysql
spec:
  containers:
    - name: mysql
      image: mysql:latest
      env:
        - name: "MYSQL_USER"
          value: "mysql"
        - name: "MYSQL_PASSWORD"
          value: "mysql"
        - name: "MYSQL_DATABASE"
          value: "testdb"
        - name: "MYSQL_ROOT_PASSWORD"
          value: "verysecure"
      ports:
        - name: http
          containerPort: 3306
          protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: db
  labels:
    name: mysql
spec:
  selector:
    name: mysql
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
```

## Test connectivity

Test Backend succeeds &#10004;

``` bash
k exec -it backend -- bash
apt-get update && apt-get install telnet -y

telnet db 3306
```

Test Frontend succeeds &#10004;

``` bash
k exec -it frontend -- bash
apt-get update && apt-get install telnet -y

telnet db 3306
```

## Apply Network Policy

We want to only allow traffic to the Database from the back-end Pod and only on Port 3306. To achieve this, apply the following Network Policy:

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: mysql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: backend
    ports:
    - protocol: TCP
      port: 3306
```

Test Backend succeeds &#10004;

``` bash
k exec -it backend -- bash
apt-get update && apt-get install telnet -y

telnet db 3306
```

Test Frontend **fails** &#10004;

``` bash
k exec -it frontend -- bash
apt-get update && apt-get install telnet -y

telnet db 3306
```