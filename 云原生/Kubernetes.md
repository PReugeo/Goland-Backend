# 概述

Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

通过Kubernetes你可以：

- 快速部署应用
- 快速扩展应用
- 无缝对接新的应用功能
- 节省资源，优化硬件资源的使用

**Kubernetes 协调一个高可用计算机集群，每个计算机作为独立单元互相连接工作。** Kubernetes 中的抽象允许您将容器化的应用部署到集群，而无需将它们绑定到某个特定的独立计算机。为了使用这种新的部署模型，应用需要以将应用与单个主机分离的方式打包：它们需要被容器化。与过去的那种应用直接以包的方式深度与主机集成的部署模型相比，容器化应用更灵活、更可用。 **Kubernetes 以更高效的方式跨集群自动分发和调度应用容器。** Kubernetes 是一个开源平台，并且可应用于生产环境。

一个 Kubernetes 集群包含两种类型的资源:

- **Master** 调度整个集群。协调集群中的所有活动，例如调度应用、维护应用的所需状态、应用扩容以及推出新的更新
- **Nodes** 负责运行应用。它们是一个虚拟机或者物理机。每个 Node 都有 Kubelet，它管理 Node 而且是 Node 与 Master 的通信代理，Node 还应该具有用于处理容器操作的工具，例如 Docker 或者 rkt。

![module_01_cluster](.images/module_01_cluster.svg)

# 创建集群（学习环境）

## Minikube

Minikube 是一种可以让你在本地轻松运行 Kubernetes 的工具。 Minikube 在笔记本电脑上的虚拟机（VM）中运行单节点 Kubernetes 集群， 供那些希望尝试 Kubernetes 或进行日常开发的用户使用。

在 mac 上安装使用

```shell
brew install minikube
```

启动 Minikube 并创建一个集群：

```shell
minikube start
```

在启动了 minikube 后，就可以使用 `kubectl` 来和集群进行交互:

```shell
kubectl cluster-info
```

### Deployment

当运行了 Kubernetes 集群，就可以在上面部署容器化应用程序。为此，需要创建 Kubernetes **Deployment** 配置。Deployment 指挥 Kubernetes 如何创建和更新应用程序的实例。创建 Deployment 后，Kubernetes master 将应用程序实例调度到集群中的各个节点上。

创建应用程序实例后，Kubernetes Deployment 控制器会持续监视这些实例。 如果托管实例的节点关闭或被删除，则 Deployment 控制器会将该实例替换为群集中另一个节点上的实例。 **这提供了一种自我修复机制来解决机器故障维护问题。**

![module_02_first_app](.images/module_02_first_app.svg)

您可以使用 Kubernetes 命令行界面 **Kubectl** 创建和管理 Deployment。Kubectl 使用 Kubernetes API 与集群进行交互。

首先使用 kubectl 查看当前 node `kubectl get nodes`

使用 kubectl 部署一个 app

```shell
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
```

这行命令会自动寻找合适的节点来运行该应用程序，并且自动配置集群以便增加新节点。

可以使用 `kubectl get deployments` 来查看正在运行的应用程序。

由于应用程序是运行在 Kubernetes 内部，虽然内部的应用程序可以互相通信，但是外部网络无法发现，可以使用 `kubectl proxy` 设置使终端可以直接访问 API。

### 查看 Pod 和工作节点

在创建了 Deployment 时，Kubernetes 添加了一个 Pod 来托管应用实例，Pod 是 Kubernetes 抽象出来的表示一组一个或多个应用程序容器（如 Docker），以及一些共享资源

* 共享存储，当做卷
* 网络，作为唯一集群 IP 地址
* 有关每个容器如何运行的信息

Pod是 Kubernetes 平台上的原子单元。 当我们在 Kubernetes 上创建 Deployment 时，该 Deployment 会在其中创建包含容器的 Pod （而不是直接创建容器）。每个 Pod 都与调度它的工作节点绑定，并保持在那里直到终止（根据重启策略）或删除。 如果工作节点发生故障，则会在群集中的其他可用工作节点上调度相同的 Pod。

