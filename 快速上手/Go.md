# 特性

* 语法简单严谨，左括号必须紧接着语句不换行
* 默认阻止指针运算，不允许包交叉依赖
* 切片和字典作为内置类型
* Goroutine 使并发非常容易，无须处理回调，无须关注线程切换
* 搭配 channel，实现 CSP 模型，无需纠结内存共享、锁粒度
* 使用 tcmalloc，使用 cache 为当前线程提供无锁内存分配
* 标准库 net/http，仅须简单几条语句就能实现一个高性能 Web Server
* 内置完整测试框架，包括单元测试、性能测试、代码覆盖率、数据竞争，以及用来调优的 pprof

# goroutine

* Go语言并发基于 goroutine，goroutine 类似于线程，但并非线程。可以将 goroutine 理解为一种虚拟线程。
* Go语言运行时会参与调度 goroutine，并将 goroutine 合理地分配到每个 CPU 中，最大限度地使用 CPU 性能。
* 多个 goroutine 中，Go语言使用通道（channel）进行通信，通道是一种内置的数据结构，可以让用户在不同的 goroutine 之间同步发送具有类型的消息。
* 这让编程模型更倾向于在 goroutine 之间发送消息，而不是让多个 goroutine 争夺同一个数据的使用权。
* 程序可以将需要并发的环节设计为生产者模式和消费者的模式，将数据放入通道，另一端代码将这些数据进行并发计算并返回结果。

例子：生产者每秒生成一个字符串，并通过通道传给消费者，生产者使用两个 goroutine 并发运行，消费者在 main() 函数的 goroutine 中进行处理。

```go
package main
import (
        "fmt"
        "math/rand"
        "time"
)
// 数据生产者
func producer(header string, channel chan<- string) {
     // 无限循环, 不停地生产数据
     for {
            // 将随机数和字符串格式化为字符串发送给通道
            channel <- fmt.Sprintf("%s: %v", header, rand.Int31())
            // 等待1秒
            time.Sleep(time.Second)
        }
}
// 数据消费者
func customer(channel <-chan string) {
     // 不停地获取数据
     for {
            // 从通道中取出数据, 此处会阻塞直到信道中返回数据
            message := <-channel
            // 打印数据
            fmt.Println(message)
        }
}
func main() {
    // 创建一个字符串类型的通道
    channel := make(chan string)
    // 创建producer()函数的并发goroutine
    go producer("cat", channel)
    go producer("dog", channel)
    // 数据消费函数
    customer(channel)
}
```

# 标准库

| Go语言标准库包名 |                              功  能                              |
|:----------------:|:----------------------------------------------------------------:|
| bufio            | 带缓冲的 I/O 操作                                                |
| bytes            | 实现字节操作                                                     |
| container        | 封装堆、列表和环形列表等容器                                     |
| crypto           | 加密算法                                                         |
| database         | 数据库驱动和接口                                                 |
| debug            | 各种调试文件格式访问及调试功能                                   |
| encoding         | 常见算法如 JSON、XML、Base64 等                                  |
| flag             | 命令行解析                                                       |
| fmt              | 格式化操作                                                       |
| go               | Go语言的词法、语法树、类型等。可通过这个包进行代码信息提取和修改 |
| html             | HTML 转义及模板系统                                              |
| image            | 常见图形格式的访问及生成                                         |
| io               | 实现 I/O 原始访问接口及访问封装                                  |
| math             | 数学库                                                           |
| net              | 网络库，支持 Socket、HTTP、邮件、RPC、SMTP 等                    |
| os               | 操作系统平台不依赖平台操作封装                                   |
| path             | 兼容各操作系统的路径操作实用函数                                 |
| plugin           | Go 1.7 加入的插件系统。支持将代码编译为插件，按需加载            |
| reflect          | 语言反射支持。可以动态获得代码中的类型信息，获取和修改变量的值   |
| regexp           | 正则表达式封装                                                   |
| runtime          | 运行时接口                                                       |
| sort             | 排序接口                                                         |
| strings          | 字符串转换、解析及实用函数                                       |
| time             | 时间接口                                                         |
| text             | 文本模板及 Token 词法器                                          |

# 指针

分为两种：

* 类型指针，允许对这个指针类型的数据进行修改，传递数据可以直接使用指针，而无须拷贝数据，类型指针不能进行偏移和运算。
* 切片，由指向起始元素的原始指针、元素数量和容量组成。

# 命令行

```bash
# 直接执行代码
go run main.go

# 编译
# 如果有main函数，生成可执行文件，否则只检查语法
go build
# 如果abc中有main函数，生成可执行文件abc
go build abc.go other.go

#打印本地环境变量
go env

# 生成文档注释
go doc package：获取包的文档注释，例如go doc fmt 会显示使用 godoc 生成的 fmt 包的文档注释；
go doc package/subpackage：获取子包的文档注释，例如go doc container/list；
go doc package function：获取某个函数在某个包中的文档注释，例如go doc fmt Printf 会显示有关 fmt.Printf() 的使用说明。

```

# 目录结构

go 项目目录结构是固定的，且引用需要使用环境变量GOPATH

* src 目录：放置项目和库的源文件；包在src子目录下。
* pkg 目录：放置编译后生成的包/库的归档文件；.a文件
* bin 目录：放置编译后生成的可执行文件。

# 常用代码

## 基本类型

```go
bool
string
int、int8、int16、int32、int64
uint、uint8、uint16、uint32、uint64、uintptr
byte // uint8 的别名
rune // int32 的别名 代表一个 Unicode 码，方便处理中文
float32、float64
complex64、complex128
所有值声明时都会初始化
const
常量：math.MaxFloat32 math.MaxFloat64 math.Pi
```

## fmt类型

* %v %d 整数
* %X 十六进制
* %U 带U+开头的十六进制，如：U+0041
* %p 打印指针或变量地址 &val
* %T 打印类型

## 基础语法

```go

import "name"

import(
    "name1"
    "name2"
)

func 函数名 (参数列表) (返回值列表){
    函数体
}

# var name type
var a, b *int
c := new(int) # 使用new函数申请的是指针类型
var name complex128 = complex(x, y)
# 使用 real 和 imag 函数返回复数的实部和虚部，如：real(name)

var (
    a int
    b string
    c []float32
    d func() bool
    e struct {
        x int
    }
)
# 有初值时可以省略类型
var a = 100
# \u 四子节 \U 八字节
var ch2 int = '\u03B2'
var ch3 int = '\U00101234'
# 交换值可以直接支持
b, a = a, b

# 简短格式 :=
#  定义变量，同时显式初始化。
#  不能提供数据类型。
#  只能用在函数内部。
i, j := 0, "abc"
# 匿名变量，不需要使用的变量
_, b := GetData()
# 多行
const str = `第一行
第二行
第三行
\r\n
`
str += "world!"
# 指针赋值
ptr := &v

for a := 0;a<10;a++{
    // 循环代码
}

if 表达式{
    // 表达式成立
}

# 只有 i++，前置自增++i，或者赋值后自增a=i++都将导致编译错误
i++

# 字符测试 ch 为 byte
#  判断是否为字母：unicode.IsLetter(ch)
#  判断是否为数字：unicode.IsDigit(ch)
#  判断是否为空白符号：unicode.IsSpace(ch)

//单行注释

/*
第一行注释
第二行注释
...
*/

```

## web server

```go
package main
import (
    "net/http"
)
func main() {
    http.Handle("/", http.FileServer(http.Dir(".")))
    http.ListenAndServe(":8080", nil)
}
```

## 自定义类型

```go
# 声明类型IntSlice
type IntSlice []int
# 实现 IntSlice.Len()
func (p IntSlice) Len() int           { return len(p) }
func (p IntSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p IntSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

# 类型别名
# 这个和上面的区别是，在类型打印时，上面是打印main.IntSlice，而这里还是会打印int
type IntAlias = int

#常量生成器，iota表示值从0开始递增
type Weekday int
const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
```


## 命令行参数解释

```go
var mode = flag.String("mode", "", "process mode")
func main() {
    // 解析命令行参数
    flag.Parse()
    // 输出命令行参数
    fmt.Println(*mode)
}
```
