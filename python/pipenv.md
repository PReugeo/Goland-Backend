## 概述

 pipenv 是 Python 官方推荐的包管理工具。 它综合了 virtualenv , pip 和 pyenv 三者的功能。能够自动为项目创建和管理虚拟环境。如果你使用过 requests 库，就一定会爱上这个库，因为是同一个大神出品。 pipenv 使用 Pipfile 和 Pipfile.lock 来管理依赖包，并且在使用 pipenv 添加或删除包时，自动维护 Pipfile 文件，同时生成 Pipfile.lock 来锁定安装包的版本和依赖信息，避免构建错误。相比 pip 需要手动维护 requirements.txt 中的安装包和版本，具有很大的进步。

**特性**

* pipenv 集成了 pip，virtualenv 两者的功能，并且完善了两者的一些缺陷
* 管理 requirements.txt 文件可能会有问题，pipenv 使用 pipfile 和 pipfile.lock 
* 各个地方使用了哈希校验
* 通过加载 .env 文件简化开发工作流程

**解决的问题**

1. requirements.txt 依赖管理的局限
2. 多个项目依赖不同第三方库、包版本问题

## 使用 

**安装**

`pip install pipenv --user`  为当前用户安装

mac 上需要加入环境变量：`export PATH="$PATH:$HOME/Library/Python/3.8/bin"`

使用 `pip show pipenv` 来查看包的位置，然后将其加入环境变量

**常用命令**

```shell
pipenv --two  # 使用当前系统中的Python2 创建环境
pipenv --three  # 使用当前系统中的Python3 创建环境

pipenv --python 3  # 指定使用Python3创建环境
pipenv --python 3.6  # 指定使用Python3.6创建环境
pipenv --python 2.7.14  # 指定使用Python2.7.14创建环境
```

1）创建环境时应使用系统中已经安装的、能够在环境变量中搜索到的Python 版本，否则会报错。
2）每次创建环境都会在当前目录下生成一个名为Pipfile文件，用来记录刚创建的环境信息，如果当前目录下之前存在该文件，会将其覆盖。
3）在使用指定版本创建环境的时候，版本号与参数 --python 之间有个空格

**常用命令**

```shell
pipenv shell # 激活虚拟环境
pipenv --where  # 显示目录信息
pipenv --venv  # 显示虚拟环境信息
pipenv --py  # 显示Python解释器信息

pipenv install XXX  # 安装XXX模块并加入到Pipfile
pipenv install XXX==1.11  # 安装固定版本的XXX模块并加入到Pipfile

pipenv graph  # 查看目前安装的库及其依赖
pipenv check  # 检查安全漏洞

pipenv update --outdated  # 查看所有需要更新的依赖项
pipenv update  # 更新所有包的依赖项
pipenv update <包名>  # 更新指定的包的依赖项

pipenv uninstall XXX  # 卸载XXX模块并从Pipfile中移除
pipenv uninstall --all  # 卸载全部包并从Pipfile中移除
pipenv uninstall --all-dev  # 卸载全部开发包并从Pipfile中移除

pipenv --rm  # 删除虚拟环境

```

**兼容 requirements.txt**

```bash
pipenv lock -r > requirements.txt  # 将Pipfile和Pipfile.lock文件里面的包导出为requirements.txt文件
pipenv lock -r --dev > requirements.txt  # 将Pipfile和Pipfile.lock文件里面的开发包导出为requirements.txt文件

pipenv install -r requirements.txt
pipenv install -r --dev requirements.txt  # 只安装开发包
```

