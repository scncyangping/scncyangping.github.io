---
layout:     post
title:      "Go并发编程 - (四)同步、锁"
subtitle:   ""
date:       2019-08-28 02:45:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

####  锁

##### sync.Mutex
##### 读写锁 sync.RWmutex

读写锁包含了四种基本方法

- func(*RWMutex) Lock()
- func(*RWMutex) UnLock()
- func(*RWMutex) RLock()
- func(*RWMutex) RUnLock()

读锁并不影响其他的读，但是会阻塞对应的写锁。写解锁会试图唤醒所有因欲进行读锁定而被阻塞的goroutine。而读解锁只会在已无任何读锁定的情况下，试图唤醒一个因欲进行写锁定而被阻塞的goroutine。若对未进行写锁定的读写锁进行解锁，会panic，读锁也一样。

```
package main

import (
	"fmt"
	"sync"
	"time"
)

func main()  {
	var rwm sync.RWMutex
	for i:=0;i<3;i++{
		go func(i int) {
			fmt.Printf("Try to lock for reading %d\n",i)
			rwm.RLock()
			fmt.Printf("Locked for reading. %d\n",i)
			time.Sleep(2 * time.Second)
			fmt.Printf("Try to unlock for reading %d\n",i)
			rwm.RUnlock()
			fmt.Printf("Unlocked for reading [%d]\n",i)
		}(i)
	}
	time.Sleep(time.Millisecond * 100)
	fmt.Println("Try to lock for writing...")
	rwm.Lock()
	fmt.Println("Locked for writing.")
}

输出
Try to lock for reading 0
Locked for reading. 0
Try to lock for reading 1
Locked for reading. 1
Try to lock for reading 2
Locked for reading. 2
Try to lock for writing...
Try to unlock for reading 1
Unlocked for reading [1]
Try to unlock for reading 0
Try to unlock for reading 2
Unlocked for reading [0]
Unlocked for reading [2]
Locked for writing.

当所有加的读锁都解锁了过后，对写锁进行加锁的操作才会执行
```

##### 条件变量

Go标准库中的sync.Cond类型代表了条件变量。无法直接声明，需要用方法创建

```
func NewCond(l Locker)*Cond
```
需要传入Locker的实现。比如一个互斥锁或者一个读写锁。

*sync.Cond类型的方法集合中有三个方法

- Wait 等待通知
- Signal 单发通知
- Broadcast 广播通知

Wait方法会自动对与该条件变量关联的那个锁进行解锁(这点很重要)。并且使它所在的goroutine阻塞。一旦几首到通知，该方法所在的goroutine就会被唤醒，并且该方法会立即尝试锁定该锁。方法Signal和Broadcast的作用都是发送通知，以唤醒正在为此阻塞的goroutine。


```
recond := sync.NewCond(sync.Mutex)
```

注意点

- 一定要在调用rcond的Wait方法之前锁定与之关联的读锁。
- 一定不要忘记在读取数据块之后及时解锁与条件变量rcond关联的那个读锁，否则对读写锁的写锁定操作将会阻塞相关的goroutine。其根本原因是，条件变量rcond的Wait方法在返回之前会重新锁定与之关联的那个读锁。


##### 原子操作

- atomic.LoadInt32(&value) 原子的读取某个数据

###### sync/atomic.Value

sync/atomic.Value是一个结构体类型。它用于存储需要原子读写的值。与sync/atomic包中其他函数不同，它可以接收被操作值的类型不限。

声明

```
var atomicVal atomic.Value
```
该类型有两个公开的指针方法--Load和store。前者用于原子地读取原子值实例中存储的值。会返回一个interface{}类型的结果且不接收任何参数。后者用于原子地在原子值实例中存储一个值。

对于store方法的限制

- 作为参数传入该方法的值不能为nil.
- 作为参数传入该方法的值必须与之前传入的值(如果有的话)类型相同。也就是说，一旦原子值实例存储了某一个类型的值，那么它之后存储的值就都必须是该类型的。

```
package main

import (
	"fmt"
	"sync/atomic"
)

func main()  {
	var countVal atomic.Value
	countVal.Store([]int{1,3,5,7})
	anotherStore(&countVal)
	fmt.Printf("The count value: %v",countVal.Load())
}

func anotherStore(countVal *atomic.Value)  {
	countVal.Store([]int{2,4,6,8})
}

// The count value: [2 4 6 8]

若不是指针类型会输出 [1,3,5,7]
```

###### sync.Once 执行一次

sync.Once也是开箱即用的

```
var once sync.Once
once.Do(func(){fmt.Println("Onece!")})
```
对同一个sync.Once类型值的Do方法的有效调用次数永远会是1。也就是说，无论调用这个方法多少次，也无论在多次调用时传递给它的参数值是否相同，都仅有第一次调用是有效的。

```
// 源码
func (o *Once) Do(f func()) {
    // 获取是否运行标记
	if atomic.LoadUint32(&o.done) == 1 {
		return
	}
	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

###### sync.WaitGroup

sync.WaitGroup类型的值是并发安全的，也是开箱即用的。该类型有3个指针方法，即：Add、Done、Wait(类似java的countDownlunch)

当调用sync.WaitGroup类型值的Wait方法时，它回去检查给定计数。如果该计数为0，那么该方法会立即返回，且不会对程序的运行产生影响。但是，如果这个计数大于0，该方法调用的那个goroutine就会阻塞，同时，等待计数会加1.直到该值的Add或Done方法被调用时，发现给定计数变回0，该值才会去唤醒因此而阻塞的所有goroutine.同时清零等待计数。

sync.WaitGroup使用规则

- 对一个sync.WaitGroup类型值的Add方法的第一次调用，发生在Done和Wait之前
- Add过后的值不能是负数，否则会报错
