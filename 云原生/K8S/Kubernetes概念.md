## 概念

Kubernetes 是一个可移植、可扩展的开源平台，主要用于管理容器化的工作负载和服务，方便声明式配置和自动化。

Kubernetes由一组成为节点（Node）的机器组成。

 主要作用管理运行程序的容器，主要功能为

* 服务发现和负载均衡
* 存储编排
* 自动部署和回滚
* 自动完成装箱计算
* 自我修复
* 密钥与配置管理

## 运行

### kubectl

kubectl 为 k8s 的命令行工具，可以用kubectl部署应用、检测和管理集群相关资源以及查看相关日志。

**安装**

- [在 Linux 上安装 kubectl](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl-linux)
- [在 macOS 上安装 kubectl](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl-macos)
- [在 Windows 上安装 kubectl](https://kubernetes.io/zh/docs/tasks/tools/install-kubectl-windows)

### minikube

minikube 采用的是生成一个单节点 K8S 集群的 VM，非常适合初学者操作。

`minikube start` 即可开启一个 kubernetes 单机集群，还可以通过 `--kubernetes-version` 指定 K8S 版本。minikube 还拥有仪表盘功能，可以让你直观的观察到你的节点的情况。

### Kind

Kind 则是将集群移动到 Docker 容器中，它将显著加快启动速度。

`kind create cluster` 命令即可启动一个 K8S 集群，通过指定标签 `--name` 可以创建几个不同的实例。

Kind 还可以导入自己的 docker 镜像到 cluster 中，通过 `kind load docker-image my-app:latest` 即可导入自己的 docker 镜像。

Kind 同样提供编程接口来创建 K8S 的集群，Kind 提供 Go 相关的编程接口。

### kubeadm

kubeadm 为一个开源工具，可以快速搭建 kubernetes 集群，可以用于搭建生产环境

### 二进制包

官网下载的二进制包，为生产环境的主流搭建方式。

## Kubectl 使用

### 生成自己的镜像

```python
import socketserver
from http import server
from http import HTTPStatus

class Handler(server.SimpleHTTPRequestHandler):
    def do_GET(self) -> None:
        self.send_response(HTTPStatus.OK)
        self.end_headers()
        self.wfile.write(b'hello world')

httpd = socketserver.TCPServer(('', 8000), Handler)
httpd.serve_forever()
```

Dockerfile 文件

```dockerfile
FROM python:3.9.6
EXPOSE 8000
COPY hello.py /home/hello.py
CMD [ "python", "/home/hello.py" ]
```

首先需要进入 `minikube` 容器中使用 docker （如果使用的是 docker 作为 VM）

`eval $(minikube docker-env)`

然后创建镜像 `docker build -t hello-node:1.0 .` 

### 创建 deploy

`kubectl create deployment hello-node --image=myapp:1.0`

* 创建型

`kubectl apply -f xxx.yaml`

* 声明型（修改 yaml 文件后可以直接再次 apply 进行更新，create 则需要删除重建）

### 查看相关状态

`kubectl get pods`

`kubectl get deployments`

查看集群信息 `kubectl get events`

查看 service `kubectl get services`

### 创建 Service

如果想要容器从 kubernetes 虚拟网络外部访问，必须通过 Service 暴露 Pod。

`kubectl expose deployment hello-node --type=LoadBalancer`

**Expose type**

1. ClusterIP（默认）：从集群内部公开 Service，此时 Service 只能从内部访问
2. NodePort：使用 NAT 在集群中每个选定 Node 的相同端口上公开 Service。`<NodeIP>:<NodePort>`从集群外部访问
3. LoadBalancer：创建一个外部负载均衡器（若支持），并为 Service 分配一个固定的外部 IP 为 NodePort 超集。
4. ExternalName：通过返回带有该名称的 CNAME 记录，使用任意名称(由 spec 中的`externalName`指定)公开 Service。不使用代理。

 使用 `minikube service hello-node` 来暴露整体服务，即可打开浏览器进行访问。

![image-20210709190933271](/Users/yanjigang01/Documents/笔记/image/image-20210709190933271.png)

也可通过 `kubectl kubectl port-forward service/hello-node 7080:8000` 暴露服务，此时可通过 `localhost:7080` 访问。

![image-20210709191313281](/Users/yanjigang01/Documents/笔记/image/image-20210709191313281.png)



### 更新镜像

在新编辑完 `hello.py` 后，使用 docker 重新打镜像 `docker build -t hello-node:2.0 .`

然后通过 Deployment 更新

`kubectl set image deployment/hello-node hello-node=hello-node:2.0`

### 删除 deploy

现在可以删除在群集中创建的资源：

```
kubectl delete service hello-node
kubectl delete deployment hello-node
```

或者停止Minikube：

```
minikube stop
```

### Debug

通过 `kubectl get pods --all-namespaces` 查看所有 pods 的状态

![image-20210709154748011](/Users/yanjigang01/Documents/笔记/image/image-20210709154748011.png)

然后通过 `kubectl describe pod hello-minikube-6ddfcc9757-rw6qv --namespace=default` 查看相应 pod 出错的详细信息。

如果还无法查看到详细出错信息则使用 `kubectl logs pod [podname]` 来进行日志排查。

## 参考

[使用Minikube在Kubernetes中运行应用 _ Kubernetes(K8S)中文文档_Kubernetes中文社区](http://docs.kubernetes.org.cn/126.html)

[使用 Service 暴露您的应用 | Kubernetes](https://kubernetes.io/zh/docs/tutorials/kubernetes-basics/expose/expose-intro/)

[minikube start | minikube (k8s.io)](https://minikube.sigs.k8s.io/docs/start/)

