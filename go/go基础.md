# go基础

## 变量与类型

放函数内或包内

var $variable type 可以以这种方式命名

var $variable 也可以这样命名

var() 可批量定义

:=也可以用于变量定义（只可在函数内使用）

```go
多重赋值: i,j = j, i 等于 t = i; i = j ;j = t ;
```

### 强制类型转换

```go
<结果类型> := <目标类型>(<表达式>)
var i int = 7
var2 := float32(i)

//var4 := []int8(i)
//var4会报错因为类型不兼容go
//指针转化
var1 := new(int32)
var2 := (*int32)(var1)

```

字符串强制类型转化为float

strconv.ParseFloat

### 数组

初始化： var arr1 [number]type(//[3]int)

var array […]type{valus} 

arr2 := [number]type{values}

 

数组的遍历: for I,v := range arr3 {}

i 为下标 v为值

或者 for i:=0 ;i<len(array);i++{}



### 切片(slice)

 Slice（切片）代表变长的序列，序列中每个元素都有相同的类型。一个slice类型一般写作[]T，其中T代表slice中元素的类型；slice的语法和数组很像，只是没有固定长度而已。 

 一个slice是一个轻量级的数据结构，提供了访问数组子序列（或者全部）元素的功能，而且slice的底层确实引用一个数组对象。一个slice由三个部分构成：指针、长度和容量。指针指向第一个slice元素对应的底层数组元素的地址。



### Map

 哈希表是一种巧妙并且实用的数据结构。它是一个无序的key/value对的集合，其中所有的key都是不同的，然后通过给定的key可以在常数时间复杂度内检索、更新或删除对应的value。 

 在Go语言中，一个map就是一个哈希表的引用，map类型可以写为map[K]V，其中K和V分别对应key和value。map中所有的key都有相同的类型，所有的value也有着相同的类型，但是key和value之间可以是不同的数据类型。其中K对应的key必须是支持==比较运算符的数据类型，所以map可以通过测试key是否相等来判断是否已经存在。 

```go
ages := make(map[string]int) // mapping from strings to ints

ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}
```



### 结构体

 结构体是一种聚合的数据类型，是由零个或多个任意类型的值聚合成的实体。每个值称为结构体的成员。 

```go
type Employee struct {
    ID        int
    Name      string     `json:name`
    Address   string	 `json:address`
    DoB       time.Time
    Position  string
    Salary    int
    ManagerID int
}

var dilbert Employee
```



**指针对象的方法**

```go
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}

r := &Point{1, 2}
r.ScaleBy(2)
fmt.Println(*r) // "{2, 4}"

p := Point{1, 2}
pptr := &p
pptr.ScaleBy(2)
fmt.Println(p) // "{2, 4}"
```



**扩展已有类型**

```go
import "image/color"

type Point struct{ X, Y float64 }

type ColoredPoint struct {
    Point
    Color color.RGBA
}
```



## 函数

### 不定参数函数

```go
func myFunc(args …int){
	for _, arg := range args{
		fmt.Println(arg)
	}
}

```

该函数可接受复数个int类型的参数
不定参数一定是最后一个参数
…type为一个语法糖 = []type 的一个切片

### 匿名函数与闭包

匿名函数即闭包

 **概念**:是指不需要定义函数名的一种函数实现方式, 可以包含自由（未绑定到特定对象）变量的代码块，这些变量不在这个代码块内或者任何全局上下文中定义，而是在定义代码块的环境中定义。

要执行的代码块（由于自由变量包含在代码块中，所以这些自由变量以及它们引用的对象没有被释放）为自由变量提供绑定的作用域.

**作用**: 可以作为函数对象或匿名函数，可以作为变量传给其他函数

能够被函数动态创建和返回

例子:

```go
func incr() func() int {
	var x int
	return func() int {
		x++
		return x
	}
}

i := incr() //此时i便是闭包
//i保存着对x的引用
//由于i有指向x的指针所以可以修改x的值
println(i()) // 1
println(i()) // 2
println(i()) // 3

//但是直接调用不会递增
println(incr()()) // 1
println(incr()()) // 1
println(incr()()) // 1
```



## 接口

定义 :

```go
type Name interface {
	function()   
}
```

`function()`  

`}`

只要一个结构体中实现了接口中的所有函数,那它就完成了这个接口

### **接口赋值:**

1. 对象实例赋值给接口:
    1. 对象要求实现该接口
2. 接口1赋值给接口2:
    1. 要求接口2包含于接口1

### **接口组合**

例:

```go
type ReadWriter interface {

Read

Writer

}
```

组合只包含接口方法不包含成员变量



### 类型断言

 类型断言是一个使用在接口值上的操作。语法上它看起来像x.(T)被称为断言类型，这里x表示一个接口的类型和T表示一个类型。一个类型断言检查它操作对象的动态类型是否和断言的类型匹配。 

```go
var w io.Writer
w = os.Stdout
f := w.(*os.File)      // success: f == os.Stdout
c := w.(*bytes.Buffer) // panic: interface holds *os.File, not *bytes.Buffer。

//防止报错
var w io.Writer = os.Stdout
f, ok := w.(*os.File)      // success:  ok, f == os.Stdout
b, ok := w.(*bytes.Buffer) // failure: !ok, b == nil
```



## 错误处理

1. error接口：

go语言中错误处理的标准模式

error接口定义： 

```go
type error interface {
    Error() string
 }
```

error一般作为多返回值的最后一个参数

 

1. defer：

确保在函数结束时发生

数据结构为栈: 先进后出

defer 作用是在抛出异常时仍执行defer后面的函数（闭包）

defer srcFile.Close()或者 defer func() {}

 

1. panic()和recover()函数

2. 1. panic():[尽量少用]

停止当前函数执行

一直向上返回,执行每一层defer

若没看见recover,程序退出

1. recover:

2. 1. 仅在defer调用中调用
    2. 获取panic的值
    3. 若无法处理,可以重新panic

 

这两个函数用于报告和处理运行时的错误和程序中的错误场景

## 测试与调试

表格驱动测试:

1. 分离测试数据和测试逻辑
2. 明确出错信息
3. 可以部分失败(不会挂程序)
4. go语言语法适合

测试函数名TestXXXX(t *testing.T)

t.Errorf("Error message") //打出测试错误信息(自定义)



性能测试:

BenchmarkXX(b *testing.B)

重复测试一个较难算的实例b.N次



查看程序所用时间具体在哪:

| 终端: | go  test -bench . -cpuprofile |
| ----- | ----------------------------- |
|       |                               |

go tool pprof cpu.out

web(需要安装graphviz)

 

根据web所给的图来进行性能调优