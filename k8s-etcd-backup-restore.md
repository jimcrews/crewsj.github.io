Using my local cluster that is built using Virtualbox and Vagrant, I am going to be taking a backup of ETCD and restoring it. All commands will be run on the **controlplane** node.


# Install ETCDCTL

Install ETCDCTL in Ubuntu using:

``` bash
sudo apt install etcd-client
```

## Get ETCD Details

On the master node, list the ETCD details that will be required later

``` bash
cat /etc/kubernetes/manifests/etcd.yaml
```

``` text
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.1.17:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --experimental-initial-corrupt-check=true
    - --experimental-watch-progress-notify-interval=5s
    - --initial-advertise-peer-urls=https://192.168.1.17:2380
    - --initial-cluster=controlplane=https://192.168.1.17:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.1.17:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.1.17:2380
    - --name=controlplane
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

For backup, we need the locations for **trusted-ca-file**, **cert-file**, and **key-file**
For restore, we also need the locations for **data-dir**, and **listen-client-urls**

## List Command Help

Run `etcdctl` without any options to get info on using the version 3 API if not already

``` text
WARNING:
   Environment variable ETCDCTL_API is not set; defaults to etcdctl v2.
   Set environment variable ETCDCTL_API=3 to use v3 API or ETCDCTL_API=2 to use v2 API.
```

We will export this:

``` bash
export ETCDCTL_API=3
```

Running `etcdctl snapshot save -h` we get the options that will be required to create a snapshot

``` text
NAME:
	snapshot save - Stores an etcd node backend snapshot to a given file

USAGE:
	etcdctl snapshot save <filename> [flags]

OPTIONS:
  -h, --help[=false]	help for save

GLOBAL OPTIONS:
      --cacert=""				verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""					identify secure client using this TLS certificate file
      --command-timeout=5s			timeout for short running command (excluding dial timeout)
      --debug[=false]				enable client-side debug logging
      --dial-timeout=2s				dial timeout for client connections
  -d, --discovery-srv=""			domain name to query for SRV records describing cluster endpoints
      --endpoints=[127.0.0.1:2379]		gRPC endpoints
      --hex[=false]				print byte strings as hex encoded strings
      --insecure-discovery[=true]		accept insecure SRV records describing cluster endpoints
      --insecure-skip-tls-verify[=false]	skip server certificate verification (CAUTION: this option should be enabled only for testing purposes)
      --insecure-transport[=true]		disable transport security for client connections
      --keepalive-time=2s			keepalive time for client connections
      --keepalive-timeout=6s			keepalive timeout for client connections
      --key=""					identify secure client using this TLS key file
      --user=""					username[:password] for authentication (prompt if password is not supplied)
  -w, --write-out="simple"			set the output format (fields, json, protobuf, simple, table)
```


## Take Backup

We need to use sudo with -E to preserve the environment variable set for etcdctl version.

``` shell
sudo -E etcdctl snapshot save myback.db --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

> Lets create a pod after taking a backup so that we can see the pod is removed after a restore.

```
k run nginx --image=nginx
```

## Restore

Get the **data-dir** location from the /manifests/etcd.yaml.

Delete everything in the directory.

``` shell
sudo rm -Rf /var/lib/etcd
```

Check the help - `etcdctl snapshot restore -h`, and create restore command:

``` shell
sudo -E etcdctl snapshot restore myback.db --data-dir=/var/lib/etcd --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

I had to delete the data dir directory again. Then re-ran command successfully.

``` shell
Error: data-dir "/var/lib/etcd" exists
vagrant@controlplane:~$ sudo rm -Rf /var/lib/etcd
```