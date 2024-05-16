## GIL 全局解释器锁

Python 多线程，变量都是线程共享的，所以多 cpu 操作多个线程时会引起结果无法预测的问题，也就是 Python 的线程不安全。

CPython 解释器使用加锁的方法来解决线程不安全的问题 GIL。

GIL（Global Interpreter Lock）全局锁就相当于厂房规定：工人要到车间工作，从厂房大门进去后要在里面反锁，完成工作后开锁出门，下一个工人再进门上锁。也就是说，任意时刻厂房里只能有一个工人，这样就保证了工作的安全性，这就是 GIL 的原理。

但是因此多线程效率大打折扣，这是 CPython 解释器的设计特点并非 python 语言特性

## 协程与线程

线程为系统级别，协程为用户级别

即线程为操作系统决定切换，人为无法干预，线程适合不适合相互引用，耦合性为 0 的代码。

线程则为用户级别，由用户决定是否切换，当遇到阻塞 IO 时可以手动切换至另一协程中工作。

## Python 生成器

python 的函数体内部含有 `yield` 关键字时，该函数即为生成器函数，执行结果为 

```shell
<generator object fibonacci at 0x112be2b88>
```

需要取值，可以通过（`for`，`next()`）来进行生成器的迭代，生成器终止时会抛出 `StopIteration` 异常，for 可以捕获该异常，异常的 value 属性值为生成器函数的 return 值。

生成器会在 `yield` 语句处暂停，协程 IO 阻塞就出现在这里。

生成器由迭代器进化而来，所以其含有 `__iter__` 和 `__next__` 方法，可以使用 for 关键字进行取值。在 Python 3.3 中出现 yield from 语法之前，生成器没有太大用途。但此时 yield 关键字还是实现了一些特性，且至关重要，就是生成器对象有 `send` 、`throw` 和 `close` 方法。这三个方法的作用分别是发送数据给生成器并赋值给 yield 语句、向生成器中抛入异常由生成器内部处理、终止生成器。

生成器（或协程）有四种存在状态：

- `GEN_CREATED` 创建完成，等待执行
- `GEN_RUNNING` 解释器正在执行（这个状态在下面的示例程序中无法看到）
- `GEN_SUSPENDED` 在 yield 表达式处暂停
- `GEN_CLOSE` 执行结束，生成器停止

可以使用 `inspect.getgeneratorstate` 获取生成器当前状态

因为程序员可以自己控制生成器的启动、暂停、终止，而且可以向内部传入数据，所以这种生成器也可以作为协程。

> 手动调用 next() 方法，也叫预先激活协程（生成器），将生成器运行至 yield 语句处暂停

预先激活协程，装饰器

```python
def coroutine(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        g = func(*args, **kwargs)
        next(g)
        return g
    return wrapper
```

获取生成器的值

```python
@coroutine
def generator():
    i = '激活生成器'
    l = []
    while True:
        value = yield
        if value == 'STOP':
            break
        l.append(value)
    return l

if __name__ == '__main__':
    g = generator()
    value = 0
    for i in ('hello', 'world', 'STOP'):
        try:
            g.send(i)
        except StopIteration as e:
            value = e.value
    
    print(value)
    ['hello', 'world']
```

`chian` 方法，将任意数量可迭代的对象返回为一个包含所有元素的迭代器。

使用 `yield` 来实现 `chian` 方法

```python
def chain(*args):
  for iter_obj in args:
    for i in iter_obj:
      yield i
```

使用 `yield from` 关键字实现

```python
def chain(*args):
  for iter_obj in args:
    yield from iter_obj
```

`yield from` 后面接收一个可迭代对象，例如上面代码中的 iter_obj 变量，在协程中，可迭代对象往往是协程对象，这样就形成了嵌套协程。

`yield from` 可以转移控制权，为生成器到协程的最重要的一步

`yield from` 会自动预激活子生成器 `iter_obj` 并且捕获子生成器终止时出发的 StopIteration 异常并将异常的 value 值返回给 `yield from` 赋值代码前面的变量也就相当于子式的 `return` 值，父生成器的 `send` 方法将发送值给子生成器。

示例代码

```python
def coroutine(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        g = func(*args, **kwargs)
        next(g)
        return g
    return wrapper

def sub_coro():
    l = []
    while True:
        value = yield
        if value == 'STOP':
            break
        l.append(value)
    return sorted(l)

# 父生成器
@coroutine
def dele_coro():
    while True:
        l = yield from sub_coro()
        print('排序后的列表：', l)
        print('------------------')

def main():
    fake = Faker().country_code
    nest_country_list = [[fake() for i in range(3)] for j in range(3)]
    for country_list in nest_country_list:
        print('国家代号列表', country_list)
        # 创建父生成器
        c = dele_coro()
        for country in country_list:
            c.send(country)
        c.send('STOP')
```



## asyncio

### 协程装饰器

在 Python 3.4 中，`asyncio` 模块出现，此时创建协程函数须使用 `asyncio.coroutine` 装饰器标记。此前的包含 `yield from` 语句的函数既可以称作生成器函数也可以称作协程函数，为了突出协程的重要性，现在使用 `asyncio.coroutine` 装饰器的函数就是真正的协程函数了。

### 概念

**coroutine 协程**

协程对象，使用 asyncio.coroutine 装饰器装饰的函数被称作协程函数，它的调用不会立即执行函数，而是返回一个协程对象，即协程函数的运行结果为协程对象，注意这里说的 “运行结果” 不是 return 值。协程对象需要包装成任务注入到事件循环，由事件循环调用。

**task 任务**

将协程对象作为参数创建任务，任务是对协程对象的进一步封装，其中包含任务的各种状态。

**event_loop 事件循环**

在上一节实验中介绍线程时，将多线程比喻为工厂里的多个车间，那么协程就是一个车间内的多台机器。在线程级程序中，一台机器开始工作，车间内的其它机器不能同时工作，需要等上一台机器停止，但其它车间内的机器可以同时启动，这样就可以显著提高工作效率。在协程程序中，一个车间内的不同机器可以同时运转，启动机器、暂停运转、延时启动、停止机器等操作都可以人为设置。

事件循环能够控制任务运行流程，也就是任务的调用方。

**示例**

```python
import asyncio

def one():
    start = time.time()

    @asyncio.coroutine
    def worker():
        print("Worker started")
        time.sleep(0.1)
        print("Worker stopped in a coroutine")
    
    print(worker())
    loop = asyncio.get_event_loop()
    coroutine = worker()
    task = loop.create_task(coroutine)
    print(isinstance(task, asyncio.Task))
    print(task._state)
    loop.run_until_complete(coroutine)
    print(task._state)
    end = time.time()
    print('运行耗时：{:.4f}'.format(end - start))     
```

协程对象无法直接运行，必须放入事件循环中或者由 `yield from` 语句调用，事件循环则是将协程包装成一个 `ayncio.Task` 对象，任务对象保存了协程运行后的状态，用于未来获取协程结果。

### async/await

在 Python 3.5 中新增了 async / await 关键字用来定义协程函数。这两个关键字是一个组合，其作用等同于 asyncio.coroutine 装饰器和 yield from 语句。此后协程与生成器就彻底泾渭分明了。

协程若是一个 IO 操作，如果需要等待它处理完后，我们能得到通知，这一需求可以通过 `future` 对象中添加回调来实现，而 `asyncio.Task` 为 `asyncio.Future` 的子类，task 对象可以添加回调函数，回调函数的最后一个参数是 `future` 或 `task` 对象，通过该对象可以获取协程返回值，如果回调需要多个参数，可以通过偏函数导入。

示例

```python
def three():
    start = time.time()

    async def corowork():
        print('[corowork] start coroutine')
        time.sleep(0.1)
        print('[corowork] this is a coroutine')

    def callback(name, task):
        print('[callback] Hello {}'.format(name))
        print('[callback] coroutine: {}'.format(task._state))
    
    loop = asyncio.get_event_loop()
    coroutine = corowork()
    task = loop.create_task(coroutine)
    # 使用 functools 来传回调函数的参数
    task.add_done_callback(functools.partial(callback, 'Hello'))
    loop.run_until_complete(task)
    end = time.time()
    print('运行耗时：{:.4f}'.format(end - start))  
```

`await` 关键字相当于 `yield from` 语句后面接协程对象，`asyncio.sleep` 方法的返回值为协程对象，这一步为阻塞运行。

1. `asyncio.sleep` 与 `time.sleep` 是不同的，前者阻塞当前协程，即 `corowork` 函数的运行，而 `time.sleep` 会阻塞整个线程，所以这里必须用前者，阻塞当前协程，CPU 可以在线程内的其它协程中执行
2. 协程的 return 值可以在协程运行结束后保存到对应的 task 对象的 result 方法中
3. 将任务对象作为参数，asyncio.gather 方法创建任务收集器。注意，asyncio.gather 方法中参数的顺序决定了协程的启动顺序
4. `run_until_complete` 方法的返回值就是协程函数的 `return` 值
5. asyncio 模块提供了 `asyncio.gather` 和 `asyncio.wait` 两个任务收集方法，它们的作用相同，都是将协程任务按顺序排定，再将返回值作为参数加入到事件循环中。前者在上文已经用到，后者与前者的区别是它可以获取任务的执行状态（PENING & FINISHED），当有一些特别的需求例如在某些情况下取消任务，可以使用 `asyncio.wait` 方法。

```python
def four():
    start = time.time()

    async def corowork(name, t):
        print('[corowork] start %s coroutine' % name)
        await asyncio.sleep(t)
        print('[corowork] this is %s' % name)
        return 'Coroutine {} OK'.format(name)
    
    loop = asyncio.get_event_loop()
    coroutine1 = corowork('ONE', 3)
    coroutine2 = corowork('TWO', 1)

    task1 = loop.create_task(coroutine1)
    task2 = loop.create_task(coroutine2)

    gather = asyncio.gather(task1, task2)
    loop.run_until_complete(gather)

    print('[task1] ', task1.result()) 
    print('[task2] ', task2.result()) 
    end = time.time()
    print('运行耗时：{:.4f}'.format(end - start))  
```

### 取消任务

`task` 对象的 `cancel` 方法可以取消任务，`asyncio.Task.all_tasks()` 可以获取事件循环中的全部任务，`task.cancel()` 可以取消未完成的任务，取消成功为 True，已经完成的则返回 False

### 排定任务

前文所示的多任务程序中，事件循环里的任务的执行顺序由 asyncio.ensure_future / loop.create_task 和 asyncio.gather 排定，这一节介绍 loop 的其它方法。

`loop.run_forever()` 事件循环的 `run_until_complete` 方法运行事件循环，当其中的全部任务完成后，自动停止事件循环；`run_forever` 方法为无限运行事件循环，需要自定义 `loop.stop` 方法并执行之才会停止

回调调用例子

```python
def callback(loop, future):
    loop.stop()

async def work(t):
    print('start')
    await asyncio.sleep(t)
    print('after {}s stop'.format(t))

loop = asyncio.get_event_loop()
task = asyncio.gather(work(1), work(2))

task.add_done_callback(functools.partial(callback, loop))

loop.run_forever()
loop.close()
```

loop.run_until_complete 方法本身也是调用 loop.run_forever 方法，然后通过回调函数调用 loop.stop 方法实现的

**`loop.call_soon`**

事件循环的 call_soon 方法可以将普通函数作为任务加入到事件循环并立即排定任务的执行顺序

call_soon 将普通函数当作 task 加入到事件循环并排定执行顺序，该方法的第一个参数为普通函数名字，普通函数的参数写在后面

**`loop.call_after`**

此方法同 loop.call_soon 一样，可将普通函数作为任务放到事件循环里，不同之处在于此方法可延时执行，第一个参数为延时时间。

**`loop.call_at` 和 `loop.time`**

- call_soon 立刻执行，call_later 延时执行，call_at 在某时刻执行
- loop.time 就是事件循环内部的一个计时方法，返回值是时刻，数据类型是 float

### 协程锁

asyncio.lock 应该叫做异步 IO 锁，之所以叫协程锁，是因为它通常使用在子协程中，其作用是将协程内部的一段代码锁住，直到这段代码运行完毕解锁。协程锁的固定用法是使用 async with 创建协程锁的上下文环境，将代码块写入其中。

```python
import asyncio

l = []
lock = asyncio.Lock()   # 协程锁

async def work(name):
    print('lalalalalalalala')     # 打印此信息是为了测试协程锁的控制范围
    # 这里加个锁，第一次调用该协程，运行到这个语句块，上锁
    # 当语句块结束后解锁，开锁前该语句块不可被运行第二次
    # 如果上锁后有其它任务调用了这个协程函数，运行到这步会被阻塞，直至解锁
    # with 是普通上下文管理器关键字，async with 是异步上下文管理器关键字
    # 能够使用 with 关键字的对象须有 __enter__ 和 __exit__ 方法
    # 能够使用 async with 关键字的对象须有 __aenter__ 和 __aexit__ 方法
    # async with 会自动运行 lock 的 __aenter__ 方法，该方法会调用 acquire 方法上锁
    # 在语句块结束时自动运行 __aexit__ 方法，该方法会调用 release 方法解锁
    # 这和 with 一样，都是简化 try ... finally 语句
    async with lock:
        print('{} start'.format(name))  # 头一次运行该协程时打印
        if 'x' in l:                    # 如果判断成功
            return name                 # 直接返回结束协程，不再向下执行
        await asyncio.sleep(0); print('----------')  # 阻塞 0 秒，切换协程
        l.append('x')
        print('{} end'.format(name))
        return name

async def one():
    name = await work('one')
    print('{} ok'.format(name))

async def two():
    name = await work('two')
    print('{} ok'.format(name))

def main():
    loop = asyncio.get_event_loop()
    tasks = asyncio.wait([one(), two()])
    loop.run_until_complete(tasks)

if __name__ == '__main__':
    main()
```

