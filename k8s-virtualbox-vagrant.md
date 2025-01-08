This is a demo of installing Kubernetes on local Virtual Machines, using my **Ubuntu** host computer.

## Prerequisites

 - [VirtualBox](https://www.virtualbox.org/wiki/Linux_Downloads)
 - [Vagrant](https://developer.hashicorp.com/vagrant/downloads#linux)

The following Vagrantfile script creates a total of 3 nodes: 1 master node and 2 worker nodes. All vm's will be configured with IP's in the 192.168.1 range and name resolution configured so communication between nodes via IP or name is good to go.

https://github.com/jimcrews/crewsj.github.io/blob/main/vagrant/Vagrantfile

The following `vagrant` commands will be used. These need to be executed in the same directory as the Vagrantfile above.

``` shell
vagrant status # get status of vagrant
vagrant up # starts and provisions the vagrant environment
vagrant ssh # connects to a machine via ssh
vagrant destroy # stops and deletes all traces of the vagrant machine
```

`vagrant status` should show nothing created:

``` text
Current machine states:

controlplane              not created (virtualbox)
node01                    not created (virtualbox)
node02                    not created (virtualbox)
```

## Provision VMs using Vagrant

Get started by bringing up the ubuntu nodes:

``` shell
vagrant up
```

`vagrant status` should show 3 nodes running:

``` text
Current machine states:

controlplane              running (virtualbox)
node01                    running (virtualbox)
node02                    running (virtualbox)
```

## Install Kubeadm

We need to use the Kubernetes Docs, so start by searching for: **installing Kubeadm**.
Then ssh-ing into the vms:

``` shell
vagrant ssh controlplane
vagrant ssh node01
vagrant ssh node02
```

Following the Docs, install Kubeadm on all 3 nodes:

1. Update the apt package index and install packages needed to use the Kubernetes apt repository:

``` shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

2. Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

``` shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

3. Add the appropriate Kubernetes apt repository:

``` shell
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4. Update the apt package index, install kubelet, kubeadm and kubectl

``` shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl # we might as well install kubelet and kubectl as well
sudo apt-mark hold kubelet kubeadm kubectl
```

With kubeadm now install, you can check its version by running `kubeadm version`, then follow the doco by clicking the link **installing the Cluster**.

## Creating a cluster with kubeadm

The Instructions start by requiring us to install a container runtime on our nodes.

Start by Enabling IPv4 packet forwarding on all nodes:

``` shell
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

Verify that net.ipv4.ip_forward is set to 1 with:

``` shell
sysctl net.ipv4.ip_forward
```

Install **containerd** as our container runtime on all nodes:

``` shell
sudo apt update
sudo apt install containerd -y
```

### cgroup drivers

It's critical that the kubelet and the container runtime use the same cgroup driver and are configured the same. The cgroupfs driver is not recommended when **systemd** is the init system.
Check the init system in use on our nodes:
``` shell
ps -p 1
```
Looks like it is systemd:
``` text
    PID TTY          TIME CMD
      1 ?        00:00:01 systemd

```

Starting with v1.22 and later, when creating a cluster with kubeadm, kubeadm defaults it to systemd. So there is nothing regarding the kubelet process for us to do as our kubeadm version is 1.32. But we do have to make sure the container runtime uses systemd as the cgroup driver. To do that we need to update the containerd settings toml: `/etc/containerd/config.toml`

Create the containerd folder on all nodes:

``` shell
sudo mkdir /etc/containerd
```

The default configuration can be generated via `containerd config default > /etc/containerd/config.toml`, but because we need to update part of that config for systemd, use the following command to generate the containerd config on all nodes:

``` shell
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sudo tee /etc/containerd/config.toml
```

Restart containerd on all nodes: `sudo systemctl restart containerd`

## Initializing your control-plane node

We need to specify a POD subnet CIDR - we will use the common one (which we need to specify). We will also need to advertise api server address (which we will set as the local ip for the controlplane vm).

On the controlplane vm:

``` shell
sudo kubeadm init --apiserver-advertise-address=192.168.1.21 --pod-network-cidr="10.244.0.0/16" --upload-certs
```

If successful, you can use the output to setup kubectl, setup networking, and join the worker nodes:

``` text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/


Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.21:6443 --token lnh.... \
	--discovery-token-ca-cert-hash sha256:....
```

## Installing Addon - Networking

Select and use the flannel networking docs to apply flannel manifest:

``` shell
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

After installing flannel, looking at the dns pods (using kubectl describe) they were in error:

``` text
loadFlannelSubnetEnv failed: open /run/flannel/subnet.env: no such file or directory
```

Using google, I need to create that file on the controlplane node (coredns isnt running on workers):

``` shell
sudo vim /run/flannel/subnet.env
```

with contents:

``` text
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

Looking at the flannel pods (using kubectl logs) they are in error:

``` text
Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory
```

Using google I need to create that file on all nodes (flannel runs on all nodes):

``` shell
sudo modprobe br_netfilter
sudo sysctl -p /etc/sysctl.conf
```

Wait for the flannel pod to restart.

## Join the Worker Nodes to the Cluster

Use the `kubeadm join` command from above on each of the worker nodes


## Cleanup

Delete everything

``` shell
vagrant destroy
```