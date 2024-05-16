 在 Kubernetes 系统中，*Kubernetes 对象* 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

- 哪些容器化应用在运行（以及在哪些节点上）
- 可以被应用使用的资源
- 关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 “目标性记录” —— 一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。 通过创建对象，本质上是在告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的， 这就是 Kubernetes 集群的 **期望状态（Desired State）**。

操作 Kubernetes 对象 —— 无论是创建、修改，或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api)。 比如，当使用 `kubectl` 命令行接口时，CLI 会执行必要的 Kubernetes API 调用， 也可以在程序中使用 [客户端库](https://kubernetes.io/zh/docs/reference/using-api/client-libraries/)直接调用 Kubernetes API。

## 描述对象

创建 K8S 对象时，必须提供对象的规约，用来描述该对象的期望状态

，以及对象的一些基本信息。当 K8S API 创建对象时，API 必须带有以上的 JSON 信息，一般情况下是在 `.yaml` 文件中进行定义，`kubectl` 发起 API 请求时，将这些信息转换成 JSON 格式。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

通过 `kubectl apply -f xxx.yaml —record` 命令创建 Kubernetes。

**必须字段**

* `apiVersion`：创建对象所使用的 K8S API 版本
* `kind` 创建对象的类别
* `metadata` 帮助唯一标识对象的一些数据，包括一个 `name` 字符串、UID 和可选的 `namespace`
* `spec` 字段，相关模板：[Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#deployment-v1-apps)



## Pods

Pod 的共享上下文包括一组 Linux 名字空间、控制组（cgroup）和可能一些其他的隔离 方面，即用来隔离 Docker 容器的技术。 在 Pod 的上下文中，每个独立的应用可能会进一步实施隔离。

就 Docker 概念的术语而言，Pod 类似于共享名字空间和文件系统卷的一组 Docker 容器。

通常你不需要直接创建 Pod，甚至单实例 Pod。 相反，你会使用诸如 [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/) 或 [Job](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/) 这类工作负载资源 来创建 Pod。如果 Pod 需要跟踪状态， 可以考虑 [StatefulSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/) 资源。

K8S 中使用 pod 主要有两种方法：

1. 运行单个容器的 Pod，这种情况下，Pod 可以看作单个容器的包装器，K8S 直接管理 Pod
2. 运行多个协同工作的容器的 Pod，Pod 可能封装由多个紧密耦合且需要共享资源的共处容器组成的应用程序，Pod 将这些容器和存储资源打包成一个可管理的实体

### Pod 如何管理容器

Pod 中容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起调度。容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身。

Pod 天生为其成员容器提供了两种共享资源：[网络](https://kubernetes.io/zh/docs/concepts/workloads/pods/#pod-networking)和 [存储](https://kubernetes.io/zh/docs/concepts/workloads/pods/#pod-storage)。

Pod 被设计为相对临时性的、用后即抛的一次性实体，Pod 当由你或控制器创建时，它被调度在集群中的节点运行，Pod 名称需要是合法的 DNS 子域名。

### Pod 控制器

Pod 控制器可以控制副本管理、上线，提供 Pod 失效的容错率（自我恢复），它们会创建替身 Pod 调度到一个健康的节点运行。

- [Deployment](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)：管理应用副本的 api 对象
- [StatefulSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/)： 管理某 Pod 集合的部署和扩缩，并为 Pod 提供持久存储和持久标识符
- [DaemonSet](https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/)： 确保 Pod 副本在集群节点上稳定运行

### Pod 模板

Pod 通过 yaml 文件创建通常使用 `pod template` 来创建

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # 这里是 Pod 模版
    spec:
      containers:
      - name: hello
        image: busybox
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # 以上为 Pod 模版
```

修改 pod 模板只会对新 pod 起作用（已经创建的不会起作用），所以有些控制器会直接将原有 pod 删除再创建新的以应对模板更新，每个控制器会有自己更新 pod 的规则。

### 网络和存储

一个 Pod 可以设置一组共享的卷，Pod 中所有容器都可以访问该共享卷，从而允许这些容器共享数据。卷还允许 pod 中的持久数据保留下来。

每个 pod 都会在有一个唯一的 ip 地址，pod 中每个容器共享网络名字空间，包括 ip 地址和网络端口，容器之间可以通过 `localhost` 互相通信
