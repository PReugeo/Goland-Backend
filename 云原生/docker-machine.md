## 概述

Docker Machine 是一种可以在虚拟主机上安装 Docker Engine 的工具，并使用docker-machine命令管理这个宿主机，可以使用Docker Machine在本地的 Mac 或者 Windows 、公司网络，数据中心或者AWS这样的云提供商上创建docker。

使用 docker-machine 命令，可以启动、审查、停止、重启托管的 docker 也可以升级 Docker 客户端和守护程序并配置 docker 客户端和宿主机通信。

Docker Machine 也可以集中管理所以得docker主机。

### 为什么要使用 Docker Machine

使用 Docker Machine 可以在各种 Linux 上配置多个 Docker 主机，其有两个主要用途

1. 在旧版 Windows 或者 Mac 上运行 Docker

![machine-mac-win](.images/machine-mac-win.png)

2. 在远程管理配置 Docker 

Docker Engine Linux 系统上原生地运行。如果你有一个 Linux 作为你的主系统，并且想要运行 docker 命令，所有你需要做的就是下载并安装 Docker Engine 。然而，如果你想要在网络上、云中甚至本地配置多个 Docker 宿主机只有一个有效的方式，你需要 Docker Machine。

无论你的主系统是 Mac、Windows 还是 Linux，你都可以在其上安装 Docker Machine，并使用 docker-machine 命令来配置和管理大量的 Docker 宿主机。它会自动创建宿主机、在其上安装 Docker Engine 、然后配置 docker 客户端。每个被管理的宿主机（“machine”）是 Docker 宿主机和配置好的客户端的结合。

### Docker Machine 和 Docker Engine 区别

Docker Engine 就是通常的 Docker，是一个客户端-服务器应用程序，由 Docker 守护进程，一个与守护进程交互的 REST API 和一个命令行接口（CLI）。Docker Engine 从 CLI 中接受docker 命令，例如 docker run <image>、docker ps 来列出正在运行的容器、docker images 来列出镜像，等等。

![engine](.images/engine.png)

Docker Machine 是一个用于配置和管理你的宿主机（上面具有 Docker Engine 的主机）的工具。通常，你在你的本地系统上安装 Docker Machine。Docker Machine有自己的命令行客户端 docker-machine 和 Docker Engine 客户端 docker。你可以使用 Machine 在一个或多个虚拟系统上安装 Docker Engine。

这些虚拟系统可以是本地的（就像你在 Mac 或 Windows 上使用 Machine 在 VirtualBox 中安装和运行 Docker Engine 一样）或远程的（就像你使用 Machine 在云提供商上 provision Dockerized 宿主机一样）。Dockerized 宿主机本身可以认为是，且有时就称为，被管理的“machines”。

![machine](.images/machine.png)

## 安装 Docker Machine

可以安装  [Docker Toolbox](https://www.docker.com/docker-toolbox)  来安装 Windows 或者 Mac 上的 Docker Machine，也可以通过命令行安装

OS X

```shell
curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/usr/local/bin/docker-machine && \
  chmod +x /usr/local/bin/docker-machine
```

Linux

```shell
curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
    chmod +x /tmp/docker-machine &&
    sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```

Windows git bash

```shell
if [[ ! -d "$HOME/bin" ]]; then mkdir -p "$HOME/bin"; fi && \
curl -L https://github.com/docker/machine/releases/download/v0.16.2/docker-machine-Windows-x86_64.exe > "$HOME/bin/docker-machine.exe" && \
chmod +x "$HOME/bin/docker-machine.exe"
```

卸载

```shell
# 删除单个 machine
docker-machine rm <machine-name>
# 删除所有 machine 在 windows 上需要加入 -force
docker-machine rm -f $(docker-machine ls -q)
# 删除 docker-machine 可执行文件
rm $(which docekr-machine)
```

## 启动应用 Docker Machine

首先需要安装 virtual box，进入官网下载即可。

通过 Docker Machine 创建 docker 

```shell
docker-machine create -d virtualbox --engine-registry-mirror=https://xxxx.mirror.aliyuncs.com  --virtualbox-no-vtx-check default
```

最后一行为忽略 CPU 虚拟化检查（用于 AMD 黑苹果或者没有虚拟化能力的 CPU）`--engine-registry-mirror` 指定 docker 的代理，一般可以选用阿里云。

启动 Docker Machine

```shell
docker-machine start default
```

配置 machine 环境（在执行完 start 后会提醒）

```shell
docker-machine env
eval $(docker-machine env default)
```

其中 `default` 为创建的 docker-machine

接下来就可以使用 Docker Engine 的命令来进行操控了。

> Docker Machine 创建的 Docker Engine IP 是不一致的需要执行 `docker-machine ip` 获取

**VScode**

在确保 code 在环境变量时，可以执行

```shell
eval $(docker-machine env default)
code
```

Vscode，即可启动在 Docker Engine 环境下，可正常使用 Docker 插件