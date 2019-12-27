# yaml 概述

YAML 是专门用来写配置文件的语言，非常简洁和强大，远比 JSON 格式方便。 

YAML 语言（发音 /ˈjæməl/ ）的设计目标，就是方便人类读写。它实质上是一种通用的数据串行化格式。

它的基本语法规则如下。

> - 大小写敏感
> - 使用缩进表示层级关系
> - 缩进时不允许使用Tab键，只允许使用空格。
> - 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可

`#` 表示注释，从这个字符一直到行尾，都会被解析器忽略。

YAML 支持的数据结构有三种。

> - 对象：键值对的集合，又称为映射（mapping）/ 哈希（hashes） / 字典（dictionary）
> - 数组：一组按次序排列的值，又称为序列（sequence） / 列表（list）
> - 纯量（scalars）：单个的、不可再分的值

例如：

```yaml
#即表示url属性值；
url: http://www.wolfcode.cn 
#即表示server.host属性的值；
server:
    host: http://www.wolfcode.cn 
#数组，即表示server为[a,b,c]
server:
    - 120.168.117.21
    - 120.168.117.22
    - 120.168.117.23
#常量
pi: 3.14   #定义一个数值3.14
hasChild: true  #定义一个boolean值
name: '你好YAML'   #定义一个字符串
```

# Go 操作 YAML

## 安装依赖

`go get gopkg.in/yaml.v2`

## 自定义 YAML 文件

```yaml
db:
  mysql:
    user: root
    pwd: 1234
    host: localhost
    dbName: db1
  mongo:
  	user: root
  	pwd: 1234
  	host: localhost
  	collection: cc
```



## Go 代码

```go
package main

import (
    "io/ioutil"
    "log"
    "gopkg.in/yaml.v2"
)

type DbConfig struct {
    Db	Db	`yaml:"db"`
}

type Db struct {
    MySqlConfig	Mysql	`yaml:"mysql"`
    MongoConfig Mongo	`yaml:"mongo"`
}

type Mysql struct {
    User	string	`yaml:"user"`
    Pwd		string	`yaml:"pwd"`
    Host	string	`yaml:"host"`
    DbName	string	`yaml:"dbName"`
}

type Mongo struct {
    User		string	`yaml:"user"`
    Pwd			string	`yaml:"pwd"`
    Host		string	`yaml:"host"`
    Collection	string	`yaml:"collection"`
}

func (c *DbConfig) GetConf() *DbConf {
	file, err := GetConfigPath()	//获取 yaml 的文件路径
	yamlFile, err := ioutil.ReadFile(file)
	if err != nil {
		log.Printf("yamlFile get err %v", err)
	}

	err = yaml.Unmarshal(yamlFile, c)
	if err != nil {
		log.Fatalf("Unmarshal err：%v", err)
	}
	return c
}

```

# Go 使用相对路径

该函数参考 beego 框架实现，即通过 `os.GetWd()` 来双重判断实现，可解决 go build 和在当前路径 go run 找不到文件问题

```go
package utils

import (
	"errors"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
)

/**
	判断文件是否存在
 */
func FileExists(name string) bool {
	if _, err := os.Stat(name); err != nil {
		if os.IsNotExist(err) {
			return false
		}
	}

	return true
}

/**
   获取父文件夹
 */
func GetParentDir(dir string) string {
	runes := []rune(dir)
	return string(runes[0: strings.LastIndex(dir, string(os.PathSeparator))])
}

/**
	获取文件执行路径 ( 可解决 go build 文件路径问题但解决不了 go run 路径问题 )
 */
func GetCurrentPath() string {
	s, err := exec.LookPath(os.Args[0])
	if err != nil {
		panic(err)
	}
	path, err := filepath.Abs(s)
	if err != nil {
		panic(err)
	}
	
	i := strings.LastIndex(s, string(os.PathSeparator))
	return path[:i]
}

/**
	获取配置文件路径(参考 Beego 实现)
 */
func GetConfigPath() (appConfigPath string, err error) {
	AppPath, err := filepath.Abs(filepath.Dir(os.Args[0]))
	//AppPath = GetParentDir(AppPath) //run
	//AppPath = GetParentDir(GetParentDir(AppPath)) //test
	if err != nil {
		return
	}
	workPath, err := os.Getwd()
	//workPath = GetParentDir(workPath) //run
	//workPath = GetParentDir(GetParentDir(workPath))
	if err != nil {
		return
	}

	var filename = "config.yml"
	appConfigPath = filepath.Join(workPath, filename)
	if !FileExists(appConfigPath) {
		appConfigPath = filepath.Join(AppPath, filename)
		if !FileExists(appConfigPath) {
			err = errors.New("config file can not find")
			//panic(workPath)
			return "", err
		}
	}
	return
}

```

