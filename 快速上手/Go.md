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
rune // int32 的别名 代表一个 Unicode 码
float32、float64
complex64、complex128
```

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

for a := 0;a<10;a++{
    // 循环代码
}

if 表达式{
    // 表达式成立
}

# 只有 i++，前置自增++i，或者赋值后自增a=i++都将导致编译错误
i++

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
