curl 在 post 请求字节大于 1024 字节时，会添加一个请求头，等 server 接收返回后在把数据 post 给 server

![30e692686c4001020518e4144789c081](/Users/yanjigang01/Golang-Backend/shell/image/30e692686c4001020518e4144789c081.png)

而有些 server 无法识别该请求头，则会报错 417，解决方案 

```shell
1. server 支持 expect 请求头
2. curl 请求添加 --http1.0 参数
```

