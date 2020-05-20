---
layout:     post
title:      "Go小记 - channel基础"
subtitle:   ""
date:       2020-05-20 02:40:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

### 起

```
Don't communicate by sharing memory, share memory by communicating.
```

channel是一个类型安全的队列(循环队列),能够运用在不同的gorutine中交换任意的资源.


#### 定义方式

```
// 定义一个int类型的channel,未初始化,nil
var c chan int

// 定义并初始化了一个无缓冲的int类型的channel
c1 := make(chan int)

// 定义并初始化了一个有缓冲的int类型的channel
c2 := make(chan int,1)

// 定义了一个只能用于写int类型数据的单向channel
var c3 chan <- int
// 定义了一个只能用于读int数据的单向channel
var c4 <- chan int
```
- 向只读channel写或只写channel读数据都会panic
- 单向channel不能转换成双向
- 更多的是用于函数参数传递，作为默认配置，避免对channel的操作行为不确定(随处读写)

#### 无缓冲channel

```
ch := make(chan int)
defer close(ch)

//ch<-5 // 位置一

go func(ch chan int) {
    num := <-ch
    fmt.Println(num)
}(ch)

// ch<-5 // 位置二
```
打开位置1，程序会报错，向无缓冲channel发送数据，若无对应接受，程序会阻塞

打开位置2，若主程序在gorutine运行后未结束，则可以打印出相关信息


#### 有缓冲channel

与上诉同样的代码，则打开任意位置，程序都能正常运行

##### 阻塞和panic

1. 向channel发送数据

- 向 nil 通道发送数据会被阻塞
- 向无缓冲channel写数据，如果读协程未准备好，会阻塞
- 向有缓冲channel写数据，若队列已满，会阻塞
- 向closed的channel写数据会panic
- 有缓冲的channel也不是很次发送、接受都要经过缓存，如果发送的时候刚好有接受的协程，那么会直接交换数据

2. 从channel接受数据

两种读取方式

```
1. multi-valued assignment
v, ok := <-ch
ok 是一个bool类型，可以通过它来判断channel是否已经关闭，如果关闭则ok为true,此时
接受到的是channel类型的零值。

2. 直接读取
v := <- ch
```

- 从nil通道接受数据会被阻塞
- 从无缓冲channel读数据，如果写协程未准备好，会阻塞
- 从有缓冲channel读数据，若缓冲为空，会阻塞
- 读取的channel如果被关闭，并不会影响正在读的数据，他会将所有数据读取完后关闭，并不会立即关闭或返回零值


3. 其他
再次关闭已关闭的channel会panic

若主程序在发送或接受channel的时候阻塞，同时，系统判断该程序中其他的gorutine都已执行完，那么程序会panic

```
fatal error: all goroutines are asleep - deadlock!
```

因为程序在等待一个永远不可能到来的channel。
##### range

可使用range语句读取channel

使用该语句，若channel已被关闭，它还是会继续执行，直到所有值被取完，然后退出执行。
若通道未关闭，但是channel没有可读的数据，它则会阻塞在range这句话上，直到被唤醒。
若channel是nil，它也会一直阻塞在range上

##### select
select 是被专门设计出来处理通道的，因为每个case后面跟的都是通道表达式，可以是读,
也可以是写。

```
ch := make(chan int)
q := make(chan int)

go func(ch, q chan int) {
    for i := 0; i < 10; i++ {
        num := <-ch
        fmt.Println(num)
    }
    q <- 1
}(ch, q)

fibonacci := func(ch, q chan int) {
    x, y := 0, 1
    for {
        select {
        case ch <- x: // 写入
            x, y = y, x+y
            // 你觉得是否会影响 for 语句的循环？
            // 不影响，只break出select
            break
        case <-q: // 读取
            fmt.Println("quit")
            return
        }
    }
}
fibonacci(ch, q)

```

- select 只要有默认语句，就不会被阻塞，换句话说，如果没有 default，然后 case 又都不能读或者写，则会被阻塞
- nil 的 channel，不管读写都会被阻塞
- select 不能够像 for-range 一样发现 channel 被关闭而终止执行，所以需要结合 multi-valued assignment 来处理
- 如果同时有多个 case 满足了条件，会使用伪随机选择一个 case 来执行
- select 语句如果不配合 for 语句使用，只会对 case 表达式求值一次
- 每次 select 语句的执行，是会扫码完所有的 case 后才确定如何执行，而不是说遇到合适的 case 就直接执行了


#### 总结

重点

1. 向 close 的 channel 写数据、再次 close 都会触发 runtime panic
2. 向 nil channel 写、读取数据，都会阻塞，可以利用这点来优化 for + select 的用法。
3. channel 的关闭最好在写入方处理，读的协程不要去关闭 channel，可以通过单向通道来表明 channel 在该位置的功能
4. 如果有多个写协程的 channel 需要关闭，可以使用额外的 channel 来标记，也可以使用 sync.Once 或者 sync.Mutex 来处理
5. channel 不管是读写都是并发安全的，不会出现多个协程同时读或者写的情况，从而实现了 CSP