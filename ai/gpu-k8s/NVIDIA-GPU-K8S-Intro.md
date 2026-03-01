# The real-world guide to the NVIDIA GPU Operator for K8s AI

Ref: https://www.spectrocloud.com/blog/the-real-world-guide-to-the-nvidia-gpu-operator-for-kubernetes-ai 

## Meet the NVIDIA GPU Operator

AI is a ‘killer app’ for Kubernetes. In our [2024 State of Production Kubernetes](https://info.spectrocloud.com/2024-state-of-production-kubernetes) research, more than two-thirds of respondents said that “Kubernetes is key to our company taking full advantage of AI”. The vast majority said they were either already running AI in production, or planned to in the coming year.

Graphics Processing Units (GPUs) are the specialized hardware that’s central to running AI workloads on Kubernetes (or anywhere else). And the vendor of choice for GPUs is NVIDIA.

In the world of cloud native, we need to find a way to give our AI workloads running in containers on Kubernetes access to the GPU resources required. The [**NVIDIA**](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html) GPU Operator is what makes that possible. There are other GPU operators — like Intel’s Device Plugin Operator and AMD’s GPU Operator — but we will focus on NVIDIA here, given its popularity and adoption. 

## The operator framework makes it all possible

Kubernetes has an Operator Framework that allows building K8s operators to easily configure and manage acceleration hardware, such as FPGAs and GPUs. 

NVIDIA has been doing this for quite some time: it actually started enabling GPUs in the Container Runtime Ecosystem back in 2016 with the NVIDIA-Docker project. It allowed driver-agnostic CUDA images and provided a Docker command line wrapper that mounted the user-mode components of the driver and the GPU device files into the container at launch. 

Why is this important? Because we’ve come a long way! All of these historical technical steps are now nicely wrapped around the NVIDIA GPU Operator. This simplifies configuring your K8s cluster to consume NVIDIA GPUs, without having to pre-install drivers, or other required components, on your nodes. 

I’ve spent a lot of time working with GPUs in Kubernetes, helping Spectro Cloud customers build their AI/ML pipelines on K8s for training and inferencing. NVIDIA’s GPU Operator is my go-to solution when working with multiple private and public clouds, as well as edge. 

Unfortunately, NVIDIA's documentation is daunting when you’re getting started. In this blog we’ll try to simplify the complexity and save you some time.

## Why does the operator matter? Abstraction!

The GPU Operator is a Kubernetes Operator that provisions and manages NVIDIA GPUs on top of Kubernetes. This ultimately exposes the GPUs as resources available to be used by your Kubernetes nodes. 

The GPU Operator completely abstracts the platform that you are running it on, which means that you no longer need to worry if you are [running on AWS](https://www.spectrocloud.com/solutions/aws) or your [bare metal](https://www.spectrocloud.com/solutions/bare-metal) servers, in your datacenter or even at the [edge](https://www.spectrocloud.com/solutions/edge). This means you don’t need to worry about:

- Different cloud providers using different sets of node images and driver versions
- Outdated vendor drivers that don’t support the GPU, or the CUDA version, that you want to run
- Different GPUs requiring different libraries and binaries to be available on the host

Ultimately this makes setting up GPU drivers and their configurations on Kubernetes a lot easier. It also unlocks a load of advanced features like [vGPU](https://www.nvidia.com/en-us/data-center/virtual-solutions/), [Multi-Instance GPU (MIG)](https://www.nvidia.com/en-us/technologies/multi-instance-gpu/), and [GPU Time-Slicing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html), as well as simplifying advanced network and storage setups with GPUDirect RDMA and GPU Direct storage.

## A deeper dive into the individual components

The GPU Operator is composed of multiple components. Understanding what they are and what they do is key to building bulletproof configurations that fit your use-cases:

- NVIDIA GPU Driver: Enables interaction between the GPU and the operating system
- NVIDIA Container Toolkit: Allows interaction with the GPU from containerized environments
- NVIDIA Device Plugin: Exposes the GPU to the kubelet API via the device plugin mechanism
- NVIDIA GPU feature discovery: Detects GPUs on nodes and labels them as GPU-enabled
- NVIDIA Data Center GPU Manager (DCGM) and Exporter: Exposes the GPU metrics (i.e., pretty Grafana dashboards) 
- NVIDIA Multi Instance GPU (MIG) Manager: Partitions GPUs into different MIG configurations/profiles
- NVIDIA GPU Direct Storage (GDS): Enables direct data transfers between storage devices and GPU memory. Typically used for high-performance AI/ML and HPC workloads.
- NVIDIA GPU Operator: Acts as an orchestrator of the entire GPU Operator stack

This is a long list, and essentially each component can be individually installed, so let’s break them into the ones that are **critical** vs the ones that are **optional**, and typically only used in more advanced use-cases. 

In practice only the driver, container toolkit, and device plugin are critical to understand as they are the foundation of how the operator works.

![GPU critical components](https://cdn.prod.website-files.com/64196dbe03e13c204de1b1c8/67c9ef21d3b1a49af9216925_AD_4nXekro-1KuDbKhOhGfiAYuSVMy4IxkZsM21b0BLG7kEfSL7R4PmIAvJGdTg-iBNm1eRJmivUBIjOdAeLOlsmW20TmTDw7_hxf5IDpjx11gmySw2apbDpw9xddZy92LIDMuW2EapH6A.png)

This image simplifies how these components work together. Just as you find abstractions for interacting with non-accelerated hardware (assembly, C, libraries and so on), the same occurs for GPUs (PTX, CUDA, CUDA drivers, etc). 

## Installing the core components of the GPU operator

### Node feature discovery

Once you deploy the GPU Operator, it starts by initialising the gpu-operator pod, which is a supervision and reconciliation service, together with the node-feature-discovery service, which creates a pod on each cluster node that detects attached GPUs, and once it does, adds labels to the relevant nodes. 

These labels help with scheduling, but also ensure that only the required components are deployed on those specific nodes. For example, if you install the GPU Operator on a K8s cluster with nodes that have no GPUs, then the GPU Operator will detect it and not install any further components.

Once the feature discovery runs and tags the GPU-enabled nodes, the three other important components are installed: the driver, container toolkit, and device plugin. These are all dependent on each other and are installed sequentially.

### NVIDIA driver

The **NVIDIA driver** is essentially a [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) that gets deployed onto each node that has a GPU. 

It then deploys a container that runs the nvidia-installer, which installs the kernel module onto the host OS. The driver is both OS and Kernel dependent, so you have to make sure that these are compatible. 

You can inspect the container logs to confirm that the driver is successfully installed and once it is, this container will continue running. 

There’re specific binaries on that container that you can leverage, such as the [**nvidia-smi**](https://docs.nvidia.com/deploy/nvidia-smi/index.html). The nvidia-smi is a very useful tool for identifying various issues and gathering general information about the GPU, such as the GPU temperature, running processes and CUDA version. 

CUDA is a computing platform for interacting with GPUs. Consider it an API for interacting with GPUs, and its version is directly tied to the driver version. A typical misconception is that the nvidia-smi gets installed on the host, but this is not the case when using the GPU Operator. For you to access it, you will need to exec into the driver installation pod.

### NVIDIA Container Toolkit

The second component, **the NVIDIA Container Toolkit** comes up at the same time as the NVIDIA driver, but it initially continually loops an init container that checks if the driver is installed. 

Installation of the container toolkit only proceeds once the driver is installed. So if you are in a situation where the container toolkit is not passing the init stage, it’s most likely because your driver is not properly installed. Inspect the driver installation pod logs for further troubleshooting. Once the driver is found, the container toolkit gets installed onto the host via a volume mount. 

In very simplistic terms, the container toolkit is a combination of files that need to be present on the host together with the configuration of the **nvidia-container-runtime**, which injects a runtime hook into containerd (by default). 

This then allows containerd to detect when a container requests GPU resources, so it can inject the required CUDA libraries and driver bindings. 

One of the most important configurations of the container toolkit is to make sure that you use the correct path for containerd, which by default is “/etc/containerd/config.toml”, but if for some reason the distribution you are using has a different path, you will have to adjust accordingly. For example, K3s stores containerd configuration in “/var/lib/rancher/k3s/agent/etc/containerd/config.toml”.

### NVIDIA Device Plugin

The third critical component is **the NVIDIA Device Plugin**, which is dependent on the container toolkit being installed successfully because it consumes the libraries and utils shared via the volume mount. 

This particular component exposes the GPUs as first-class resources in Kubernetes, and it uses the Kubernetes device plugin framework to communicate with the kubelet. This means that from this point onwards the GPUs are registered as schedulable and allocatable resources.

When you are looking at more advanced features, such as MIG, this is the component that is required for MIG to run on top of.

Once you have all of these components installed — and assuming you haven’t configured any advanced features — the GPU operator brings up the validator component, spawning another pod that checks that the drivers and plugins are correctly installed and working. 

You are now graduated to go make the most of your GPUs with the NVIDIA GPU Operator! 🥳

## Powering up with optional features

As you start dipping your toes further into GPUs and the Operator, you may want to explore optional features. 

### Monitoring with DCGM Exporter

The first is the DCGM (Data Center GPU Manager) Exporter, which monitors the status and health of NVIDIA GPUs in the cluster by collecting GPU metrics (usage, temperature, memory utilization, kWh) and exposing them to monitoring systems such as Prometheus and Grafana dashboards.

### Optimizing utilization with MIG, vGPU and time-slicing

As you scale with more GPUs and more applications (potentially across thousands of clusters in a GPUaaS use case), it becomes more important to optimize how you’re utilizing these expensive resources. 

This is where advanced features and components come in like Multi-instance GPU (MIG), GPUDirect RDMA (Remote Direct Memory Access), GPUDirect Storage and Bin Packing. 

The GPU scheduling topic is very hot and this will be a discussion for another blog, but for now we’ll give you a teaser.

By definition GPUs in Kubernetes cannot be overcommitted, and workloads cannot request parts of a GPU — an entire GPU unit must be requested. This can kill your utilization in cases where a single workload may not require an entire GPU. To address this there are a few advanced features that you can explore:

1. **MIG (Multi-instance GPU):** This slices a physical GPU into multiple independent instances (up to seven, depending on the GPU), each with its own dedicated resources (memory, compute cores, and cache), improving workload isolation and efficiency by avoiding contention between workloads. The major issue with this feature is that only a few NVIDIA GPUs support this, mostly the Ampere and Hopper families. 
2. **vGPU (Virtual GPU):** This enables multiple virtual machines to have direct access to a single GPU. It requires a considerable amount of setting up and it must run on a supported hypervisor. This capability is supported in a wide range of NVIDIA GPUs but it does require NVIDIA AI Enterprise licensing. The good news is that NVIDIA did recently [open-source the Linux Driver Code For vGPU](https://lore.kernel.org/kvm/20240924164151.GJ9417@nvidia.com/T/#m219a9503d570772306defac223545c006d43cb0a), so watch this space!
3. **GPU Time-Slicing:** This is a mechanism where a physical GPU is shared between multiple workloads by rapidly switching execution contexts, which means that each container gets a time slice (configured via ConfigMaps) of the GPU, giving the impression that multiple workloads are concurrently running. This is actually enabled at the CUDA level so this is supported by all NVIDIA GPUs, however, it must be said that there’s essentially no isolation given that workloads share the same context and memory. It’s also not ideal for latency-sensitive workloads.

Suffice it to say that getting the most from these features for your use case requires some careful planning.

## Avoiding common pitfalls

As you install the NVIDIA GPU Operator, there are a few things you really need to check to avoid hours of troubleshooting:

1. **Ensure your GPU is supported.** Not all NVIDIA GPUs are [supported](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html#supported-nvidia-data-center-gpus-and-systems) by the Operator — including Jetsons, for example.
2. **Check your CUDA version.** Make sure that the [CUDA version](https://docs.nvidia.com/deploy/cuda-compatibility/) you need, for example a specific version for PyTorch, is fully supported by the [NVIDIA driver](https://www.nvidia.com/en-us/drivers/) that you are installing. Each GPU Operator version comes with a [compatibility matrix](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/24.6.2/platform-support.html#gpu-operator-component-matrix), which shows the different driver and component versions that are supported.
3. **Match OS to driver.** Ensure that the NVIDIA driver you’re using is supported by the underlying operating system, by [checking that the driver image tag](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/driver/tags) exists for your operating system of choice. New OS versions may take a bit longer to have an NVIDIA GPU driver available.

## Conclusion

Managing GPUs in Kubernetes clusters can be a daunting task, especially at scale. Traditional methods that require manual installations and scripts to configure GPU drivers and other components at the OS level are time-consuming, error-prone and, most worryingly, create snowflakes. 

Enterprises like yours need a way to deploy validated NVIDIA Kubernetes GPU Operator configurations across multiple clusters, across multiple platforms.

Spectro Cloud Palette with its Cluster Profiles concept and its [native support for the NVIDIA GPU Operator](https://docs.spectrocloud.com/integrations/packs/?pack=nvidia-gpu-operator) allows you to build exactly that. 

So whether you’re looking to build GPUaaS on bare metal, or deploy K8s AI workflows for inference at the edge, [come talk to us and see how we can help.](https://www.spectrocloud.com/get-started)

Nov 28, 2025
