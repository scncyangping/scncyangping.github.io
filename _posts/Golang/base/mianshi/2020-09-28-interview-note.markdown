---
layout:     post
title:      "Go小记 - 知识点"
subtitle:   ""
date:       2020-09-28 02:40:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### Go中的锁
Go中的三种锁包括:互斥锁,读写锁,sync.Map的安全的锁.

- 互斥锁

Go并发程序对共享资源进行访问控制的主要手段，由标准库代码包中sync中的Mutex结构体表示。
```
//Mutex 是互斥锁， 零值是解锁的互斥锁， 首次使用后不得复制互斥锁。
type Mutex struct {
   state int32
   sema  uint32
}
```

sync.Mutex包中的类型只有两个公开的指针方法Lock和Unlock。
```
// Locker表示可以锁定和解锁的对象。
type Locker interface {
   Lock()
   Unlock()
}

// 锁定当前的互斥量
// 如果锁已被使用，则调用goroutine
// 阻塞直到互斥锁可用。
func (m *Mutex) Lock()

// 对当前互斥量进行解锁
// 如果在进入解锁时未锁定m，则为运行时错误。
// 锁定的互斥锁与特定的goroutine无关。
// 允许一个goroutine锁定Mutex然后安排另一个goroutine来解锁它。
func (m *Mutex) Unlock()
```

互斥锁对他进行多次加锁或解锁操作的时候会发生panic


- 读写锁

读写锁是针对读写操作的互斥锁，可以分别针对读操作与写操作进行锁定和解锁操作

1. 多个写操作之间是互斥的
2. 写操作与读操作之间也是互斥的
3. 多个读操作之间不是互斥的

```
/ RWMutex是一个读/写互斥锁，可以由任意数量的读操作或单个写操作持有。
// RWMutex的零值是未锁定的互斥锁。
// 首次使用后，不得复制RWMutex。
// 如果goroutine持有RWMutex进行读取而另一个goroutine可能会调用Lock，那么在释放初始读锁之前，goroutine不应该期望能够获取读锁定。
// 特别是，这种禁止递归读锁定。 这是为了确保锁最终变得可用; 阻止的锁定会阻止新读操作获取锁定。
type RWMutex struct {
   w           Mutex  //如果有待处理的写操作就持有
   writerSem   uint32 // 写操作等待读操作完成的信号量
   readerSem   uint32 //读操作等待写操作完成的信号量
   readerCount int32  // 待处理的读操作数量
   readerWait  int32  // number of departing readers
}

//对读操作的锁定
func (rw *RWMutex) RLock()
//对读操作的解锁
func (rw *RWMutex) RUnlock()
//对写操作的锁定
func (rw *RWMutex) Lock()
//对写操作的解锁
func (rw *RWMutex) Unlock()

//返回一个实现了sync.Locker接口类型的值，实际上是回调rw.RLock and rw.RUnlock.
func (rw *RWMutex) RLocker() Locker

```

Unlock方法会试图唤醒所有想进行读锁定而被阻塞的协程，而 RUnlock方法只会在已无任何读锁定的情况下，试图唤醒一个因欲进行写锁定而被阻塞的协程。若对一个未被写锁定的读写锁进行写解锁，就会引发一个不可恢复的panic，同理对一个未被读锁定的读写锁进行读写锁也会如此。

由于读写锁控制下的多个读操作之间不是互斥的，因此对于读解锁更容易被忽视。对于同一个读写锁，添加多少个读锁定，就必要有等量的读解锁，这样才能其他协程有机会进行操作。

