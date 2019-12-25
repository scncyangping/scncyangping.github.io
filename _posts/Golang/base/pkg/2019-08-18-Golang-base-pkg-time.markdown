---
layout:     post
title:      "Go标准库-time"
subtitle:   ""
date:       2019-08-18 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### Time

Time 代表一个纳秒精度的时间点。程序中应使用Time类型值来保存和传递时间,而不是指针。 就是说,表示时间的变量和字段应为time.Time类型,而不是*time.Time类型。

一个Time类型的值可以被多个go程同时使用。时间点可以使用Before、After、Equal方法进行比较。Sub方法让两个时间点相减,生成一个Duration类型值(代表时间段)。Add方法给一个时间点加上一个时间段,生成一个新的Time类型时间点。

每⼀个时间都具有⼀个地点信息（及对应地点的时区信息），当计算时间的表示格式时，如Format、Hour和Year等⽅法，都会考虑该信息。Local、UTC和In⽅法返回⼀
个指定时区（但指向同⼀时间点）的Time。修改地点/时区信息只是会改变其表示；不会修改被表示的时间点，因此也不会影响其计算。通过 == ⽐较 Time 时，Location 信息也会参与⽐较，因此 Time 不应该作为 map 的key

```
type Time struct {
	// wall and ext encode the wall time seconds, wall time nanoseconds,
	// and optional monotonic clock reading in nanoseconds.
	//
	// From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
	// a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
	// The nanoseconds field is in the range [0, 999999999].
	// If the hasMonotonic bit is 0, then the 33-bit field must be zero
	// and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
	// If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
	// unsigned wall seconds since Jan 1 year 1885, and ext holds a
	// signed 64-bit monotonic clock reading, nanoseconds since process start.
	wall uint64
	ext  int64

	// loc specifies the Location that should be used to
	// determine the minute, hour, month, day, and year
	// that correspond to this Time.
	// The nil location means UTC.
	// All UTC times are represented with loc==nil, never loc==&utcLoc.
	loc *Location
}
```

##### 与 Unix 时间戳的转换

- time.Unix(sec, nsec int64) 通过 Unix 时间戳⽣成 time.Time 实例
- time.Time.Unix() 得到 Unix 时间戳
- time.Time.UnixNano() 得到 Unix 时间戳的纳秒表示

```
// 传入秒数 + 纳秒数 --> 返回二者之和代表的时间
fmt.Println(time.Unix(1565858822, 555))
输出: 2019-08-15 16:47:02.000000555 +0800 CST


fmt.Println(time.Now().Unix())
输出: 1565859300


fmt.Println(time.Now().UnixNano())
输出: 1565859300949946000
```

##### 格式化和解析

- time.Parse 和 time.ParseInLocation
- time.Time.Format

time.Now()获取的时间时区是Local,而time.Parse解析出来的时区是UTC,所以一般不用time.Parse,而是用time.ParseInLocation

```
// Local
fmt.Println(time.Now().Location().String())

t, _ := time.Parse("2006-01-02 15:04:05", "2019-08-15 14:14:00")
// UTC
fmt.Println(t.Location().String())

// -4.979531593611111 时区不对,所以错了
fmt.Println(time.Now().Sub(t).Hours())

t2, _ := time.ParseInLocation("2006-01-02 15:04:05", "2019-08-15 14:14:00", time.Local)
// Local
fmt.Println(t2.Location().String())

// 3.020468415
fmt.Println(time.Now().Sub(t2).Hours())

//
fmt.Println(time.Now().Format("2006-01-02 15:04:05"))

```

##### Round 和 Truncate ⽅法

⽐如，有这么个需求：获取当前时间整点的Time实例。例如，当前时间是15:54:23，需要的是15:00:00。我们可以这么做:

```
t, _ := time.ParseInLocation("2006-01-02 15:04:05", time.Now().Format("2006-01-02 15:0
0:00"), time.Local)

fmt.Println(t)
```

time 包给我们提供了专⻔的⽅法，功能更强⼤，性能也更好，这就是Round和Trunate它们区别，⼀个是取最接近的，⼀个是向下取整

```
t, _ := time.ParseInLocation("2006-01-02 15:04:05", "2016-06-13 15:34:39", time.Local)
// 整点（向下取整）
fmt.Println(t.Truncate(1 * time.Hour))

// 整点（最接近）
fmt.Println(t.Round(1 * time.Hour))

// 整分（向下取整）
fmt.Println(t.Truncate(1 * time.Minute))

// 整分（最接近）
fmt.Println(t.Round(1 * time.Minute))
t2, _ := time.ParseInLocation("2006-01-02 15:04:05", t.Format("2006-01-02 15:00:00"),
time.Local)
fmt.Println(t2)
```

##### 定时器

- Timer (到达指定时间触发且只触发⼀次)
- Ticker (间隔特定时间触发)

###### Timer

Timer 类型代表单次时间事件。当 Timer 到期时，当时的时间会被发送给 C (channel)，除⾮Timer 是被 AfterFunc 函数创建的

```
type Timer struct {
    C <-chan Time // The channel on which the time is delivered.
    r runtimeTimer
}
```

runtimeTimer

它定义在sleep.go⽂件中，必须和runtime包中time.go⽂件中的timer必须保持⼀致

```
type timer struct {
    i int // heap index
    // Timer wakes up at when, and then at when+period, ... (period > 0 only)
    // each time calling f(now, arg) in the timer goroutine, so f must be
    // a well-behaved function and not block.
    when int64
    period int64
    f func(interface{}, uintptr)
    arg interface{}
    seq uintptr
}
```

创建

```
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
    C: c,
    r: runtimeTimer{
    when: when(d),
    f: sendTime,
    arg: c,
    },
    }
    startTimer(&t.r)
    return t
}

```

在 when 表示的时间到时，会往 Timer.C中发送当前时间。when表示的时间是纳秒
时间,正常通过 runtimeNano() + int64(d) 赋值。跟上⼀节中讲到的 now() 类
似,runtimeNano() 也在 runtime 中实现（ runtime·nanotime ）

- 调⽤系统调⽤ clock_gettime获取时钟值（这是POSIX时钟）。其中clockid_t时钟类型是 CLOCK_MONOTONIC，也就是不可设定的恒定态时钟。具体的是什么时间，SUSv3 规定始于未予规范的过去某⼀点，Linux 上，始于系统启动。
- 如果 clock_gettime 不存在，则使⽤精度差些的系统调⽤ gettimeofday


定时器的具体实现逻辑，都在runtime中的time.go中，它的实现，没有采⽤经典Unix间隔定时器setitimer系统调⽤，也没有采⽤POSIX间隔式定时器（相关系统调⽤：timer_create、timer_settime和timer_delete），⽽是通过四叉树堆(heep)实现的（runtimeTimer结构中的i字段，表示在堆中的索引）。通过构建⼀个最⼩堆，保证最快拿到到期了的定时器执⾏。定时器的执⾏，在专⻔的goroutine中进⾏的：gotimerproc()。有兴趣的同学，可以阅读runtime/time.go的源码

###### Timer 相关函数或⽅法的使⽤

通过 time.After 模拟超时

```
c := make(chan int)
go func() {
    // time.Sleep(1 * time.Second)
    time.Sleep(3 * time.Second)
    <-c
}()

select {
case c <- 1:
    fmt.Println("channel...")
case <-time.After(2 * time.Second):
    close(c)
    fmt.Println("timeout...")
}

输出 timeout...
```

###### time.Stop 停⽌定时器 或 time.Reset 重置定时器

```
start := time.Now()
timer := time.AfterFunc(2*time.Second, func() {
    fmt.Println("after func callback, elaspe:", time.Now().Sub(start))
})
time.Sleep(1 * time.Second)
// time.Sleep(3*time.Second)
// Reset 在 Timer 还未触发时返回 true；触发了或Stop了，返回false
if timer.Reset(3 * time.Second) {
    fmt.Println("timer has not trigger!")
} else {
    fmt.Println("timer had expired or stop!")
}
time.Sleep(10 * time.Second)
// output:
// timer has not trigger!
// after func callback, elaspe: 4.00026461s

```

###### sleep内部实现

查看runtime/time.go⽂件中的timeSleep可知，Sleep的是通过Timer实现的，把当前goroutine作为arg参数（getg())。

如果定时器还未触发，Stop会将其移除，并返回true；否则返回false；后续再对该Timer调⽤Stop，直接返回false。Reset会先调⽤stopTimer再调⽤startTimer，类似于废弃之前的定时器，重新启动⼀个定时器。返回值和Stop⼀样。

##### Ticker 相关函数或⽅法的使⽤

Ticker和Timer类似，区别是：Ticker中的runtimeTimer字段的period字段会赋值为NewTicker(dDuration)中的d，表示每间隔d纳秒，定时器就会触发⼀次。除⾮程序终⽌前定时器⼀直需要触发，否则，不需要时应该调⽤Ticker.Stop来释放相关资源。如果程序终⽌前需要定时器⼀直触发，可以使⽤更简单⽅便的time.Tick函数，因为Ticker实例隐藏起来了，因此，该函数启动的定时器⽆法停⽌。