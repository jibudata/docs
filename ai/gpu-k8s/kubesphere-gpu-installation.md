## KubeSphere 最佳实战：openEuler 22.03 LTS SP3 安装 NVIDIA 显卡驱动

> **2024 年****云原生**[**运维**](https://cloud.tencent.com/solution/operation?from_column=20065&from=20065)**实战文档 99 篇原创计划**  *第 024 篇* **｜玩转 AIGC「2024」系列** *第 005 篇*

你好，欢迎来到**运维有术**。

今天分享的内容是 **KubeSphere 最佳实战「2024」** 系列文档中的 **openEuler 22.03 LTS SP3 安装 NVIDIA 显卡驱动**。

本文将详细介绍如何在操作系统 openEuler 22.03 LTS SP3 手工安装 NVIDIA 显卡驱动。

> **实战**[**服务器**](https://cloud.tencent.com/product/cvm?from_column=20065&from=20065)**配置(架构1:1复刻小规模生产环境，配置略有不同)**

|      主机名      |      IP       | CPU  | 内存 | 系统盘 | 数据盘 |                    用途                    |
| :--------------: | :-----------: | :--: | :--: | :----: | :----: | :----------------------------------------: |
|  ksp-control-1   | 192.168.9.91  |  4   |  8   |   40   |  100   |        KubeSphere/k8s-control-plane        |
|  ksp-control-2   | 192.168.9.92  |  4   |  8   |   40   |  100   |        KubeSphere/k8s-control-plane        |
|  ksp-control-3   | 192.168.9.93  |  4   |  8   |   40   |  100   |        KubeSphere/k8s-control-plane        |
|   ksp-worker-1   | 192.168.9.94  |  4   |  16  |   40   |  100   |               k8s-worker/CI                |
|   ksp-worker-2   | 192.168.9.95  |  4   |  16  |   40   |  100   |                 k8s-worker                 |
|   ksp-worker-3   | 192.168.9.96  |  4   |  16  |   40   |  100   |                 k8s-worker                 |
|  ksp-storage-1   | 192.168.9.97  |  4   |  8   |   40   |  300+  |      ElasticSearch/Ceph/Longhorn/NFS/      |
|  ksp-storage-2   | 192.168.9.98  |  4   |  8   |   40   |  300+  |        ElasticSearch//Ceph/Longhorn        |
|  ksp-storage-3   | 192.168.9.99  |  4   |  8   |   40   |  300+  |        ElasticSearch//Ceph/Longhorn        |
| ksp-gpu-worker-1 | 192.168.9.101 |  4   |  16  |   40   |  100   |      k8s-worker(GPU NVIDIA Tesla M40)      |
| ksp-gpu-worker-2 | 192.168.9.102 |  4   |  16  |   40   |  100   |     k8s-worker(GPU NVIDIA Tesla P100)      |
|   ksp-registry   | 192.168.9.90  |  4   |  8   |   40   |  200   |              Harbor 镜像仓库               |
|  ksp-gateway-1   | 192.168.9.103 |  2   |  4   |   40   |        |  自建应用服务代理网关/VIP：192.168.9.100   |
|  ksp-gateway-2   | 192.168.9.104 |  2   |  4   |   40   |        |  自建应用服务代理网关/VIP：192.168.9.100   |
|     ksp-mid      | 192.168.9.105 |  4   |  8   |   40   |  100   | 部署在 k8s 集群之外的服务节点（Gitlab 等） |
|       合计       |      15       |  56  | 152  |  600   |  2000  |                                            |

> **实战环境涉及软件版本信息**

- 操作系统：**openEuler 22.03 LTS SP3 x86_64**
- KubeSphere：**v3.4.1**
- Kubernetes：**v1.28.8**
- KubeKey:  **v3.1.1**
- Containerd：**1.7.13**
- NVIDIA Container Toolkit：**1.15**
- NVIDIA 显卡： **P100 16G 和 M40 24G**

### 1. 前置条件

#### 1.1 操作系统初始化配置

请参考 [Kubernetes 集群节点 openEuler 22.03 LTS SP3 系统初始化指南](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FYDnvnuTqYfmgvF3HGOJ4WQ&objectId=2419926&objectType=1&contentType=undefined)，完成操作系统初始化配置。

初始化配置指南中没有涉及操作系统升级的任务，在能联网的环境初始化系统的时候**一定要升级操作系统**，然后重启节点。

#### 1.2 安装显卡驱动编译工具

```bash
yum install gcc make kernel-devel
```

#### 1.3 安装显卡驱动依赖包

```bash
yum install vulkan-loader
```

**可选安装项**，不安装该系统包时会出现以下警告提示，但不影响安装和使用。

![nvidia-installer-vulkan-loader](https://developer.qcloudimg.com/http-save/10642399/3f3e7b6cde9fd80c1705cb236d11c458.png)

nvidia-installer-vulkan-loader

### 2. 安装 NVIDIA GPU 驱动

生产环境建议选择 **.run** 格式的驱动安装包。从官方[NVIDIA 显卡驱动下载地址](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fwww.nvidia.cn%2FDownload%2Findex.aspx%3Flang%3Dzh-cn&objectId=2419926&objectType=1&contentType=undefined)下载驱动 NVIDIA-Linux-x86_64-550.54.15.run，并上传到每个 GPU 节点。

- 下载选项

![nvidia-p100-driver-download](https://developer.qcloudimg.com/http-save/10642399/e2f0608d581eaae6e90d3c1f753a0bb8.png)

nvidia-p100-driver-download

- 550 版驱动支持显卡列表

![nvidia-driver-550-support-list](https://developer.qcloudimg.com/http-save/10642399/22676011c0dad476051025bea7404b78.png)

nvidia-driver-550-support-list

#### 2.1 安装显卡驱动

```bash
chmod u+x NVIDIA-Linux-x86_64-550.54.15.run
./NVIDIA-Linux-x86_64-550.54.15.run
```

初次执行，请按提示操作，然后重启服务器。

安装过程大部分截图如下：

![nvidia-installer-nouveau](https://developer.qcloudimg.com/http-save/10642399/983d3efea57e936d86651d044bfed55a.png)

nvidia-installer-nouveau

![nvidia-installer-nouveau-modprobe](https://developer.qcloudimg.com/http-save/10642399/119354cc11550c2432951bd5104a6a7e.png)

nvidia-installer-nouveau-modprobe

![nvidia-installer-nouveau-modprobe-written](https://developer.qcloudimg.com/http-save/10642399/df2f5056da7e62b590f5ed5d7e61e8fe.png)

nvidia-installer-nouveau-modprobe-written

选择 **Abort installation**，然后重启服务器。

![nvidia-installer-continue](https://developer.qcloudimg.com/http-save/10642399/43be0763a2d0166b8768656d249e70c5.png)

nvidia-installer-continue

服务器重启完成后，再次执行安装命令，会自动执行构建、安装的任务（截图不全）。

![nvidia-installer-32bit](https://developer.qcloudimg.com/http-save/10642399/de7a892ed5147506cb8cc73a0a1ff959.png)

nvidia-installer-32bit

![nvidia-installer-complete](https://developer.qcloudimg.com/http-save/10642399/3fbbc017e5a1f7d7335975c88c69e55f.png)

nvidia-installer-complete

**建议驱动安装完成后，再次重启服务器。**

#### 2.2 验证显卡驱动

- 执行下面的命令

```bash
nvidia-smi
```

**Tesla M40** 节点，正确执行后，输出结果如下：

```bash
$ nvidia-smi
Thu May 19 08:59:57 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.54.15              Driver Version: 550.54.15      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla M40 24GB                 Off |   00000000:00:10.0 Off |                    0 |
| N/A   37C    P0             65W /  250W |       0MiB /  23040MiB |    100%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

**Tesla P100** 节点，正确执行后，输出结果如下：

```bash
$ nvidia-smi
Thu May 19 09:19:19 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.54.15              Driver Version: 550.54.15      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla P100-PCIE-16GB           Off |   00000000:00:10.0 Off |                    0 |
| N/A   40C    P0             31W /  250W |       0MiB /  16384MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

### 3. 自动化 Shell 脚本

文章中所有操作步骤，已全部编排为自动化脚本，包含以下内容（因篇幅限制，不在此文档中展示）：

- Ansible 初始化 GPU 节点操作系统基础配置
- Ansible 初始化磁盘配置
- Ansible 安装 NVIDIA 显卡驱动依赖
