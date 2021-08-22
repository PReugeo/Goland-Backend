## 背景

go 泛型目前已经在 1.17 合入 master 分支，预计在 Go1.18 实现。

在 1.17 中 Go 泛型像早期 Go mod 一样，需要先设置类似 `GO111MODULE` 类似的变量，目前使用 -G 标识泛型的开关

* -G=0：继续使用传统类型检查器
* -G=1：使用 types2，不支持泛型【目前在 go1.17 默认开启】
* -G=2：使用 tpyes2，支持泛型

目前在 go1.17 指定 `-gcflags=-G=3` 来开启编译器泛型语法支持

> 目前泛型方法无法被导出（首字母大写）

## 基本语法

```go
package main

import "fmt"

type Count struct {
	Name  string
	Count int
}

func printSlice[T any] (s []T) {
	for _, v := range s {
		fmt.Printf("%v", v)
	}
	fmt.Println()
}

func main() {
	printSlice[int]([]int{1, 2, 3, 4, 5, 6, 7})
	printSlice[float64]([]float64{1.2, 1.5, 1.7, 1.9})
	printSlice[string]([]string{"AAA", "BBB", "CCC"})
	printSlice[Count]([]Count{
		{
			Name: "abc",
			Count: 1,
		},
		{
			Name: "bbb",
			Count: 2,
		},
	})
	printSlice([]Count{
		{
			Name: "abc",
			Count: 1,
		},
		{
			Name: "bbb",
			Count: 2,
		},
	})
}
```

通过编译

```powershell
$ go run -gcflags=-G=3 main.go
1234567
1.21.51.71.9
AAABBBCCC
{abc 1}{bbb 2}
{abc 1}{bbb 2}
```

代码中 `[T any]` 为类型参数（Type Parameters），代表该函数支持任何 T 类型

在调用该范型函数时，可以显式指定类型参数类型，如：`printSlice[int]([]int{1, 2, 3, 4, 5})`，以帮助编译器实行类型推导； 不过在编译器完全可以实现类型推导时，也可以省略显式类型，如：``printSlice([]int64{5, 4, 3, 2, 1})`

### 泛型类型

声明一个泛型类型的切片，并将其实例化打印出来。

使用泛型类型，需要先将其实例化

```go
package main

import (
	"fmt"
)

# 泛型切片
type tslice[T any] []T


func printSlice[T any] (s []T) {
	for _, v := range s {
		fmt.Printf("%v ", v)
	}
	fmt.Println()
}


func main() {
	v := tslice[int]{1, 2}
	printSlice(v)

	v2 := tslice[string]{"123", "456"}
	printSlice(v2)
}
```

### 类型函数约束

通过声明类型列表表达式来对类型参数进行约束

```go
type Addable interface {
	type int, int8, int16, int32, int64, uint, uint8, uint16, uint32,
	uint64, uintptr, float32, float64, complex64, complex128, string
}

type X struct {
	Num int
}

func add[T Addable] (a, b T) T {
	return a + b
}

func main() {
	fmt.Println(add(3, 4))
	fmt.Println(add("Go", "lang"))
    fmt.Println(add(X{1}, X{2}))
}

// .\main.go:21:17: X does not satisfy Addable (struct{Num int} not found in int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, uintptr, float32, float64, complex64, complex128, string)
```

> 类型接口无法用作变量类型

### 接口约束泛型

可以通过接口来约束类型参数

```go
package main

import (
	"strconv"
	"fmt"
)

type Price int

type ShowPrice interface {
	String() string
}

func (i Price) String() string {
	return strconv.Itoa(int(i))
}

func show[T ShowPrice] (s []T) (ret []string) {
	for idx, v := range s {
		ret = append(ret, "第"+strconv.Itoa(idx+1)+"价格为："+v.String())
	}
	return ret
} 

func main() {
	s := []Price{25, 29, 128, 219, 169}
	ret := show(s)
	fmt.Println(ret)
}

```

### 接口类型约束泛型

类型参数可以同时被接口和类型约束

```go
package main

import (
	"strconv"
	"fmt"
)

type Price int
type Dish string

type ShowPrice interface {
	type int, int8, int16, int32, int64
	String() string
}

func (i Price) String() string {
	return strconv.Itoa(int(i))
}

func (d Dish) String() string {
	return string(d)
}

func show[T ShowPrice] (s []T) (ret []string) {
	for idx, v := range s {
		ret = append(ret, "第"+strconv.Itoa(idx+1)+"价格为："+v.String())
	}
	return ret
} 

func main() {
	s := []Price{25, 29, 128, 219, 169}
	str := []Dish{"123", "456", "28"}
	ret := show(s)
	fmt.Println(ret)
	ret = show(str)
	fmt.Println(ret)
}
// Dish does not satisfy ShowPrice (string not found in int, int8, int16, int32, int64)
```

### 泛型比较

Go 不支持运算符重载， 因此，我们即便对interface做了语法扩展，依然无法表达类型是否支持==和!=。

Go 引入了 `comparable` 这个预定义的类型约束

```go
package main

import "fmt"

func index[T comparable] (s []T, x T) int {
	for i, v := range s {
		if v == x {
			return i
		}
	}
	return -1
}

type Foo struct {
	a string
	b int
}

func main() {
	fmt.Println(index([]int{1, 2, 3, 4, 5}, 3)) 
	fmt.Println(index([]string{"a", "b", "c", "d", "e"}, "d")) 
	fmt.Println(index([]Foo{ {"a", 1}, {"b", 2}, {"c", 3}, {"d", 4}, {"e", 5}, }, Foo{"b", 2}))
}
// 2
// 3
// 1
```

`comparable`可以看成一个由Go编译器特殊处理的、包含由所有内置可比较类型组成的  `type list` 的 `interface` 类型，因此可以被嵌套作为其他约束的接口类型中。

### 泛型方法

与泛型函数中类型参数的约束方法一样，我们也可以对泛型类型方法，做同样的约束，如实现一个 set 集合

```go
package main

import "fmt"

type addable interface {
	comparable
}

type set[T addable] map[T]struct{}

// add v to set s
func (s set[T]) add(v T) {
	s[v] = struct{}{}
}

// check if v contain in s
func (s set[T]) contain(v T) bool {
	_, ok := s[v]
	return ok
}

func (s set[T]) len() int {
	return len(s)
}

func (s set[T]) delete(v T) {
	delete(s, v)
}

// iterate invokes f on each element of s
func (s set[T]) iterate(f func(T)) {
	for v := range s {
		f(v)
	}
}

func print[T addable](s T) {
	fmt.Println(s)
}

func main() {
	s := make(set[int])
	s.add(1)
	s.add(11)
	s.add(111)
	fmt.Println(s)

	if s.contain(1) {
		fmt.Println("the set contain 1")
	} else {
		fmt.Println("the set do not contain 1")
	}

	fmt.Println(s.len())

	s.delete(11)
	if s.contain(11) {
		fmt.Println("the set contain 11")
	} else {
		fmt.Println("the set do not contain 11")
	}

	s.iterate(func(x int) {
		fmt.Println(x + 1)
	})
	
    // error: incompatibal type
    // cannot use print (value of type func[T addable](s T)) as func(int) value
	s.iterate(print)
}
```

### 泛型引用方法

同样也可以声明引用方法

```go
package main 
import ( "fmt" ) 
type queue[T any] []T 
func (q *queue[T]) enqueue(v T) { 
    *q = append(*q, v) 
} 
func (q *queue[T]) dequeue() (T, bool) { 
    if len(*q) == 0 { 
        var zero T return zero, false 
    } 
    r := (*q)[0] *q = (*q)[1:] 
    return r, true 
} 
func main() { 
    q := new(queue[int])
    q.enqueue(5) 
    q.enqueue(6) 
    fmt.Println(q) 
    fmt.Println(q.dequeue()) 
    fmt.Println(q.dequeue()) 
    fmt.Println(q.dequeue()) 
}

```

泛型方法与通用的 go 方法基本一致

