Values for helm update:
---

```bash
[root@demo1 prometheus]# kubectl images -n prometheus -o json |grep image | awk '{print $2}' | sort |uniq
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/busybox:1.37.0"
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/grafana:12.4.0"
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/kube-state-metrics:v2.18.0"
"quay.io/kiwigrid/k8s-sidecar:2.5.0"
"quay.io/prometheus/alertmanager:v0.31.1"
"quay.io/prometheus/node-exporter:v1.10.2"
"quay.io/prometheus-operator/prometheus-config-reloader:v0.89.0"
"quay.io/prometheus-operator/prometheus-operator:v0.89.0"
"quay.io/prometheus/prometheus:v3.10.0"

[root@demo1 prometheus]# ls
kube-prometheus-stack  kube-prometheus-stack-82.4.3.tgz  values.yaml
[root@demo1 prometheus]# helm upgrade prometheus ./kube-prometheus-stack -n prometheus -f ./values.yaml 

[root@demo1 prometheus]# diff values.yaml kube-prometheus-stack/values.yaml 
44,46c44,46
<         registry: jibutech-registry.cn-hangzhou.cr.aliyuncs.com
<         repository: ys1000/busybox
<         tag: "1.37.0"
---
>         registry: docker.io
>         repository: busybox
>         tag: "latest"
1021,1028c1021,1028
<     storage: 
<       volumeClaimTemplate:
<         spec:
<           storageClassName: openebs-hostpath
<           accessModes: ["ReadWriteOnce"]
<           resources:
<             requests:
<               storage: 10Gi
---
>     storage: {}
>     # volumeClaimTemplate:
>     #   spec:
>     #     storageClassName: gluster
>     #     accessModes: ["ReadWriteOnce"]
>     #     resources:
>     #       requests:
>     #         storage: 50Gi
1241,1252d1240
<   # Update init docker image
<   initChownData:
<     image:
<       registry: jibutech-registry.cn-hangzhou.cr.aliyuncs.com
<       repository: ys1000/busybox
<       tag: "1.37.0"
<  
<   # grafana image
<   image:
<     registry: jibutech-registry.cn-hangzhou.cr.aliyuncs.com
<     repository: ys1000/grafana
<     tag: "12.4.0"
1382,1388c1370,1376
<   persistence:
<     enabled: true
<     type: sts
<     storageClassName: "openebs-hostpath"
<     accessModes:
<       - ReadWriteOnce
<     size: 20Gi
---
>   # persistence:
>   #   enabled: true
>   #   type: sts
>   #   storageClassName: "storageClassName"
>   #   accessModes:
>   #     - ReadWriteOnce
>   #   size: 20Gi
1390c1378
<   #      - kubernetes.io/pvc-protection
---
>   #     - kubernetes.io/pvc-protection
2519,2522d2506
<   image:
<     registry: jibutech-registry.cn-hangzhou.cr.aliyuncs.com
<     repository: ys1000/kube-state-metrics
<     tag: "v2.18.0"
4389c4373
<     storageSpec: 
---
>     storageSpec: {}
4392,4398c4376,4382
<       volumeClaimTemplate:
<         spec:
<           storageClassName: openebs-hostpath
<           accessModes: ["ReadWriteOnce"]
<           resources:
<             requests:
<               storage: 50Gi
---
>     #  volumeClaimTemplate:
>     #    spec:
>     #      storageClassName: gluster
>     #      accessModes: ["ReadWriteOnce"]
>     #      resources:
>     #        requests:
>     #          storage: 50Gi



```
