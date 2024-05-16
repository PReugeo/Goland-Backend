## 开源项目介绍

轻量级、多租户可扩展且安全的网关，使 jupyternotebook 可以在分布式集群内共享资源（K8s, Spark），可远程启动 notebook 的 kernel

功能：

1. 优化以及分布式资源分配
2. 安全性：提供端对端的安全通信（encrypted HTTP）
3. 通过用户模拟支持多租户（kerberos）

## 源码阅读

### 代码结构

![image-20210806162042163](/Users/yanjigang01/Golang-Backend/python/源码阅读/image/image-20210806162042163.png)

`docs` 为文档文件夹 `enterprise_gateway` 为代码主要逻辑文件夹，为主要阅读的代码文件夹，`etc` 则包含了 `docker` `kubernetes` 相关的部署代码

整体使用 `tornado` 异步网络框架进行网络通信

### base

**模块基类 `mixing.py`**

主要定义了一些混合基类，供 `base/handler` 使用，如 `CORS` 校验，`TokenAuth` 鉴权校验， `JSON` 错误返回 `response` 的定义

 `enterprise_gateway` 配置初始化

1. 服务的 IP，Port，Url 等的定义以及环境变量的读取
2. token，cors 请求头的初始化
3. cert 证书配置的初始化
4. ssl 配置以及 kernel 相关的配置等

`kernel` 相关环境变量配置用于存储 `kernel` 状态等相关信息便于 api 返回相关信息



**base/handler.py**

该模块主要对 `jupyter_server` 的 `base.APIHandler` 进行扩展以使用 `tornado handler` 定义，

使用 `mixing` 模块的 `token`， `cors` 以及错误定义类的继承，扩展了 `APIHandler` 添加了 `cors` 跨域支持以及 `token` 鉴权功能

```python
class APIVersionHandler(TokenAuthorizationMixin,
                        CORSMixin,
                        JSONErrorsMixin,
                        APIHandler):
    """"
    扩展 jupyter_server base API handelr 添加 token 校验，cors 跨域支持，
    json 错误请求发送功能等生产环境信息功能
    """
```

以及空返回处理器的定义 `class NotFoundHandler(JSONErrorMixin, web.RequestHandler)`

并将基类注册到 url 中，分别经手 `/api` 以及 `/*` url 的请求

### client

client 中定义了 `gateway_client.py` 模块，该模块主要作用为**实验测试 enterprise gateway 功能**也可以作为微服务类型的连接

作用即为通过 `http` 请求以及 `ws` 请求 CRUD kernel 的相关代码

### itests

测试基础功能文件夹，如 `base` 中的相关鉴权功能，以及 `client` 在不同集群上以及使用不同 `kernel` 的功能测试

### service

service 为 gateway 主要功能的代码文件夹

#### api

api 文件夹定义了，返回静态文件的相关功能，主要返回 `swagger` api 的静态页面

#### kernel

**handler 模块**

主要为处理 `kernel` crud 以及通信的相关 api 代码。

主要重写了创建和获取 `kernel` 的过程方法（`jupyter_server_handlers.MainKernelHandler`）以及加入相关鉴权模块

其中重要环境变量如下（定义于 `mixins`）：

`env_whitelist(EG_ENV_WHITELIST)`：当客户端请求一个新 kernel 时，从 kernel 内核创建时便设置的环境变量由客户端指定，* 时则继承 `request` 中的所有环境变量

`env_process_whitelist`：kernel 内核生成时便创建的环境变量

具体操作代码

```python
 model = self.get_json_body()
    if model is not None and 'env' in model:
        env = {'PATH': os.getenv('PATH', '')}
        # Whitelist environment variables from current process environment
        env.update({key: value for key, value in os.environ.items()
                        if key in self.env_process_whitelist})
        env_whitelist = self.env_whitelist
        if env_whitelist == ['*']:
            env_whitelist = model['env'].keys()
        env.update({key: value for key, value in model['env'].items()
                       if key.startswith('KERNEL_') or key in env_whitelist})
        # No way to override the call to start_kernel on the kernel manager
        # so do a temporary partial (ugh)
        orig_start = self.kernel_manager.start_kernel
        self.kernel_manager.start_kernel = partial(self.kernel_manager.start_kernel,
        env=env,
        kernel_headers=kernel_headers)
            try:
                await super(MainKernelHandler, self).post()
            finally:
                self.kernel_manager.start_kernel = orig_start
		else:
        await super(MainKernelHandler, self).post()

```

然后去注入相关环境变量去调用 `jupyter_server_handlers.post()` 去启动 `kernel`，`get` 方法为重写 super 的方法，获取 `kernel list` 并且加入了一些鉴权和 cors 跨域处理代码。

`handlers.py` 还会将所有 `jupyter_server` 的 handler 加上鉴权和 cors 跨域处理类继承（如果 `eg` 没有覆写该方法的话）。

**remotemanager 模块**

针对远程 kernel 的处理模块



#### processproxies

代理处理模块基类，封装了处理连接 kernel 相关的方法。

将 kernel 分为 `local` 和 `remote` kernel 分别用不同方法处理，`remote` 则另外区分其是否为本地 ip kernel

获取本地 ip 方法

```python
def _get_local_ip():
    """
    通过遍历禁止 ip 列表，定位第一个不在列表的 ip
    """
    for ip in jupyeter_client.localinterfaces.public_ips():
        is_prohibited = False
        for prohibited_ip in prohibited_local_ips:  # exhaust prohibited list, applying regexs
            if re.match(prohibited_ip, ip):
                is_prohibited = True
                break
        if not is_prohibited:
            return ip
    # 全部被禁止所以使用第一个 ip
    return localinterfaces.public_ips()[0] 
```

**ProcessProxy 抽象类 `BaseProcessProxyABC`**

1. 提供了基础 `launch_process` 抽象方法，所有子类都应该调用该方法来进行一些通用的环境初始化和基本鉴权，具体启动则由子类实现。子类同样需要执行 `cleanup` 方法（kernel 关闭后的清理工作）

2. 提供了轮询方法 `poll` 以及轮询方法 `wait`。`poll` 检测 process proxy 是否存活，如果为本地 `local_proc` 则调用其的 `poll` 否则使用 `send_signal` 方法。wait 方法，等待进程到非活跃状态（阻塞轮询），有本地则调用本地，否则执行 `poll` 方法直到返回 false

   ​		Send_signal 方法发送 `signum` 给 process proxy

   ```python
   """send_signal"""
   def send_signal(self, signum):
   """
   Send signal `signum` to process proxy.
   
   Parameters
   ----------
   signum : int
     The signal number to send.  Zero is used to determine heartbeat.
   """
     result = None
     if self.local_proc:
       if self.pgid > 0 and hasattr(os, "killpg"):
         try:
           os.killpg(self.pgid, signum)
           return result
         except OSError:
           pass
       result = self.local_proc.send_signal(signum)
     else:
       if self.ip and self.pid > 0:
         if BaseProcessProxyABC.ip_is_local(self.ip):
           # local_signal 通过 subprocess.call() 执行 kill -signum
           result = self.local_signal(signum)
         else:
           # remote_signal 使用 ssh 通道远程连接执行 kill -signum
           result = self.remote_signal(signum)
     return result
   ```

3. `kill` 方法的实现，首先调用 `terminate` 正常关闭 process proxy （kill -15)，失败次数超出后则使用 `kill -9` 强行关闭进程
4. `select_ports` 方法，通过 `socket` 随机选用 `count` 个可用端口（范围为 `port_range` 中）

端口校验方法实现

```python
"""端口校验，是否符合规定以及在 port_range 中"""
def _validate_port_range(self):
    """
    Validates the port range configuration option to ensure appropriate values.
    """
    # Let port_range override global value - if set on kernelspec...
    port_range = self.kernel_manager.port_range
    if self.proxy_config.get('port_range'):
        port_range = self.proxy_config.get('port_range')

    try:
        port_ranges = port_range.split("..")

        self.lower_port = int(port_ranges[0])
        self.upper_port = int(port_ranges[1])

        port_range_size = self.upper_port - self.lower_port
        if port_range_size != 0:
            if port_range_size < min_port_range_size:
                self.log_and_raise()

   					# According to RFC 793
            if self.lower_port < 1024 or self.lower_port > 65535:
                self.log_and_raise()
            if self.upper_port < 1024 or self.upper_port > 65535:
                self.log_and_raise()
    except ValueError as ve:
        self.log_and_raise()
    except IndexError as ie:
        self.log_and_raise()

    self.kernel_manager.port_range = port_range
```

选择端口方法实现

```python
def select_ports(self, count):
    ports = []
    sockets = []
    for i in range(count):
        sock = self.select_socket()
        ports.append(sock.getsockname()[1])
        sockets.append(sock)
    for sock in sockets:
        sock.close()
    return ports

def select_socket(self, ip=''):
    """
    创建并返回符合配置端口的 socket 连接
    Parameters
    ----------
    ip : str
        Optional ip address to which the port is bound
    Returns
    -------
    socket - Bound socket that is available and adheres to configured port range
    """
    sock = socket(AF_INET, SOCK_STREAM)
    found_port = False
    retries = 0
    while not found_port:
        try:
            # _get_candidate_port 随机选择一个符合规则的端口
            sock.bind((ip, self._get_candidate_port()))
            found_port = True
        except Exception:
            retries = retries + 1
            if retries > max_port_range_retries:
                self.log_and_raise('log info')
    return soc
```

**LocalProcessProxy**

本地启动 process proxy 的实现

```python
async def launch_process(self, kernel_cmd, **kwargs):
    await super(LocalProcessProxy, self).launch_process(kernel_cmd, **kwargs)

    # launch the local run.sh
    # 调用了 jupyter_client.launch_kernel()
    self.local_proc = self.launch_kernel(kernel_cmd, **kwargs)
    self.pid = self.local_proc.pid
    if hasattr(os, "getpgid"):
        try:
            self.pgid = os.getpgid(self.pid)
        except OSError:
            pass
          
    # local_ip = _get_local_ip()
    self.ip = local_ip
    return self
```

**RemoteProcessProxy**

远程创建 process proxy 的基类，实现了基本的通用方法，具体方法将根据 `k8s` `yarn` `docker_swarm` 等分布式系统的不同来具体实现。

在 `__init__` 方法中注册了 `response_manager` 为一个单例管理响应的管理器（1. 加密解密响应相关信息 2. 创建一个基于配置的监听定期回调的 socket 3. 将响应的负载挂载到 kernel_id 映射中 4. 为调用者提供等待机制，根据 kernel_id 轮询获取它们的连接信息）

子类需要实现自己的 `launch_process` 以及 `confirm_remote_startup` 方法，后者为确定远程 kernel 是否启动的相关方法

基类为后者提供帮助方法 `detect_launch_failure` 检测是否 `self.local_proc` 异常终止

还提供了如下方法

1. `receive_connection_info` 监控远程地址通过连接信息发送的响应地址，可以接受 socket  轮询，返回加密好的信息以及连接就绪符

   ```python
   async def receive_connection_info(self):
     ready_to_connect = False 
     try:
       connect_info = await self.response_manager.get_connection_info(self.kernel_id)
       # 设置相关信息到 self 中，启动 5 ZMQ kernel ports 以及 ip 相关信息
       self._setup_connection_info(connect_info)
       ready_to_connect = True
     except:
       # handler except
     return ready_to_connect
   ```

2. `handler_timeout` 超时处理方法

```python
async def handle_timeout(self):
    """
    轮询方法
    """
    await asyncio.sleep(poll_interval)
    time_interval = RemoteProcessProxy.get_time_diff(self.start_time, RemoteProcessProxy.get_current_time())
    if time_interval > self.kernel_launch_timeout:
        # handle except
        await asyncio.get_event_loop().run_in_executor(None, self.kill)
        # handle log print

@staticmethod
def get_current_time():
    # Return the current time stamp in UTC time epoch format in milliseconds, e.g.
    return timegm(_tz.utcnow().utctimetuple()) * 1000

@staticmethod
def get_time_diff(time1, time2):
    # Return the difference between two timestamps in seconds, assuming the timestamp is in milliseconds
    # e.g. the difference between 1504028203000 and 1504028208300 is 5300 milliseconds or 5.3 seconds
    diff = abs(time2 - time1)
    return float("%d.%d" % (diff / 1000, diff % 1000))
```

3. `listener` 相关处理方法，调用者可以传入 `comm_port` 加入监听器，设置了 `comm_port` 后 `remote process proxy` 就会通过 socket 来通过该端口进行其他的通信

```python
def _send_listener_request(self, request, shutdown_socket=False):
    if self.comm_port > 0:
        sock = socket(AF_INET, SOCK_STREAM)
        try:
            sock.settimeout(socket_timeout)
            sock.connect((self.comm_ip, self.comm_port))
            sock.send(json.dumps(request).encode(encoding='utf-8'))
        finally:
            if shutdown_socket:
                try:
                    sock.shutdown(SHUT_WR)
                except Exception as e2:
                     # handle except
            sock.close()
    else:
        self.log.debug(
            "Invalid comm port, not sending request '{}' to comm_port '{}'.",
            request,
            self.comm_port,
        )
```

