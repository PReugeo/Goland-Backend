## 资源模型

在 kubernetes 中，任何可以被申请、分配，最终被使用的对象，都是 kubernetes 中的资源，比如 CPU、内存。

而且针对每一种被 kubernetes 所管理的资源，都会被赋予一个【资源类型名】，这些名字都是符合 RFC 1123 规则的，比如 CPU，它对应的资源名全称为 http://kubernetes.io/cpu（在展示时一般直接简写为 cpu）；GPU 资源对应的资源名为 http://alpha.kubernetes.io/nvidia-gpu

除了名字以外，针对每一种资源还会对应有且仅有一种【基本单位】。这个基本单位一般是在 kubernetes 组件内部统一用来表示这种资源的数量的，比如 memory 的基本单位就是字节。

所有的资源类型，又可以被划分为两大类：可压缩(compressible)和不可压缩(incompressible)的。其评判标准就在于：

如果系统限制或者缩小容器对可压缩资源的使用的话，只会影响服务对外的服务性能，比如 CPU 就是一种非常典型的可压缩资源。

对于不可压缩资源来说，资源的紧缺是有可能导致服务对外不可用的，比如内存就是一种非常典型的不可压缩资源。

## 资源类型

目前 kubernetes 默认带有两类基本资源：

• CPU

• memory

其中 CPU，不管底层的机器是通过何种方式提供的（物理机 or 虚拟机），一个单位的 CPU 资源都会被标准化为一个标准的 "Kubernetes Compute Unit" ，大致和 x86 处理器的一个单个超线程核心是相同的。

CPU 资源的基本单位是 millicores，因为 CPU 资源其实准确来讲，指的是 CPU 时间。所以它的基本单位为 millicores，1 个核等于 1000 millicores（1000 m）。也代表了 kubernetes 可以将单位 CPU 时间细分为 1000 份，分配给某个容器。

另外，kubernetes 针对用户的自定制需求，还为用户提供了 device plugin 机制，让用户可以将资源类型进一步扩充，比如现在比较常用的 nvidia gpu 资源。这个特性可以使用户能够在 kubernetes 中管理自己业务中特有的资源，并且无需修改 kubernetes 自身源码。

而且 kubernetes 自身后续还会进一步支持更多非常通用的资源类型，比如网络带宽、存储空间、存储 iops 等等。

## K8S 计算资源管理

Kubernetes 在使用 `get node -o yaml` 可以获取 node 有关资源类型的描述:

![image-20211108190914946](/Users/yanjigang01/Golang-Backend/云原生/K8S/image/image-20211108190914946.png)

`capacity` 表示这台 Node 的资源真实量，由每台机器 `kubelet` 确定，通过 [cAdvisor](https://github.com/google/cadvisor) 子模块获取机器上面的各种信息，然后通过 kubelet 上报给 apiserver。

`allocatable` 则表示这台 Node 可以被 K8S 分配的真实资源量，由 [Node Allocatable](https://kubernetes.io/zh/docs/tasks/administer-cluster/reserve-compute-resources/) 特性确认，该特性有助于为系统守护进程预留计算资源，Node 中预留的资源主要为这两类资源：

* kubernetes daemon
* system daemon

### K8S 如何实现 Node Allocatable

K8S 底层是通过 cgroup 特性来实现资源隔离与限制，Node Allocatable 也是通过 cgroup 实现的资源预留。

在默认情况下，针对每一种基本资源（CPU、memory），kubernetes 首先会创建一个根 cgroup 组，作为所有容器 cgroup 的根，名字叫做 kubepods。这个 cgroup 就是用来限制这台计算节点上所有 pod 所使用的资源的。默认情况下这个 kubepods cgroup 组所获取的资源就等同于该计算节点的全部资源。

如果开启了 `Kube-Reserved` 和 `System-Reserved` 特性时，系统就会根据指定的 cgroup 和预留的资源量来与 kubepods 共同分配这台 Node 的资源。

**Kube-Reserved**

- **Kubelet 标志**: `--kube-reserved=[cpu=100m][,][memory=100Mi][,][ephemeral-storage=1Gi][,][pid=1000]`
- **Kubelet 标志**: `--kube-reserved-cgroup=`

**System-Reserved**

- **Kubelet 标志**: `--system-reserved=[cpu=100m][,][memory=100Mi][,][ephemeral-storage=1Gi][,][pid=1000]`

- **Kubelet 标志**: `--system-reserved-cgroup=`

  

在计算 memory 资源的 allocatable 数量时，除了 Kube-Reserved，System-Reserved 之外，还要考虑另外一个因素。那就是 Eviction Threshold。

### Eviction Threshold

Eviction Threshold 对应的是 kubernetes 的 eviction policy 特性。该特性也是 kubernetes 引入的用于保护物理节点稳定性的重要特性。当机器上面的【内存】以及【磁盘资源】这两种不可压缩资源严重不足时，很有可能导致物理机自身进入一种不稳定的状态，这个很显然是无法接受的。

所以 kubernetes 官方引入了 eviction policy 特性，该特性允许用户为每台机器针对【内存】、【磁盘】这两种不可压缩资源分别指定一个 eviction hard threshold, 即资源量的阈值。

比如我们可以设定内存的 eviction hard threshold 为 100M，那么当这台机器的内存可用资源不足 100M 时，kubelet 就会根据这台机器上面所有 pod 的 QoS 级别（Qos 级别下面会介绍），以及他们的内存使用情况，进行一个综合排名，把排名最靠前的 pod 进行迁移，从而释放出足够的内存资源。

所以针对内存资源，它的 allocatable 应该是 [capacity] - [kube-reserved] - [system-reserved] - [hard-eviction]

## Pod 资源管理与分配

创建 Pod 时可以使用 `request` 和 `limit` 指定 CPU 和内存的资源量，GPU 则需要安装相应的插件

* request：针对这种资源，这个容器希望能获取到的最少的量，实际上 CPU reqeust 是可以通过 cpu.shares 特性来实现的，但是内存资源时不可压缩的，如果设置过高的 request 可能导致其他 pod 连 request 的资源都无法满足

* limit：指容器对这个资源获取的上限

  但是这两种资源在针对容器使用量超过 limit 所表现出的行为也是不同的。

  对 CPU 来说，容器使用 CPU 过多，内核调度器就会切换，使其使用的量不会超过 limit。

  对内存来说，容器使用内存超过 limit，这个容器就会被 OOM kill 掉，从而发生容器的重启。

在容器没有指定 request 的时候，request 的值和 limit 默认相等。

而如果容器没有指定 limit 的时候，request 和 limit 会被设置成的值则根据不同的资源有不同的策略。

### Pod QoS 分类

QoS 级别在 K8S 驱逐调度方面表现为 pod 的相关优先级

kubernetes 支持用户容器通过 request、limit 两个字段指定自己的申请资源信息。那么根据容器指定资源的不同情况，Pod 也被划分为 3 种不同的 QoS 级别。分别为：

• Guaranteed：CPU、内存的 limit、request 必须被指定并且相等

• Burstable：pod 中至少有一个容器指定了 cpu 或者内存的 request 信息

• BestEffort

 Guaranteed level 的 Pod 是优先级最高的，系统管理员一般对这类 Pod 的资源占用量比较明确。

 Burstable level 的 Pod 优先级其次，管理员一般知道这个 Pod 的资源需求的最小量，但是当机器资源充足的时候，还是希望他们能够使用更多的资源，所以一般 limit > request。

 BestEffort level 的 Pod 优先级最低，一般不需要对这个 Pod 指定资源量。所以无论当前资源使用如何，这个 Pod 一定会被调度上去，并且它使用资源的逻辑也是见缝插针。当机器资源充足的时候，它可以充分使用，但是当机器资源被 Guaranteed、Burstable 的 Pod 所抢占的时候，它的资源也会被剥夺，被无限压缩，在资源不够时也是第一个被驱逐。

### Pod 资源分配原理

kubelet 就是基于 【pod 申请的资源】 + 【pod 的 QoS 级别】来最终为这个 pod 分配资源的。

而分配资源的根本方法就是基于 cgroup 的机制

kubernetes 在拿到一个 pod 的资源申请信息后，针对每一种资源，他都会做如下几件事情：

* 对 pod 中的每一个容器，都创建一个 container level cgroup（注：这一步真实情况是 kubernetes 向 docker daemon 发送命令完成的）。

* 然后为这个 pod 创建一个 pod level cgroup ，它会成为这个 pod 下面包含的所有 container level cgroup 的父 cgroup。

* 最终，这个 pod level cgroup 最终会根据这个 pod 的 QoS 级别，可能被划分到某一个 QoS level cgroup 中，成为这个 QoS level cgroup 的子 cgroup。

* 整个 QoS level cgroup 还是所有容器的根 cgroup - kubepods 的子 cgroup。

![v2-6f6fb4936d0fc281525c6cc61501864c_720w](/Users/yanjigang01/Golang-Backend/云原生/K8S/image/v2-6f6fb4936d0fc281525c6cc61501864c_720w.jpg)



