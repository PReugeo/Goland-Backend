## 继承重写

重写 Kernel 可以继承 `from ipykernel.kernelbase import Kernel`

自己实现 `do_excute()` 方法即可执行代码

也可以直接继承 `from ipykernel.ipkernel import IPythonKernel`

继承 ipykernel 相关能力，在基础上进行修改



```python
import sys
import asyncio

from ipykernel.ipkernel import IPythonKernel
from ipykernel.kernelapp import IPKernelApp

try:
    from IPython.core.interactiveshell import _asyncio_runner
except ImportError:
    _asyncio_runner = None


class Kernel(IPythonKernel):
    def __init__(self, **kwargs):
        super(Kernel, self).__init__(**kwargs)

    async def do_execute(self, code, silent, store_history=True,
                         user_expressions=None, allow_stdin=False):
				ret = await super(kernel, self).do_excute(code, silent, store_history=True,
                         user_expressions=None, allow_stdin=False)
        print(ret)
        return ret

class KernelApp(IPKernelApp):
    
    kernel_class = Kernel

main = KernelApp.launch_instance

```

`ret` 为一个 json，标识代码执行次数顺序以及执行结果是否成功，并不存储

执行代码能力为 `zmqshell.ZMQInteractiveShell` 执行结果 publish

实际执行代码的为 `iPython.core.interactiveshell.run_shell`

`raw_cell` 为输出传输的所有的代码集合

`raw_cell` 转换为 `transformed_code` 提供给 shell 执行

最终是调用 `exec` 或者 `eval`. 方法来执行代码

![image-20211011142551910](/Users/yanjigang01/Golang-Backend/python/源码阅读/ipython/image/image-20211011142551910.png)

![image-20211011143045614](/Users/yanjigang01/Golang-Backend/python/源码阅读/ipython/image/image-20211011143045614.png)

