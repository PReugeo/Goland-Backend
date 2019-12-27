

# 辅助库

github.com/julienschmidt/httprouter

net/http



## net/http 官方库

net/http 库为 Golang 内置的处理 HTTP 请求的库，可以比较方便的开发一个 HTTP 服务。

客户端例子（ 通过 Get， Head， Post， PostForm 发送请求 ）：

```go
package main
import (
    "fmt"
    "io/ioutil"
    "net/http"
)

func main() {
    response, err := http.Get("http://www.baidu.com")
    if err != nil {
    // handle error
    }
    //程序在使用完回复后必须关闭回复的主体。
    defer response.Body.Close()

    body, _ := ioutil.ReadAll(response.Body)
    fmt.Println(string(body))
}
```



服务端例子：

```go
func main() {
	http.HandleFunc("/",Index)

	log.Fatal(http.ListenAndServe(":8080", nil))
}

func Index(w http.ResponseWriter, r *http.Request){
	fmt.Fprint(w,"hello, world")
}

hello, world
```

其中调用 `http.HandleFunc`调用了在源码中默认的 `DefaultServeMux` 路由的 `HandleFunc`, 其中 `DefaultServeMux` 的类型为 `ServeMux`: 

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}

type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}
```

然后调用了 `DefaultServeMux` 的 `Handle` 函数:

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	//省略加锁和判断代码

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	//把我们注册的路径和相应的处理函数存入了m字段中
	mux.m[pattern] = muxEntry{h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

`Handle` 函数将注册的路径和相应的处理函数存入了 m 字段中，即在 `DefaultServeMux` 的 m 字段中增加相应的 handler 和路由规则。

接着调用了 `http.LitenAndServe(":8080", nil)`

在源码中依此做了以下事情：

> - 实例化 Server
>
> - 调用 Server 的 ListenAndServe()
>
> - 调用 net.Listen(“tcp”, addr) 监听端口
>
> - 启动一个 for 循环，在循环体中 Accept 请求
>
> - 对每个请求实例化一个 Conn，并且开启一个 goroutine 为这个请求进行服务 go c.serve()
>
> - 读取每个请求的内容w, err := c.readRequest()
>
> - 判断 header 是否为空，如果没有设置 handler，handler 就设置为 DefaultServeMux
>
> - 调用 handler 的 ServeHttp
>
> - 在这个例子中，下面就进入到 DefaultServerMux.ServeHttp
>
> - 根据 request 选择 handler，并且进入到这个 handler 的 ServeHTTP
>
> - ```go
>     mux.handler(r).ServeHTTP(w, r)
>     ```
>
> - 选择 Handler 时：
>
>     - A 判断是否有路由能满足这个request（循环遍历ServerMux的muxEntry）
>
>         B 如果有路由满足，调用这个路由handler的ServeHttp
>
>         C 如果没有路由满足，调用NotFoundHandler的ServeHttp

### net/http 的不足

1. 不能单独的对请求方法(POST,GET等)注册特定的处理函数
2. 不支持Path变量参数
3. 不能自动对Path进行校准
4. 性能一般
5. 扩展性不足

解决可以自己实现一个路由，因为 golang 自带路由功能较弱因为使用的是默认路径

```go
type MyHandler struct{}
func (mh *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path == "/hello"{
        w.Write([]byte("hello page"))
        return //这里的return是要加的，不然下面的代码也会执行了
    }
    if r.URL.Path == "/world"{
        w.Write([]byte("world page"))
        return 
    }
    w.Write([]byte("root page"))
    // 可以继续写自己的路由匹配规则
}

//或者更简单一点
func main() {
    if err := http.ListenAndServe(":12345", MyHandler{});err != nil{
        fmt.Println("start http server faild:",err)
    }
}
```




## httprouter

 httprouter 是一个高性能、可扩展的 HTTP 路由，上面我们列举的`net/http`默认路由的不足，都被 httprouter 实现 ，它使用radix tree实现存储和匹配查找，所以效率非常高，内存占用也很低。golang 可以使用 httprouter 快速搭建 REST API 服务器。

例子：

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/julienschmidt/httprouter"
)

func Index(w http.ResponseWriter, r *http.Request, p httprouter.Params) {
	fmt.Fprint(w, "Welcome!\n")
}

func main() {
	router := httprouter.New()
	router.GET("/", Index)

	log.Fatal(http.ListenAndServe(":8080", router))
}
```

`Index` 为一个 handler 函数，需要传入三个参数，之后该函数被注册到 `/` 路径上



### httprouter 命名参数

现代的API，基本上都是REST API，httprouter提供的命名参数的支持，可以很方便的帮助我们开发Restful API。比如我们设计的API`/hello/zhangsan`，这这样一个URL，可以查看 `zhangsan` 这个用户的信息，如果要查看其他用户的，比如`lisi`,我们只需要访问API`/hello/lisi`即可。

现在我们可以发现，其实这是一种URL匹配模式，我们可以把它总结为`/hello/:name`,这是一个通配符。

```go
func Hello(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
    fmt.Fprintf(w, "hello, %s!\n", ps.ByName("name"))
}

func main() {
    router := httprouter.New()
    router.GET("/hello/:name", Hello)

    log.Fatal(http.ListenAndServe(":8080", router))
}
```

从官方文档上看，httprouter 提供的命名参数有两种：

| Syntax | Type       |
| ------ | ---------- |
| :name  | 命名参数   |
| *name  | 全匹配参数 |

命名参数的匹配规则：

```http
Path: /blog/:category/:post

Requests:
 /blog/go/request-routers            匹配: category="go", post="request-routers"
 /blog/go/request-routers/           不匹配，但是会重定向该路由
 /blog/go/                           不匹配
 /blog/go/request-routers/comments   不匹配
```

全匹配参数的匹配规则：

```http
Path: /files/*filepath

Requests:
 /files/                             match: filepath="/"
 /files/LICENSE                      match: filepath="/LICENSE"
 /files/templates/article.html       match: filepath="/templates/article.html"
 /files    
```

代码中获取命名参数和全匹配参数：

```go
func Hello(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
    category := ps.ByName("category")
    post := ps.ByName("post")
    //通过切片下标访问
    firstKey := ps[0].Key //第一个参数的健
    firstValue := ps[0].Value //第一个参数的值
}
```



### Handler 处理链

例子：对多个不同二级域名，进行不同路由处理

```go
//一个新类型，用于存储域名对应的路由
type HostSwitch map[string]http.Handler

//实现http.Handler接口，进行不同域名的路由分发
func (hs HostSwitch) ServeHTTP(w http.ResponseWriter, r *http.Request) {

    //根据域名获取对应的Handler路由，然后调用处理（分发机制）
	if handler := hs[r.Host]; handler != nil {
		handler.ServeHTTP(w, r)
	} else {
		http.Error(w, "Forbidden", 403)
	}
}

func main() {
    //声明两个路由
	playRouter := httprouter.New()
	playRouter.GET("/", PlayIndex)
	
	toolRouter := httprouter.New()
	toolRouter.GET("/", ToolIndex)

    //分别用于处理不同的二级域名
	hs := make(HostSwitch)
	hs["play.flysnow.org:12345"] = playRouter
	hs["tool.flysnow.org:12345"] = toolRouter

    //HostSwitch实现了http.Handler,所以可以直接用
	log.Fatal(http.ListenAndServe(":12345", hs))
}
```

 这个例子中,`HostSwitch`和`httprouter.Router`这两个`http.Handler`就组成了一个`http.Handler`处理链。 

### 静态文件服务

用来定义静态文件，比如 js，css，图片等静态资源。

`router.ServeFiles(path string, root http.FileSystem)`

其中 path 用来定义 URL 路径。

root 为本地文件目录，需要使用 `http.Dir("xxx")` 转换一下，因为 Dir 实现了 http.FileSystem 接口。

示例：如果定义路由 `router.ServeFiles("/public/*filepath", http.Dir("/etc/static"))` 则可以通过路由 localhost:8080/public/js, 将 /etc/static/js 的文件显示了。



### httprouter 异常捕获

可以通过设置 `PanicHandler` 来处理 HTTP 请求中出现的 panic。

```go
func Index(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	panic("故意抛出的异常")
}

func main() {
	router := httprouter.New()
	router.GET("/", Index)
	router.PanicHandler = func(w http.ResponseWriter, r *http.Request, v interface{}) {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, "error:%s",v)
	}

	log.Fatal(http.ListenAndServe(":8080", router))
}
```

 这是一种非常好的方式，可以让我们对painc进行统一处理，不至于因为漏掉的panic影响用户使用。 



# UUID ( 通用同一标识码 )

**通用唯一识别码**（英语：**U**niversally **U**nique **Id**entifier，缩写：**UUID**）是用于计算机体系中以识别信息数目的一个128位标识符，还有相关的术语：[全局唯一标识符](https://zh.wikipedia.org/wiki/全局唯一标识符)（GUID）。

根据标准方法生成，不依赖中央机构的注册和分配，UUID具有唯一性，这与其他大多数编号方案不同。重复UUID码概率接近零，可以忽略不计。



**相关库**

crypto/rand

## 代码

```go
func NewUUID() (string, error) {
    uuid := make([]byte, 16)
    n, err := io.ReadFull(rand.Reader, uuid)
    if n != len(uuid) || err != nil {
        return "", err
    }
    
    //variant bits; see section 4.1.1
    uuid[8] = uuid[8]&^0xc0 | 0x80
    //version4 (pseudo-random); see section 4.1.3
    uuid[6] = uuid[6]&^0xf0 | 0x40
    
    return fmt.Sprintf("%x-%x-%x-%x-%x", uuid[0:4], uuid[4:6], uuid[6:8], uuid[8:])
}
```



参考博客： https://www.kancloud.cn/digest/batu-go/153529 

 [https://www.flysnow.org/2019/01/07/golang-classic-libs-httprouter.html#httprouter-%E5%91%BD%E5%90%8D%E5%8F%82%E6%95%B0](https://www.flysnow.org/2019/01/07/golang-classic-libs-httprouter.html#httprouter-命名参数) 