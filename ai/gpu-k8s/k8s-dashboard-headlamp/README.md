Notes:

```
helm upgrade headlamp ./headlamp-0.40.0.tgz -n kube-system  -f ./values.yaml

[root@demo1 headlamp]# diff values.yaml headlamp/values.yaml 
10c10
<   registry: jibutech-registry.cn-hangzhou.cr.aliyuncs.com
---
>   registry: ghcr.io
12c12
<   repository: ys1000/headlamp
---
>   repository: headlamp-k8s/headlamp
190c190
<   type: NodePort
---
>   type: ClusterIP
200c200
<   nodePort: 30080
---
>   nodePort: null


[root@demo1 headlamp]# kubectl images -n kube-system -o json  |grep image |awk '{print $2}' | sort |uniq
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/headlamp:v0.40.0"
"registry.cn-beijing.aliyuncs.com/kubesphereio/cni:v3.27.4"
"registry.cn-beijing.aliyuncs.com/kubesphereio/coredns:1.9.3"
"registry.cn-beijing.aliyuncs.com/kubesphereio/k8s-dns-node-cache:1.22.20"
"registry.cn-beijing.aliyuncs.com/kubesphereio/kube-apiserver:v1.28.15"
"registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controller-manager:v1.28.15"
"registry.cn-beijing.aliyuncs.com/kubesphereio/kube-controllers:v3.27.4"
"registry.cn-beijing.aliyuncs.com/kubesphereio/kube-proxy:v1.28.15"
"registry.cn-beijing.aliyuncs.com/kubesphereio/kube-scheduler:v1.28.15"
"registry.cn-beijing.aliyuncs.com/kubesphereio/node:v3.27.4"
"registry.cn-shanghai.aliyuncs.com/jibutech/snapshot-controller:v5.0.1"
```
