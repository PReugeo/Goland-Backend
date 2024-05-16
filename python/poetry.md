## 介绍

Python 打包和依赖管理器，比 pip 强大的多，还包含了以下功能：

* 虚拟环境管理
* 套件依赖性管理
* 套件打包发布

**套件依赖性管理**

管理环境中安装的所有套件及其版本（package，dependency）

可以用于管理套件依赖之间的版本冲突，类似于 `pipenv`的 `pipfile.lock`

## 为何不使用 pip

从 `pip uninstall` 说起，在使用 `pip install` 时，pip 会帮助安装包的其他依赖包，但是在执行 `pip uninstall` 时，pip 只会移除主套件，而其依赖套件则不会被删除，只能去手动删除，但是手动删除会存在一个其他问题，当其他套件也依赖手动删除的组件时，便会出现问题从而无法使用。手动删除时的问题

1. 无法确定手动删除的套件是否被其他依赖
2. 无法确定手动删除套件有多少其他依赖套件

所以 pip 只适合只新增不删除的方案

## Pipenv vs Poetry

为什么不使用 `pipenv` 进行依赖管理

1. pipenv 本身存在许多问题

Pipenv Lock 耗时长、windows 支持差、对 Pypi 套件打包支持差、无法随时更新锁定的依赖

2. pyproject.toml 的出现

`pyproject.toml`时  [PEP 518](https://peps.python.org/pep-0518/) 提出的新标准，将该文件作为套件打包的标准格式，后续又有了  [PEP 621](https://peps.python.org/pep-0621/) ，将其设定为设定所有 python 项目的工具配置，将其取代 `setup.cfg` 文件或者其他工具配置文件。

`poetry`诞生于 `pyproject.toml` ，`pyproject.toml`相当于 `pipfile` 对于 `pipenv` 或者 `package.json`对于 `npm`

## 使用

`poetry`依赖 `python3.7+` 官方推荐直接安装而不是使用 `pip install` 因为其依赖组件较多，可能会污染干净的 pip 环境

### 安装

```shell
# linux or mac
curl -sSL https://install.python-poetry.org | python3 -
# windows powershell
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | py -
```

然后根据提示将 bin 目录加入到 `PATH` 中即可

### 初始化项目

像 git 一样，`poetry` 需要一个初始化过程来创建 `pyproject.toml`来管理依赖，在项目根目录执行

```shell
poetry init
```

然后根据提示就可以完成初始化了

### 初始化虚拟环境

`poetry` 和 `pipenv` 一样自带了 `venv` 功能，使用 `poetry env use python` 可以自动创建一个虚拟环境

[Managing environments | Documentation | Poetry - Python dependency management and packaging made easy (python-poetry.org)](https://python-poetry.org/docs/managing-environments/)

和 `pipenv` 一样，会自动创建一个 venv 环境，可以通过 `poetry config` 修改创建虚拟环境的配置

进入 shell 时也和 `pipenv` 一样

```shell
$ poetry shell
```

### 常用指令

算來算去，Poetry 的常用指令主要有下面幾個：

- `poetry add` 类似 `pip install`
- `poetry remove` 类似 `pip uninstall`
- `poetry export` 输出 requirements.txt 文件
    - `poetry export -f requirements.txt -o requirements.txt `
- `poetry env use` 创建虚拟环境
- `poetry shell` 进入虚拟环境 shell
- `poetry show` 显示依赖项
- `poetry init` 初始化环境

* `poetry update` 更新套件

- `poetry install` 直接安装原有环境 `poetry.lock`记录的套件版本

[再見了 pip！最佳 Python 套件管理器——Poetry 完全入門指南 - Code and Me (kyomind.tw)](https://blog.kyomind.tw/python-poetry/)