## \_\_name__ 方法

python 中由于函数也是一个对象，而且函数对象可以被赋值给变量，所以，通过变量也能调用该函数。

函数对象有一个`__name__`属性，可以拿到函数的名字

## 闭包

闭包为一种特殊的函数，分为内函数和外函数

例如

```python
# outer是外部函数
def outer(a):
    # inner是内函数
    def inner( b ):
        #在内函数中 用到了外函数的临时变量
        print(a+b)
    # 外函数的返回值是内函数的引用
    return inner
ret = outer(5) #ret = inner
ret(10) #15 ret 存了外函数的返回值，也就是inner函数的引用，这里相当于执行inner函数
```

## 装饰器

装饰器是 python 的一个语法糖，也可以称为一种特殊的闭包，装饰器一般是在类、函数上使用 `@` 符号定义

例子一个嵌套函数

```python
def decora(func):
    def wrapper():
        print("hello world")
        func()
    return wrapper
  
def greet():
    print("rua rua rua")
 
greeter = decora(greet)
greeter()
"""
output: 
hello world
rua rua rua
"""
```

使用装饰器

```python
def decora(func):
    def wrapper():
        print("hello world")
        func()
    return wrapper

@decora
def greet():
    print("rua rua rua")
 
greet()
"""
output;
hello world
rua rua rua
"""
```

可见使用装饰器 `@` 相当于执行了以下语句 `greet=decora(greet); greet()`



## 带参数嵌套函数的装饰器

如果 decorator 函数本身需要传入参数，那就需要写一个返回 decorator 的高阶函数，内部函数需要使用 `*args, **kwargs`，去接受

`*args`接收元组， `**kwargs` 接收字典

例如:

```python
def repeat(num):
    def my_decorator(func):
        def wrapper(*args, **kwargs):
            for i in range(num):
                print('wrapper of decorator')
                func(*args, **kwargs)
        return wrapper
    return my_decorator


@repeat(2)
def send_msg():
    print('hello world')

send_msg()
print(send_msg.__name__)
"""
wrapper of decorator
hello world
wrapper of decorator
hello world
wrapper
"""
```

## \_\_name__ 问题

使用了装饰器后的函数，其 `__name__` 方法会变成 `wrapper`,所以需要将原始函数的 name 复制到 wrapper 函数中，否则有些依赖函数签名的代码执行就会出错。

在 python 中内置了 `functools.wraps` 可以防止编写像 `wrapper.__name__=func.__name__` 这种代码。

综上所属，一个完整的 decorator 的写法为：

````python
import functools
def decora(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print("hello world")
        func(*args, **kwargs)
    return wrapper

@decora
def greet():
    print("rua rua rua")
 
# greeter = greet()
# greeter()
greet()
print(greet.__name__)

"""
hello world
rua rua rua
greet
"""
````



带参数的装饰器为

```python
import functools
def repeat(num):
    def my_decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(num):
                print('wrapper of decorator')
                func(*args, **kwargs)
        return wrapper
    return my_decorator


@repeat(2)
def send_msg():
    print('hello world')

send_msg()
print(send_msg.__name__)
```

