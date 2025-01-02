To get the kubelet configuration static Pod path

``` shell
ps -ef | grep kubelet | grep config
```

look for --config=/var/lib/kubelet/config.yaml

cat that file and look for: staticPodPath