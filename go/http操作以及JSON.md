# Go 操作 JSON

JSON（JavaScript Object Notation） 是一种轻量级的数据交换语言，常用于前后端数据交换。

接下来介绍 JSON 在 GO 中的使用

## 导入包

```go
import "encoding/json"
```



## 反序列化

### 1. 解析数据到结构体

```go
type Res struct {
    Id		int		`json:"id"`
    Some	string	`json:"some"`
}
```

首先定义一个用于接收 json 的结构体。

在结构体中定义的 `json:"id"` 为 go 语言自带特性 `Struct tag` json 这个包就是根据这个 tag 来解析 json 到结构体中。

然后我们自己定义一个 json 数据，也可以是从前端发过来的 json。

```go
jsonTest := []byte(`
	{
		"id": 1,
		"some": "hello"
	}
`)
```

接下来使用 json 包来处理这个 json 串

```go
result := &Res{}

err := json.Unmarshal(jsonTest, result)
```

如果 JSON 数据不包含某字段，或者某些字段不需要转换为 JSON 那么这些会被忽略。

```go
package main

import (
	"fmt"
	"encoding/json"
)


type Res struct {
    Id		int		`json:"id"`
	Some	string	`json:"some"`
	Ok		int
	More	[]string `json:"more"`
}

func main() {
	jsonTest := []byte(`
		{
			"id": 1,
			"some": "hello"
		}
	`)

	result := &Res{}

	_ = json.Unmarshal(jsonTest, result)

	fmt.Println(result)
    //output:
    //&{1 hello 0 []}
}
```



### 2. 解析数据到其他类型

#### map

map的 key 必须定义为 string 类型，key 可以定义为 json 类型或者 interface{} 例如：

```go
jsonTest := []byte(`
		{
			"id": 1,
			"some": "hello"
		}
	`)

mapTest := make(map[string] interface{})

_ = json.Unmarshal(jsonTest, &mapTest)

fmt.Println("data:", mapTest)

```



### 2. JSON 数据与 Go 数据对应关系

`sturct tag` 可以指明 json 的一些行为，一般只会反序列化到大写首字母的结构体字段，如果要跳过即不反序列某大写字段，可在其后面添加 tag `json:"-"` 与 go 占位符同理，这样就跳过该字段了。

若要指定某字段的值为某对应类型的 `zero-value` ，即序列化后的 JSON 不包含此字段，可以在该字段添加 tag `json:"omitempty"`。

```go
type Res struct {
    Id		int		`json:"id"`
	Some	string	`json:"some"`
    Ok		int		`json:"ok, omitempty"`
	More	[]string `json:"more"`
}
```

如果 ok == 0，则不会被序列化。



JSON 数据格式与 Go 数据的关系：

* bool 对应 JSON 的 boolean
* float64 对应 JSON 的 numbers
* string 对应 JSON 的 strings
* nil 对应 JSON 的 null

如果在反序列时不知道 JSON 数据的具体类型，可以用空接口 `interface{}` 接收该数据。



## 序列化

把 Go `struct` 转换为 JSON，可以使用 Marshal 方法

```go
type Res struct {
    Id		int		`json:"id"`
	Some	string	`json:"some"`
    Ok		int		`json:"ok, omitempty"`
	More	[]string `json:"more"`
}

res := Res{1, "111", 1, ["abc", "aaa"]}

b, err := json.Marshal(res)
fmt.Println(string(b))
```

在 Go 中并不是所有的类型都能进行序列化：

- JSON object key 只支持 string
- Channel、complex、function 等 type 无法进行序列化
- 数据中如果存在循环引用，则不能进行序列化，因为序列化时会进行递归
- Pointer 序列化之后是其指向的值或者是 nil



## JSON 流处理

json 包还提供了 `NewEncode` 和 `NewDecoder` ，提供 `Encoder` 和 `Decoder` 类型，可支持 `io.Writer` `io.Reader` 接口做流处理

### 1. Decoder

```go
jsonFile, err := os.Open("test.json")
if err != nil {
    fmt.Printf("Error opening file: %v", err)
    return
}

defer jsonFile.Close()

decode := json.NewDecoder(jsonFile)
for {
    var res []Res
    err := decode.Decode(&res)
    if err == io.EOF {
        break
    }

    if err != nil {
        fmt.Println("Error of decode file", err)
        return
    }

    fmt.Println(res)
}
//output:
//[{1 hello 0 []} {2 world 0 []}]
```

其中 `test.json` 文件内容为：

```json
[
    {
    	"id": 1,
    	"some": "hello"
    },
    {
        "id": 2,
        "some": "world"
    }
]
```

### 2. Encoder

```go
package main

import (  
    "encoding/json"
    "os"
)

type Person struct {  
    Name   string
    Age    int
    Emails []string
}

func main() {  
    bingo := Person{
        Name:   "Bingo Huang",
        Age:    30,
        Emails: []string{"go@bingohuang.com","me@bingohuang.com"},
    }
    fileWriter, _ := os.Create("output.json")
    json.NewEncoder(fileWriter).Encode(bingo)
}
```





# Go HTTP 操作

## 获取 URL 中的 query

我们可以使用 go 语言自带 http 库来获取查询参数

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    res := r.URL.Query()
}
```

其中 r 为一个 `map[string][]string` 对象，类型为 `URL.Values`，其中 r 不会为 nil 值，如果没有查询参数，则 r 的成员为空大小为 0。



### 获取参数值

1. 通过 map 的 key 值获取

    1. ```go
        res := r.URL.Query()
        a, ok := res["id"]
        if ok {
            xxx
        }
        ```

2. 使用 Get 方法获取

    ```go
    a := res.Get("id")
    ```



## 获取 Form 中的值（ form-data ）

### 1. 获取 GET 参数

```go
func myHandler(w http.ResponseWriter, r *http.Request) {
    res := r.ParseForm()
    if len(r.Form["name"]) > 0 {
        fmt.Fprintln(w, r.Form["name"][0])
    }
}
```

如果在表单提交时，又有 get 方法，又有 post 参数且其名称一样时则需要处理一下,例如：

```html
<form action="http://127.0.0.1:8080/?name=mark" method="POST">
    <input type="text" name="name" value="best" />
    <input type="submit" value="submit" />
</form>
```

```go
form := r.URL.Query()
if len(form["name"]) > 0 {
    fmt.Fprintln(w, form["name"][0])
}
```



### 2. 获取 POST 参数

这里要分两种情况：

- 普通的post表单请求，Content-Type=application/x-www-form-urlencoded
- 有文件上传的表单，Content-Type=multipart/form-data

第一种情况比较简单，直接用PostFormValue就可以取到了:

```go
r.ParseForm()
fmt.Fprintln(w, r.PostFormValue("name"))
```

第二种情况复杂一些，如下表单：

```html
<form action="http://127.0.0.1:9090" method="POST" enctype="multipart/form-data">
    <input type="text" name="name" value="markbest" />
    <input type="file" name="pic" />
    <input type="submit" value="submit" />
</form>
```

因为需要上传文件，所以表单enctype要设置成multipart/form-data，此时需要使用ParseMultipartForm方可解析：

```go
r.ParseMultipartForm(32 << 20)
fmt.Println(r.PostForm)
fmt.Println(r.MultipartForm.File)
```

这样子就能获取post参数以及上传的文件参数。



### 3. 获取 Cookie 参数

```go
cookie, err := r.Cookie("id")
if err == nil {
    fmt.Fprintln(w, "Domain:", cookie.Domain)
    fmt.Fprintln(w, "Expires:", cookie.Expires)
    fmt.Fprintln(w, "Name:", cookie.Name)
    fmt.Fprintln(w, "Value:", cookie.Value)
}
```



### 4. 处理上传文件

```go
func UploadHandler(w http.ResponseWriter, r *http.Request) {
    //设置内存大小
	r.ParseMultipartForm(32 << 20)
	//获取上传的文件组
	files := r.MultipartForm.File["uploadfile"]
	len := len(files)
	for i := 0; i < len; i++ {
		//打开上传文件
		file, err := files[i].Open()
		defer file.Close()
		if err != nil {
			log.Fatal(err)
		}
		//创建上传目录
		os.Mkdir("./upload", os.ModePerm)
		//创建上传文件
		cur, err := os.Create("./upload/" + files[i].Filename)

		defer cur.Close()
		if err != nil {
			log.Fatal(err)
		}
		io.Copy(cur, file)
		fmt.Println(files[i].Filename) //输出上传的文件名
	}

	fmt.Fprintf(w, "upload success \n") 
}
```





参考：

https://golang.org/pkg/encoding/json/

https://sanyuesha.com/2018/05/07/go-json/

https://bingohuang.com/go-json/

http://static.markbest.site/blog/86.html