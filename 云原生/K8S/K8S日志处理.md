## 容器日志采集与管理

### 日志采集场景

**日志采集场景主要分为以下四种：** 

**（1）集群核心组件日志**
    审计需要 kube-apiserver 日志，诊断调度需要 kube-scheduler 日志，接入层流量分析需要 Ingress 日志。

**（2）主机内核日志**

​    内核日志可以用于帮助开发及运维同学诊断影响节点稳定的异常，如：文件系统异常，网络栈异常，设备驱动异常等。

**（3）用运行时日志**

​    Docker 是最常见的容器运行时，可以利用 Docker 和 Kubelet 日志排查 Pod 创建和启动失败等问题。

**（4）业务应用日志**

​    通过分析业务的运行日志分析和观察业务状态，诊断异常。

### **日志采集指标** 

​    Kubernetes 对容器日志的期望处理方式为：集群级日志处理（cluster-level-logging）

​    即：与容器、Pod、节点生命周期完全无关。

​    对于一个容器，当应用将日志输出到 stdout 和 stderr 后，docker 默认将这些日志输出到宿主机上一个 JSON 文件中。

### 日志采集方式

 **日志采集方式1：使用节点级日志代理**

​    核心是 logging-agent （fluentd，etc ）；

​    Logging-agent 以 DaemonSet 方式运行在节点上；

​    挂载宿主机上的容器日志目录；

​    转发日志至后端存储（ElasticSearch, etc）;

​    **优点：**对应用和 Pod 完全无侵入，一个节点仅需部署一个 agent。

​    **缺点：**要求应用日志直接输出至容器的 stdout 和 stderr。 

**日志采集方式2：使用 sidecar 容器和日志代理**

​    容器全部或部分日志输出到文件

​    一个或多个 sidecar 容器将应用程序日志传送到自己的 stdout 和 stderr。

​    **优点：**能够继续使用日志采集方式1。

​    **缺点：**成倍增加磁盘占用，造成浪费。（应用和sidecar容器写入两份相同日志文件）

**日志采集方式3：使用具有日志代理功能的 sidecar 容器**

​    相当于将 logging-agent 直接集成进 Pod。

​    应用和输出日志至 stdout&stderr 或文件。

​    Logging-agent 的输入源为应用日志文件。

​    **优点：**部署简单，对宿主机友好。

​    **缺点：①**Sidecar 容器可能消耗较多资源，甚至拖挂应用容器。

​         	 ②无法使用 kubectl logs 命令查看容器日志。

**总结：**

**实现集群级日志采集的三种方式：**

（1）使用节点级日志代理。

（2）使用 sidecar 容器和日志代理。

（3）使用具有日志代理功能的 sidecar 容器。

**建议**：

使用方案1，将应用日志输出到 stdout & stderr，通过宿主机上直接部署 logging-agent 的方式集中处理日志。

（1）管理简单。

（2）可以使用 kubectl logs 命令查看日志。

（3）宿主机本身可能已有 rstlogd 等成熟日志收集组件可使用。

## 容器监控指标的采集与管理

### 监控场景

​	**资源监控**

​	CPU，内存，网络等资源类监控

​    **性能监控：**

​    应用的内部监控。通常是 Hock 机制在虚拟机层，字节码执行层隐式回调，或者在应用层显式注入，获取更深层次的监控指标，常用来应用诊断与调优。

​    比如 Jvm 通过 Hock 机制,拿到类似 Jvm 里面的垃圾回收的次数，各种内存带的分布以及网络连接数的一些指标。通过这样的方式来进行应用的诊断与调优。

​    **安全监控：**

​    针对安全进行一系列监控策略，例如越权管理，安全漏洞扫描等。

​    **事件监控：**

​    Kubernetes 中特有的监控方式，贴合 Kubernetes 设计理念，作为常规监控方案的补充。

​    为什么说事件监控贴合 Kubernetes 设计理念呢？这是因为 Kubernetes 其中一个设计理念就是基于状态机的状态转换。从正常状态转换成另一个状态的时候，会发生一个 Normal 级别的事件(也就是正常的事件)而从一个正常状态转换成异常状态时，平台会触发一个 Warning 级别（也就是警告级别的事件）通常，Warning 级别的事件是我们关心的事件。

​    而事件监控就可以把 Normal 级别的事件或 Warning 级别的事件离线存储到数据中心，然后通过数据中心的分析与报警，将相应的异常通过短信，邮件的方式暴露，弥补常规监控的弊端。

### Prometheus

**架构**

![1489604-20190515184649649-873258872](/Users/yanjigang01/Golang-Backend/云原生/image/1489604-20190515184649649-873258872.png)

prometheus 根据配置定时去拉取各个节点的数据，默认使用的拉取方式是 pull，也可以使用 pushgateway 提供的 push 方式获取各个监控节点的数据。将获取到的数据存入 TSDB，一款时序型数据库。此时 prometheus 已经获取到了监控数据，可以使用内置的 PromQL 进行查询。它的报警功能使用 Alertmanager 提供，Alertmanager 是 prometheus 的告警管理和发送报警的一个组件。prometheus 原生的图标功能过于简单，可将 prometheus 数据接入 grafana，由 grafana 进行统一管理。

**Prometheus 指标来源**

（1）宿主机的监控数据：需要借助 Node Exporter 向外暴露; Exporter 代替被监控对象来向 Prometheus 暴露可以被抓取的指标信息。

（2）Kubernetes 组件如 APIServer, kubelet 等的/metrics AP:除CPU，内存外，还包括各个组件的核心监控指标。

（3）Kubernetes 核心的监控数据：包括Pod、Node、容器、Service等主要核心概念的 metrics，其中容器相关的指标来源于 kubectl 内置的 cAdvisor 服务。

**Prometheus 特点**

（1）简洁强大的接入标准。只要实现 Promethus Client 接口，就可以直接实现数据的采集。

（2）多种数据采集方式。包括：在线，离线，push, pull 联邦的方式进行数据采集。

（3）和 Kubernetes完全兼容。

（4）丰富的插件机制和生态。

（5）Prometheus Operator 助力。使 Prometheus 的运维实现自动化。

