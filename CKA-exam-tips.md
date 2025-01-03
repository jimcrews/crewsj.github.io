### Check pod logs

``` shell
k logs etcd-master

# or view docker logs

docker logs 65919fb9ba6a9
# or
crictl logs 65919fb9ba6a9
```

### How man roles are there in all namespaces ?

use `--no-headers`, `wc` and `-l` for 'lines'
``` shell
k get roles -A --no-headers | wc -l
```

### Inspect certificate file

``` shell
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```