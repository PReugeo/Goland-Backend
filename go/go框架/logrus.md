## Logrus

Golang 关于日志处理有很多包可以使用，标准库提供的 log 包功能比较少，不支持日志级别的精确控制，自定义添加日志字段等。在众多的日志包中，更推荐使用第三方的 logrus 包，完全兼容自带的 log 包。logrus 是目前 Github 上 star 数量最多的日志库，logrus 功能强大，性能高效，而且具有高度灵活性，提供了自定义插件的功能。

### logrus 特性

- 完全兼容 golang 标准库日志模块：logrus 拥有六种日志级别：debug、info、warn、error、fatal 和 panic，这是 golang 标准库日志模块的 API 的超集。
- `logrus.Debug(“Useful debugging information.”`
- `logrus.Info(“Something noteworthy happened!”)`
- `logrus.Warn(“You should probably take a look at this.”)`
- `logrus.Error(“Something failed but I’m not quitting.”)`
- `logrus.Fatal(“Bye.”)` //log之后会调用os.Exit(1)
- `logrus.Panic(“I’m bailing.”)` //log之后会panic()
- 可扩展的 Hook 机制：允许使用者通过 hook 的方式将日志分发到任意地方，如本地文件系统、标准输出、logstash、elasticsearch 或者 mq 等，或者通过 hook 定义日志内容和格式等。
  - [mgorus](https://link.segmentfault.com/?url=https%3A%2F%2Fgithub.com%2Fweekface%2Fmgorus)：将日志发送到 mongodb；
  - [logrus-redis-hook](https://link.segmentfault.com/?url=https%3A%2F%2Fgithub.com%2Frogierlommers%2Flogrus-redis-hook)：将日志发送到 redis；
  - [logrus-amqp](https://link.segmentfault.com/?url=https%3A%2F%2Fgithub.com%2Fvladoatanasov%2Flogrus_amqp)：将日志发送到 ActiveMQ。
  - [lfshook](https://github.com/rifflock/lfshook)：写入文件 hook
- 可选的日志输出格式：logrus 内置了两种日志格式，JSONFormatter 和 TextFormatter，如果这两个格式不满足需求，可以自己动手实现接口 Formatter 接口来定义自己的日志格式。
- Field 机制：logrus 鼓励通过 Field 机制进行精细化的、结构化的日志记录,而不是通过冗长的消息来记录日志。
- logrus 是一个可插拔的、结构化的日志框架。
- Entry: logrus.WithFields 会自动返回一个 *Entry，Entry里面的有些变量会被自动加上
- time:entry 被创建时的时间戳
- msg:在调用.Info()等方法时被添加
- level，当前日志级别

### 基本使用

```go
package main

import (
    "os"

    "github.com/sirupsen/logrus"
    log "github.com/sirupsen/logrus"
)

var logger *logrus.Entry

func init() {
    // 设置日志格式为json格式
    log.SetFormatter(&log.JSONFormatter{})
    log.SetOutput(os.Stdout)
    log.SetLevel(log.InfoLevel)
    logger = log.WithFields(log.Fields{"request_id": "123444", "user_ip": "127.0.0.1"})
}

func main() {
    logger.Info("hello, logrus....")
    logger.Info("hello, logrus1....")
    // log.WithFields(log.Fields{
    //  "animal": "walrus",
    //  "size":   10,
    // }).Info("A group of walrus emerges from the ocean")
		// 结构化日志数据
    // log.WithFields(log.Fields{
    //  "omg":    true,
    //  "number": 122,
    // }).Warn("The group's number increased tremendously!")

    // log.WithFields(log.Fields{
    //  "omg":    true,
    //  "number": 100,
    // }).Fatal("The ice breaks!")
}
```

## Ifshook

方便将 logrus 写入文件系统的 hook，能够快速配置 logrus

```go
var Log *logrus.Logger
func NewLogger() *logrus.Logger {
	if Log != nil {
		return Log
	}

	pathMap := lfshook.PathMap{
		logrus.InfoLevel:  "./info.log",
		logrus.PanicLevel: "./panic.log",
	}

	Log = logrus.New()

	Log.Hooks.Add(lfshook.NewHook(pathMap, &logrus.JSONFormatter{}))

	return Log
}
```

## file-rotatelogs

日志轮转帮助代码库，方便进行日志切分防止日志过大，提供一个 io.Writer 配合 logrus 实现日志分割

```go
var Log *logrus.Logger
func NewLoggerWithRotate() *logrus.Logger {
	if Log != nil {
		return Log
	}

	path := "./test.log"
	writer, _ := rotatelogs.New(
		path+".%Y%m%d%H%M",
		rotatelogs.WithLinkName(path),               // 生成软链，指向最新日志文件
		rotatelogs.WithMaxAge(60*time.Second),       // 文件最大保存时间
		rotatelogs.WithRotationTime(20*time.Second), // 日志切割时间间隔
	)

	pathMap := lfshook.WriterMap{
		logrus.InfoLevel:  writer,
		logrus.PanicLevel: writer,
	}

	Log = logrus.New()
	Log.Hooks.Add(lfshook.NewHook(
		pathMap,
		&logrus.JSONFormatter{},
	))

	return Log
}
```

