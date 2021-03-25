## 介绍

docopt 它能够根据命令行程序中定义的接口描述，来自动生成解析器。

## 使用

### 安装

`pip install docopt==0.6.2 --user`

### API

```python
from docopt import docopt
docopt(doc, argv=None, help=True, version=None, options_first=False)

"""
doc 参数可以是 __doc__ 或者是其他帮助信息，这个值会解析为参数解析器
argv 是一个可选参数矢量，可以为接口参数提供一个默认值
version --version 打印出来的字符
options_first 如果为 True，将禁止混合选项和位置参数
"""
```

### 解析命令

我们可以编写一个 `cmd.py` 

```python
"""Num accumulator.

Usage:
  cmd.py [--sum] <num>...
  cmd.py (-h | --help)

Options:
  -h --help     Show help.
  --sum         Sum the nums (default: find the max).
"""
from docopt import docopt

arguments = docopt(__doc__, options_first=True)
print(arguments)
```

> doc 注释信息需要放在 docopt 模块导入语句上

```shell
python3 cal.py
Usage:
  cmd.py [--sum] <num>...
  cmd.py (-h | --help)
```

 运行会打印出以上信息。

根据提示信息来使用参数

```shell
python3 cal.py --sum 1 2 3
```

打印出了以下信息

```shell
{'--help': False,
 '--sum': True,
 '<num>': ['1', '2', '3']}
```

可以看到：

- 没有提供 `-h` 或者 `--help`，所以 `arguments` 中 `--help` 为 `False`
- 提供了 `--sum`，所以 `arguments` 中 `--sum` 为 `True`
- 提供了 `<num>...` 为 `1 2 3`，所以 `arguments` 中 `<num>` 为 `['1', '2', '3']`

由此可以推出，docopt 将 `__doc__` 信息，转化为了一个 `dict` 来存储相关的数据信息

### 业务逻辑

在获取了解析的命令参数后，我们就可以根据其写一些业务逻辑，在示例中我们希望获取 `--sum` 后的值来进行相加运算

```python
from docopt import docopt

arguments = docopt(__doc__, options_first=True)

nums = (int(num) for num in arguments['<num>'])
result = 0
if arguments.get('--sum'):
    result = sum(nums)
else:
    result = max(nums)
print(result)
```



## 深入解析

### 使用模式

`docopt` 接口描述的总体规则为：

- 位于关键字 `usage:`（大小写不敏感）和一个可见的空行之间的文本内容会被解释为一个个使用模式。

- `useage:` 后的第一个词会被解释为程序的名称，比如下面就是一个没有命令行参数的示例程序：

    ```tex
    Usage: cli
    ```

### 接口参数

**位置参数**

使用 `<` 和 `>` 包裹的参数为位置参数

**选项参数**

`-0 --option`

以单个破折号开头的参数为短选项，`--` 则为长选项

- 短选项支持集中表达多个短选项，比如 `-abc` 等价于 `-a`、`-b` 和 `-c`
- 长选项后可跟参数，通过 `空格` 或 `=` 指定，比如 `--input ARG` 等价于 `--input=ARG`
- 短选项后可跟参数，通可选的 `空格` 指定，比如 `-f FILE` 等价于 `-fFILE`

**子命令**

在 `docopt` 中，凡是不符合 `--options` 或 `<arguments>` 约定的词，均会被解释为子命令

在下面这个例子中，我们支持 `create` 和 `delete` 两个子命令，用来创建或删除指定路径。而 `delete` 命令支持 `--recursive` 参数来表明是否递归删除指定路径：

```python
"""
Usage:
  cli create
  cli delete [--recursive]

Options:
  -r, --recursive   Recursively remove the directory.
"""
from docopt import docopt

arguments = docopt(__doc__)
print(arguments)
```

直接指定 `delete -r`，输出如下：

```text
$ python3 cli.py delete -r

{'--recursive': True,
 'create': False,
 'delete': True}
```



**可选参数**

以中括号“[]”包裹的元素（选项、参数和命令）均会被标记为可选。多个元素放在一对中括号中或各自放在中括号中是等价的。



**必填参数**

以 `() ` 包裹的为必填参数

**互斥参数**

使用 `|` 分割的为互斥参数

`my_program.py (--clockwise | --counter-clockwise) TIME`

**可变参数**

在参数后面加 `…` 表示为可变参数

**选项简写**

“[options]”用于简写选项，比如下面的示例中定义了 3 个选项：

```text
Usage: my_program [--all --long --human-readable] <path>

--all             List everything.
--long            Long output.
--human-readable  Display in human-readable format.
```

可以简写为：

```text
Usage: my_program [options] <path>

--all             List everything.
--long            Long output.
--human-readable  Display in human-readable format.
```

