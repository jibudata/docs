NOTES:


openebs hostpath installation

```
kubectl apply -f ./openebs-operator.yaml

[root@demo1 ~]# kubectl -n openebs get pods
NAME                                            READY   STATUS    RESTARTS       AGE
openebs-localpv-provisioner-77c5dbd769-x92f8    1/1     Running   11 (32d ago)   312d
openebs-ndm-6ztng                               1/1     Running   0              312d
openebs-ndm-cluster-exporter-59c7444d8c-pvmck   1/1     Running   0              312d
openebs-ndm-node-exporter-xfvzq                 1/1     Running   0              312d
openebs-ndm-operator-fb96994b4-xd9w2            1/1     Running   0              312d
[root@demo1 ~]# kubectl -n openebs get sc
NAME                        PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-hostpath            openebs.io/local             Delete          WaitForFirstConsumer   false                  312d

[root@demo1 ~]# kubectl images -n openebs -o json  |grep image |awk '{print $2}' | sort |uniq
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/openebs-node-disk-exporter:2.1.0"
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/openebs-node-disk-manager:2.1.0"
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/openebs-node-disk-operator:2.1.0"
"jibutech-registry.cn-hangzhou.cr.aliyuncs.com/ys1000/openebs-provisioner-localpv:3.4.0"

```

nfs installation

```
 helm install nfs-subdir-external-provisioner ./nfs-subdir-external-provisioner-4.0.16.tgz --namespace=nfs-storage --create-namespace -f ./nfs-values.yaml --set nfs.server=192.168.3.134 --set nfs.path=/home/k8s-nfs-share --set storageClass.name=nfs-client 
 # registry.cn-shanghai.aliyuncs.com/ys1000/nfs-subdir-external-provisioner:v4.0.2
```
