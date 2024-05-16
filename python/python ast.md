## AST 模块

Abstract Syntax Trees 抽象语法树，为 python 源码到字节码的中间产物，借助 ast 模块可以从语法书的角度分析源码结构，可以将 source 生成的语法树 unparse 成 python 源码，使用 ast 可以给 python 源码检查、语法分析、修改代码以及代码调试。

python 代码处理过程如下：

> 源代码解析 --> 语法树 --> 抽象语法树(AST) --> 控制流程图 --> 字节码

## 使用

获取抽象语法树（AST）

```python
import ast

code = compile('import os; print(os.cpu_count())\n', "<>", "exec", flags=ast.PyCF_ONLY_AST)
code2 = ast.parse('import os; print(os.cpu_count())\n')
print(ast.dump(code))
print(ast.dump(code2))
```

