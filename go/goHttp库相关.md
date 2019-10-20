# goHttp库

### 辅助库

github.com/julienschmidt/httprouter

net/http

### UUID

**通用唯一识别码**（英语：**U**niversally **U**nique **Id**entifier，缩写：**UUID**）是用于计算机体系中以识别信息数目的一个128位标识符，还有相关的术语：[全局唯一标识符](https://zh.wikipedia.org/wiki/全局唯一标识符)（GUID）。

根据标准方法生成，不依赖中央机构的注册和分配，UUID具有唯一性，这与其他大多数编号方案不同。重复UUID码概率接近零，可以忽略不计。



**相关库**

crypto/rand

#### 代码

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

