Using a local KIND cluster, follow these steps to practice setting up new users and groups.

1. Test Access: Verify current access permissions.
2. Create Private Key and CSR: Generate a private key and Certificate Signing Request (CSR) for the new user.
3. Create Certificate Signing Request: Encode the CSR and create a Kubernetes CSR manifest, then approve it.
4. Create a Role: Define a role with specific permissions.
5. Create a Role Binding: Bind the role to the new user.
6. Log in as New User: Update kubeconfig to use the new user's credentials.
7. Curl Instead of Kubectl: Use curl with the necessary certificates for authentication.
8. Create a Cluster Role: Define a cluster-wide role.
9. Create a Cluster Role Binding: Bind the cluster role to the new user.
10. Test Access: Verify the new user's access permissions.

## Test access
Let's begin by seeing what access looks like before touching anything. This is done by using the `kubectl auth` commands **can-i** and **whoami**:

``` shell
k auth can-i get pod
k auth can-i get pod --as adam # NO
```

``` shell
k auth whoami
```

## Create private key and CSR for user

Imagine there is a new starter called Adam that we as an admin need to provide access to the cluster. As Adam, start the process by creating a private key and csr:

``` shell
openssl genrsa -out adam.key 2048
openssl req -new -key adam.key -out adam.csr -subj "/CN=adam"
```

## Create Certificate Signing Request

As an admin, next - get the csr from Adam and creates the Kubernetes Certificate Signing Request object. (We must first base64 encode the csr and remove line breaks):

``` shell
cat adam.csr | base64 | tr -d "\n"
```

Use the encoded csr in the CertificateSigningRequest k8s manifest:

``` yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: adam
spec:
  request: < base64 encoded csr >
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
```

Apply manifest to create CSR: `k apply -f adam_csr.yaml`.
Approve CSR: `k certificate approve adam`

Adam (new user) will need the certificate for their kube config:

``` shell
k get csr adam -o jsonpath='{.status.certificate}'| base64 -d > adam.crt
```


## Create a Role
Create a new Role that will allow its members to get pods: `k create role pod-reader --verb=get --verb=list --resource=pods`:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Apply manifest to create new role: `k apply -f role_pod_reader.yaml`

## Create a Role Binding

A Role Binding is required to map a User to a Role. Create a new Role Binding for Adam to join the pod-reader role.

``` shell
k create rolebinding pod-reader-binding-adam --role=pod-reader --user=adam
```

### Test access

Adam can now get pods:

``` shell
k auth can-i get pods --as adam # YES
```

## Log in as New User

As Adam (new user), update kube config in order to use cluster with kubectl:

 - Add new credential

``` shell
k config set-credentials adam --client-key=adam.key --client-certificate=adam.crt --embed-certs=true
```

 - Add the context (get cluster name: `k config get-clusters`)

``` shell
k config set-context adam --cluster=kind-playground --user=adam
```

 - Test by changing to Adam context

``` shell
k config use-context adam
```

## Curl Instead of Kubectl

Kubectl passes all the required certificates for authentication to the control plane. We can use Curl, and pass the certificates manually:
ca.crt needs to be grabbed from the control plane. Using KIND, we need to docker exec in and copy the file:

``` shell
docker exec -it playground-control-plane /bin/bash
cat /etc/kubernetes/pki/ca.crt
```

docker will tell us what port is being exposed.

``` shell
curl https://localhost:46309/api/v1/namespaces/default/pods --key adam.key --cacert ca.crt --cert adam.crt 
```

## Create a Cluster Role
Create a Role to get nodes: `k create clusterrole node-reader --verb=get,list,watch --resource=node`

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: node-reader
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch

```

## Create a Cluster Role Binding

Bind a Cluster Role to a User

``` shell
k create clusterrolebinding node-reader-binding-adam --clusterrole=node-reader --user=adam
```

## Test access

``` shell
k auth can-i get nodes --as adam # YES
```


## Create a Service Account

Create an account to be used by a build service such as Jenkins:

``` shell
k create sa build-sa
```

Create a token for the service account:

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-sa
type: kubernetes.io/service-account-token
```