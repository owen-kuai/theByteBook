# 7.7.1 资源模型

## 1. 资源的分类
与调度最具有紧密关系的资源就是 CPU 和内存，根据资源不足表现的差异，这两类资源又可分为：

- **可压缩的资源**：典型的是 CPU，此类资源超限时，容器进程使用 CPU 会被限制，应用表现变得卡顿，业务延迟明显增加，但**容器进程不会被杀掉**。CPU 资源其实准确来讲，指的是 CPU 时间。所以它的基本单位为 millicores，1 个核等于 1000 millicores。也代表了 kubernetes 可以将单位 CPU 时间细分为 1000 份。

- **不可压缩的资源**：典型的是内存，这类资源容器之间无法共享，完全独占。这也就意味着资源一旦耗尽或者不足，容器进程一定产生 OOM 问题（Out of Memory，内存溢出）并**被杀掉**。内存基本单位是字节，计量方式有多种写法，譬如使用 M（Megabyte）、Mi（Mebibyte）以及不带单位的数字，以下表达式所代表的是相同的值。

```plain
128974848, 129e6, 129M, 123Mi
```
注意 Mebibyte 和 Megabyte 的区分，123 Mi = `123*1024*1024 B` 、123 M = `1*1000*1000 B`，显然使用带小 i 的更准确。

## 2. 资源扩展与设备插件

Kubernetes 在 Pod 中并没有专门为 GPU 设置一个专门的资源类型，而是使用了一个特殊字段（Extended Resource），来负责传递 GPU 资源。

为了能让调度器知道这个扩展资源在每台节点的可用量信息，节点本身就要通过 APIServer 汇报自身的资源情况。如下所示，通过发送 PATCH 请求，为节点增加自定义的资源类型。

```bash
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: k8s-master:8080

[
  {
    "op": "add",
    "path": "/status/capacity/nvidia.com~1gpu",
    "value": "4"
  }
]
```

输出展示了刚才扩展的 nvidia.com/gpu 资源：

```bash
$ kubectl describe node <your-node-name>
...
Capacity:
  cpu: 2
  memory: 2049008Ki
  nvidia.com/gpu: 4
...
```

接下来就可以在 Pod 中使用扩展资源了，比如下面这个例子。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: cuda-container
      image: nvidia/cuda:10.0-base
      resources:
        limits:
          nvidia.com/gpu: 1
```

可以看到，上面 Pod resources 中，GPU 的资源名称为 nvidia.com/gpu。也就是说这个 Pod 声明了要使用 nvidia 类型的 GPU。容器启动的时候，再通过挂载宿主机的 GPU 驱动，就能直接使用 GPU 资源了。

在 Kubernetes 支持的 GPU 方案中，你并不需要去操作上述 Extended Resource 的逻辑，Kubernetes 中，所有的硬件加速设备的管理都通过 Device Plugin 插件来支持，也包括对该硬件的 Extended Resource 进行汇报的逻辑。

各个硬件设备商开发的 Device Plugin 插件以 DaemonSet 方式运行在集群当中，Kubelet 通过 gRPC 接口与 Device Plugin 插件交互，实现设备发现、状态更新、资源上报等。最后，Pod 通过 resource request、limit 显示声明使用即可，如同 CPU、MEM 一样。

:::tip 问题

你注意到 Device Plugin 的问题了么？

Pod 只能过"nvidia.com/gpu:2" 这种简单的“计数形式”，来申请 2 块 GPU，但是关于 2 块卡分别是什么型号、是否拓扑最优、是否共享/独享等等内容，都没有能力进行选择。

在这些特殊场景的催化下，Nvidia、Intel 等头部厂商联合推出了 DRA（Dynamic Resource Allocation，动态资源分配）机制，用于解决现有 Device Plugin 的不足。

:::

## 3. 节点资源管理

由于每台节点上都运行着 kubelet、docker 或 containerd 等 Kubernetes 基础服务，因此在进行资源管理和调度时，需要为此类基础服务预留资源。预留之后的剩余资源才是 Pod 真正可以使用的。

为节点上的基础服务预留多少资源，我们可以使用如下的几个 kubelet 参数来控制：

- --kube-reserved=[cpu=100m][,][memory=100Mi][,][ephemeral-storage=1Gi]：预留给 kubernetes 组件 CPU、内存和存储资源。
- --system-reserved=[cpu=100mi][,][memory=100Mi][,][ephemeral-storage=1Gi]：预留给操作系统的 CPU、内存和存储资源。

需要注意的是，考虑到 Kubernetes 驱逐机制（7.7.2 节介绍）的存在，kubelet 会确保节点上的资源使用率不会达到 100%，因此 Pod 实际可用的资源会再少一些。最终，一个节点资源分配如图 7-33 所示。节点可分配资源（Node Allocatable Resource）= 节点所有资源（Node Capacity） -（ Kubernetes 组件预留资源（Kube Reserved）-系统预留资源（System Reserved）- 为驱逐预留的资源（Eviction-Threshold）。

:::center
  ![](../assets/k8s-resource.svg)<br/>
  图 7-33 Node 资源逻辑分配图
:::








