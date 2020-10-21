## dock 更丝滑（自动隐藏去除延迟）

defaults write com.apple.Dock autohide-delay -float 0 && killall Dock

## brew 换源

**中科大源**

```bash
# 替换brew.git:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.ustc.edu.cn/brew.git

# 替换homebrew-core.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git

# 替换homebrew-cask.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-cask"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git

# 应用生效
brew update
# 替换homebrew-bottles:
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc
```

**切换回原来的源**

```bash
# 替换brew.git:
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git

# 替换homebrew-core.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core.git

# 替换homebrew-cask.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-cask"
git remote set-url origin https://github.com/Homebrew/homebrew-cask.git

# 应用生效
brew update

# 删除.zshrc变量
vim  ~/.zshrc
# 删除如下变量
# export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles

# 执行更新
source ~/.zshrc
```

安装失败时 ：cd: no such file or directory: /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core

```bash
cd /usr/local/Homebrew/Libraty/Taps
mkdir homebrew
cd homebrew
git clone https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
```

然后进行换源即可。



## AMD 黑苹果安装 docker



AMD 下的黑苹果无法正常使用 Docker ，由于苹果从来没有用过 AMD 的 CPU ，所以 MacOs 下的 Docker 只对 Intel 做了支持，AMD 黑苹果是破解内核实现 所以即使开启了主板虚拟化 也不能正常使用 Docker

Docker 对老版本的 MacOs 系统提供了 Docker Toolbox 简单来说就是需要依赖虚拟机(VirtualBox)来实现 Docker 的功能，但是即使 AMD 黑苹果机器下载下来 还是不能正常使用 这里需要手动创建虚拟机

[Download && Install Docker Toolbox For mac](https://docs.docker.com/toolbox/toolbox_install_mac/)

打开 Terminal 输入如下命令:

```shell
docker-machine create -d virtualbox --virtualbox-no-vtx-check default
```

这里安装的 Docker 版本即 Docker Toolbox 里面的 Docker 版本 ，如果需要安装不同版本的 Docker 输入命令:

```shell
docker-machine create -d virtualbox --virtualbox-no-vtx-check --virtualbox-boot2docker-url https://github.com/boot2docker/boot2docker/releases/download/v17.09.0-ce/boot2docker.iso default
```

值得一说的是这种方式创建的 Docker 网段可能和你本地的网段不一致 比如:

local 192.168.31.106 -> docker 192.168.99.100

可以考虑加入 Host

```shell
192.168.99.100 docker
```

使用前使用

```shell
eval "$(docker-machine env default)"
```

设置 docker 初始运行环境.

