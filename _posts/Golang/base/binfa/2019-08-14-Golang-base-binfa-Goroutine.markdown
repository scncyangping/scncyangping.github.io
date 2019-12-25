---
layout:     post
title:      "Go并发编程 - (三)Goroutine"
subtitle:   ""
date:       2019-08-28 02:44:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### 示例

```
package main

import (
	"fmt"
	"time"
)
func main() {
	name := []string{"Eric", "Harry", "Robert", "Jim"}

	for _, d := range name {
		go func() {
			fmt.Printf("Hello,%s!\n", d)
		}()
	}

	time.Sleep(2 * time.Second)
}

输出：
Jim
Jim
Jim
Jim
```

goroutine开启之后执行的时间并不确定，在上述例子当中，4个新开的G其实封装的都是参数d，而参数d在for循环结束之后变成了Jim,所以会连续输出4个Jim



```
package main

import (
	"fmt"
	"time"
)
func main() {
	name := []string{"Eric", "Harry", "Robert", "Jim"}

	for _, d := range name {
		go func(tName string) {
			fmt.Printf("Hello,%s!\n", tName)
		}(d)
	}

	time.Sleep(2 * time.Second)
}
输出：
Eric
Harry
Robert
Jim
```
#### 主goroutine的运作

封装main函数的goroutine称为主goroutine。主goroutine会由runtime.m0负责运行

主goroutine所做的事儿不仅仅是执行main函数那么简单。它首先要做的是：设定每一个goroutine所能申请的栈空间最大尺寸。在32位计算机系统中此最大尺寸是250MNB,而在64位系统中最大尺寸是1G。若某一个超出了，会抛出一个栈溢出(stack overflow)。随机主goroutine也退出。

设定好尺寸过后，会在当前M的g0上(每一个M都有一个g0)执行系统检测任务。因为系统监测任务的作用就是为调度器查漏补缺的，这也是让系统监测任务的执行先于main函数的原因之一。

此后，主goroutine会进行一系列的初始化工作

- 检查当前M是否是runtime.m0，如果不是，就说明之前的程序出了某种问题。这时，会立即抛出异常
- 创建一个特殊的defer语句，用于在主goroutine退出时做必要的事儿。因为主goroutine也可能非正常的结束
- 启用专用于后台清扫内存垃圾的goroutine，并设置GC可用标识
- 执行main包的init函数


此后，会运行main函数。在main函数执行期间，运行时系统会根据Go程序中的go语句，复用或新建goroutine来封装go函数。这些goroutine都会被放入相应P的G队列中，然后等待调度器调用。

#### channel

在go语言中，channel既指通道类型，也指代可以传递某种类型的值的通道。通道即某一个通道类型的值。

与切片类型和字典类型相同，通道类型也属于引用类型。因此，在一个通道类型的变量在被初始化之前，其值一定是nil。

##### 定义

1. chan T
声明一个双向通信的channel
2. chan<- T
声明一个只能用于发送的channel
3. <-chan T
声明一个只能接受的channel
4. make(chan int,10)
初始化一个通道类型的值，该通道在同一时刻最多可缓冲10个元素值

str,ok := <-chan t

##### 特性
通道是在多个goroutine之间传递数据和同步的重要手段，而对通道的操作本身也是同步的。在同一时刻，仅有一个goroutine能向一个通道发送元素值，同时也仅有一个goroutine能从它那里接受元素值。各个元素值都是严格按照发送到此的先后顺序排列的。最早被发送至通道的元素值会最先被接受。因此，通道相当于一个FIFO的消息队列。此外，通道中的元素值都具有原子性，是不可分割的。通道中每一个元素值都只可能被某一个goroutine接受，已被接收的元素会立刻从通道中删除。

为了能够从通道接收元素值，必须先向其发送元素值。有以下规则：

- 发送操作会使通道复制被发送的元素。若因通道的缓冲空间已满而无法立即复制，则阻塞进行发送操作的goroutine。复制的目的地址有两种，当通道已空且有接收方在等待元素时，它会是最早等待的那个接收方持有的内存地址，否则会是通道持有的缓冲中的内存地址
- 接收操作会使得通道给出一个已发给它的元素值的副本，若因通道的缓冲空间已空而无法立即给出，则阻塞进行接收操作的goroutine。一般情况下，接收方会从通道持有的缓冲中得到元素值。
- 对于同一个元素值来说，把它发送给某个通道的操作，一定会在从该通道接收它的操作完成之前完成。换句话说，在通道完全复制一个元素值之前，任何goroutine都不可能从它那里接收到这个元素值的副本。

```
package main

import (
	"fmt"
	"time"
)

var strChan = make(chan string, 3)

func main() {
	syncChan1 := make(chan struct{}, 1)
	syncChan2 := make(chan struct{}, 2)

	go func() {
		res := <-syncChan1
		fmt.Println(res)
		fmt.Println("Received a sync signal")
		time.Sleep(2 * time.Second)
		for {
			if elem, ok := <-strChan; ok {
				fmt.Println("Receive:", elem, "[receiver]")
			} else {
				break
			}
		}
		fmt.Println("Stopped,[receiver]")
		syncChan2 <- struct{}{}
	}()

	go func() {
		for _, elem := range []string{"a", "b", "c", "d"} {
			strChan <- elem
			fmt.Println("Send:", elem, "[sender]")
			if elem == "c" {
				syncChan1 <- struct{}{}
				fmt.Println("Send a sync singal,[sender]")
			}
		}
		fmt.Println("wait 3 seconds [sender]")
		time.Sleep(time.Second * 3)

		close(strChan)
		syncChan2 <- struct{}{}
	}()

	<-syncChan2
	<-syncChan2
}
输出
Send: a [sender]
Send: b [sender]
Send: c [sender]
Send a sync singal,[sender]
{}
Received a sync signal
Receive: a [receiver]
Send: d [sender]
wait 3 seconds [sender]
Receive: b [receiver]
Receive: c [receiver]
Receive: d [receiver]
Stopped,[receiver]
```

syncChan1h哦syncChan2的元素类型都是struct{}。struct{}代表的是不含含任何字段的结构体类型，也可称为空的结构体类型。在Go语言中，空结构体类型的变量是不占用内存空间的，并且所有该类型的变量都拥有相同的内存地址。

与接收操作类型，当向一个值为nil的通道类型的变量发送元素值时，当前goroutine也会被永久地阻塞。当试图向一个已关闭的通道发送元素值时，会panic。为了避免这样的流程中断，我们可以在select代码块中执行发送操作。

如果有多个goroutine因向同一个已满的通道发送元素值而被阻塞，那么当该通道中有多余空间的时候，最早被阻塞的那个goroutine会最先被唤醒。对于接收操作来说，也是如此。

发送方向通道发送的值会被复制，接收方接受的总是该值的副本，而不是该值本身。经由通道传递的值至少会被复制一次，至多会被复制两次。例如：当向一个已空的通道发送值，且已有至少一个接收方因此等待时，该通道会绕过本身的缓冲队列，直接把这个值复制给最早等待的那个接收方。又例如：当从一个已满的通道接收值，且已有至少一个发送方因此等待时，该通道会把缓冲队列中最早进入的那个值复制给接收方，再把最早等待发送方要发送的数据复制到那个值的原先位置上。通道的缓冲队列属于环形队列，所以这样是没问题的。这种情况下，涉及的那两个值在传递到接收方之前都会被复制两次。

因此，当接收方从通道接收到一个值类型的值时，对该值的修改就不会影响到发送方持有的那个源值。但对于引用类型来说，这种修改会同时影响发送双方持有的值。

```
package main

import (
	"fmt"
	"time"
)

var mapChan = make(chan map[string]int, 1)

func main() {
	syncChan := make(chan struct{}, 2)
	go func() {
		for {
			if elem, ok := <-mapChan; ok {
				elem["count"]++
			} else {
				break
			}
		}

		fmt.Println("Stopped. [receiver]")
		syncChan <- struct{}{}
	}()

	go func() {
		countMap := make(map[string]int)
		for i := 0; i < 5; i++ {
			mapChan <- countMap
			time.Sleep(1 * time.Second)
			fmt.Printf("countMap : %v \n", countMap)
		}
		close(mapChan)
		fmt.Println("Stopped. [sender]")
		syncChan <- struct{}{}
	}()

	<-syncChan
	<-syncChan

}
输出

countMap : map[count:1]
countMap : map[count:2]
countMap : map[count:3]
countMap : map[count:4]
countMap : map[count:5]
Stopped. [sender]
Stopped. [reeceiver]



例2:

package main

import (
	"fmt"
	"time"
)

type Counter struct {
	count int
}

var mapChan = make(chan map[string]*Counter, 1)

func main() {
	syncChan := make(chan struct{}, 2)
	go func() {
		for {
			if elem, ok := <-mapChan; ok {
				counter := elem["count"]
				counter.count++
			} else {
				break
			}
		}
		fmt.Println("Stopped. [receiver]")
		syncChan <- struct{}{}
	}()

	go func() {
		countMap := map[string]*Counter{
			"count": {},
		}
		for i := 0; i < 5; i++ {
			mapChan <- countMap
			time.Sleep(1 * time.Second)
			fmt.Printf("countMap : %v \n", countMap)
		}
		close(mapChan)
		fmt.Println("Stopped. [sender]")
		syncChan <- struct{}{}
	}()

	<-syncChan
	<-syncChan

}

// 实现打印接口,格式化输出
func (counter *Counter) String() string {
	return fmt.Sprintf("{count: %d}", counter.count)
}

//若不加*号(不传递地址，那么count的值是不会改变的)

输出
countMap : map[count:{count: 1}]
countMap : map[count:{count: 2}]
countMap : map[count:{count: 3}]
countMap : map[count:{count: 4}]
countMap : map[count:{count: 5}]
Stopped. [sender]
Stopped. [receiver]
```

###### select语句

selecdt语句是一种仅能用于通道发送和接收操作的专用语句。一条select语句执行时，会选择其中的某一个分之并执行。select语句中，每个分之以关键字case开始，与switch语句不同，跟在每个case后面的只能是针对某个通道的发送语句或接收语句。另外，select关键字的右边并没有像switch语句那样的switch表达式，而是直接后跟左花括号。

若同时有多个case语句满足条件，即发送或接收是ok的，会自动实现伪随机选择。同时，若没有个case满足条件，并且没有default case，会报错

```
接收操作符 "<-" 可以从一个通道接收一个元素值，也可通过 = 或 := 连接把操作结果赋给一个或两个变量，第二个变量会指明当前通道是否关闭或已无元素值。

package main

import (
	"fmt"
)

func main() {
	intChan := make(chan int, 10)
	for i := 0; i < 10; i++ {
		intChan <- i
	}

	close(intChan)

	syncChan := make(chan struct{}, 1)

	go func() {
	Loop:
		for {
			select {
			case e, ok := <-intChan:
				if !ok {
					fmt.Println("End.")
					break Loop
				}
				fmt.Printf("received: %v\n", e)
			}
		}
		syncChan <- struct{}{}

	}()
	<-syncChan
}
//
Loop 为标签的名字，意为中断紧贴于该标签之下的该语句的执行，类似于java中的label
```

###### 非缓冲的channel

- 向此类通道发送元素值的操作会被阻塞，直到至少有一个针对该通道的接收操作进行为止。该接收操作会先得到元素值的副本，然后在唤醒发送方所在的goroutine之后返回，也就是说，这个接收操作会在对应的发送操作完成之前完成
- 从此类通道接收元素值的操作会被阻塞，直到至少有一个针对该通道的发送操作进行为止。该发送操作会直接把元素值复制给接收方，然后再唤醒接收方所在的goroutine之后返回。也就是说，这时的发送操作会在对应的接收操作完成之前完成。

这两条规则都是源码级别的。

###### Timer

```
package main

import (
	"fmt"
	"time"
)

func main() {
	timer := time.NewTimer(2 * time.Second)

	fmt.Println(time.Now())

	expirationTime := <-timer.C

	fmt.Println(expirationTime)
}

// <- timer.C会一直阻塞

2019-08-26 17:54:59.232836 +0800 CST m=+0.000252932
2019-08-26 17:55:01.235835 +0800 CST m=+2.003237605
```

运用timer进行超时时间设置

```
package main

import (
	"fmt"
	"time"
)

func main() {
	intChan := make(chan int, 1)
	go func() {
		time.Sleep(2 * time.Second)
	}()

	select {
	case e := <-intChan:
		fmt.Println(e)
	case <-time.NewTimer(3 * time.Second).C:
		fmt.Println("超时")
	}
}

```

复用timer

```
package main

import (
	"fmt"
	"time"
)

func main() {
	intChan := make(chan int, 1)

	go func() {
		for i := 0; i < 5; i++ {
			time.Sleep(1 * time.Second)
			intChan <- i
		}
	}()

	timeOut := time.Millisecond * 500
	var timer *time.Timer

	for {
		if timer == nil {
			timer = time.NewTimer(timeOut)
		} else {
			timer.Reset(timeOut)
		}

		select {
		case e, ok := <-intChan:
			if !ok {
				fmt.Println("End.")
				return
			}
			fmt.Printf("Received: %v\n", e)
		case <-timer.C:
			fmt.Println("timeOut")

		}

	}
}
输出：
timeOut
Received: 0
timeOut
Received: 1
timeOut
timeOut
Received: 2
timeOut
Received: 3
timeOut
Received: 4
timeOut
...
```

###### Ticker

每隔一段时间执行一次

初始化

```
var ticker *time.Ticker = time.NewTicker(time.Second)
```
示例：

```
package main

import (
	"fmt"
	"time"
)

func main() {
	incChan := make(chan int, 1)
	ticker := time.NewTicker(time.Second)
	go func() {
		for range ticker.C {
			select {
			case incChan <- 1:
			case incChan <- 2:
			case incChan <- 3:
			}
		}
		fmt.Println("End. sender")
	}()

	var sum int
	for e := range incChan {
		fmt.Printf("Received :%v\n", e)
		sum += e
		if sum > 10 {
			fmt.Printf("Got: %v\n", sum)
			break
		}
	}
	fmt.Println("End. receiver")
}

输出
Received :2
Received :1
Received :2
Received :2
Received :1
Received :3
Got: 11
End. receiver
```