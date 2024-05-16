通过WSGI 容器 `Gunicorn` 或者 `uWsgi` 来启动 http 服务。

## Gunicorn

[Gunicorn](https://gunicorn.org/) ‘Green Unicorn’ 是一个 UNIX 下的 WSGI HTTP 服务器，它是一个 移植自 Ruby 的 Unicorn 项目的 pre-fork worker 模型。它既支持 [eventlet](https://eventlet.net/) ， 也支持 [greenlet](https://greenlet.readthedocs.io/en/latest/) 。在 Gunicorn 上运行 Flask 应用非常简单:

```shell
$ gunicorn myproject:app
```

[Gunicorn](https://gunicorn.org/) 提供许多命令行参数，可以使用 `gunicorn -h` 来获得帮助。下面 的例子使用 4 worker 进程（ `-w 4` ）来运行 Flask 应用，绑定到 localhost 的 4000 端口（ `-b 127.0.0.1:4000` ）:

```shell
$ gunicorn -w 4 -b 127.0.0.1:4000 myproject:app
```

`gunicorn` 命令需要你应用或者包的名称和应用实例。如果你使用工厂模式，那么 可以传递一个调用来实现:

```shell
$ gunicorn "myproject:create_app()"
```

关闭重启 `gunicorn`

```shell
pstree -ap | grep gunicorn
kill -HUB pid
```

## 代理设置

通过 `nginx` 或者 `Apache` 来处理和响应 http 请求

示例 `nginx` 配置，代理目标为 `localhost:8000` 端口提供的服务，并且为响应设置了适当的头部

```nginx
server {
    listen 80;

    server_name _;

    access_log  /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass           http://127.0.0.1:8000/;
        proxy_redirect     off;
        proxy_set_header   Host                 $host;
        proxy_set_header   X-Real-IP            $remote_addr;
        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto    $scheme;
    }
}
```

