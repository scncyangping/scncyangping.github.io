---
layout:     post
title:      "Golang标准库-sort、container"
subtitle:   ""
date:       2019-08-13 02:40:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### sort包

该包实现了四种基本排序算法：插⼊排序、归并排序、堆排序和快速排序.但是这四
种排序⽅法是不公开的，它们只被⽤于sort包内部使⽤。所以在对数据集合排序时不必考虑应当选择哪⼀种排序⽅法，只要实现了sort.Interface定义的三个⽅法：

- 获取数据集合⻓度的Len()⽅法
- ⽐较两个元素⼤⼩的Less()⽅法
- 交换两个元素位置的Swap()⽅法

就可以顺利对数据集合进⾏排序。sort包会根据实际数据⾃动选择⾼效的排序算法。 除此之外，为了⽅便对常⽤数据类型的操作，sort包提供了对[]int切⽚、
[]float64切⽚和[]string切⽚完整⽀持，主要包括:

- 对基本数据类型切⽚的排序⽀持
- 基本数据元素查找
- 判断基本数据类型切⽚是否已经排好序
- 对排好序的数据集合逆序

sort.Interface接⼝:

```
type Interface interface {
    // 获取数据集合元素个数
    Len() int
    // 如果i索引的数据⼩于j所以的数据，返回true，不会调⽤
    // 下⾯的Swap()，即数据升序排序。
    Less(i, j int) bool
    // 交换i和j索引的两个元素的位置
    Swap(i, j int)
}
```

数据集合实现了这三个⽅法后，即可调⽤该包的Sort()⽅法进⾏排序,Sort()⽅法惟⼀的参数就是待排序的数据集合

```
func Sort(data Interface)
```

###### 示例

```
package main

import (
	"fmt"
	"sort"
)

type Stu struct {
	Name string
	Age  int
}

type Stus []Stu

func (s Stus) Len() int {
	return len(s)
}

func (s Stus) Less(i, j int) bool {
	return s[i].Age > s[j].Age
}

func (s Stus) Swap(i, j int) {
	s[i], s[j] = s[j], s[i]
}

func main() {
	stuss := Stus{
		{"张三", 12},
		{"里斯", 23},
		{"王五", 5},
	}

	for _, v := range stuss {
		fmt.Println(v.Age, " : ", v.Name)
	}

	fmt.Println("----------")

	sort.Sort(stuss)

	for _, v := range stuss {
		fmt.Println(v.Age, " : ", v.Name)
	}

	fmt.Println("Is Sorted ?", sort.IsSorted(stuss))
}

```
######  Reverse()

若需要降序排列,可以更改Less()中的方法,sort包也提供了Reverse()方法,可以将其倒叙排序

到Reverse()返回的⼀个sort.Interface接⼝类型

```
//定义了⼀个reverse结构类型，嵌⼊Interface接⼝
type reverse struct {
    Interface
}
//reverse结构类型的Less()⽅法拥有嵌⼊的Less()⽅法相反的⾏为
//Len()和Swap()⽅法则会保持嵌⼊类型的⽅法⾏为
func (r reverse) Less(i, j int) bool {
    return r.Interface.Less(j, i)
}
//返回新的实现Interface接⼝的数据类型
func Reverse(data Interface) Interface {
    return &reverse{data}
}
```

###### Search(()

> Search()⽅法回使⽤“⼆分查找”算法来搜索某指定切⽚[0:n]，并返回能够使
f(i)=true的最 ⼩的i（0<=i<n）值，并且会假定，如果f(i)=true，则f(i+1)=true，即对于切⽚[0:n]，i之前的切⽚元素会使f()函数返回false，i及i之后的元素会使f()函数返回true。但是，当在切⽚中⽆法找到时f(i)=true的i时（此时切⽚元素都不能使f()函数返回true），Search() ⽅法会返回n

Search()函数⼀个常⽤的使⽤⽅式是搜索元素x是否在已经升序排好的切⽚s中

```
x := 11
s := []int{3, 6, 8, 11, 45} //注意已经升序排序
pos := sort.Search(len(s), func(i int) bool { return s[i] >= x })
if pos < len(s) && s[pos] == x {
    fmt.Println(x, "在s中的位置为：", pos)
} else {
    fmt.Println("s不包含元素", x)
}
```

##### 原声支持方法

sort包原生支持[]int,[]float64,[]string三种内建数据类型切片的排序操作,即不必我们自己实现相关的Len()、Less()、Swap()方法

- sort.Ints()

```
s := []int{5, 2, 6, 3, 1, 4} // 未排序的切⽚数据
sort.Ints(s)
fmt.Println(s) //将会输出[1 2 3 4 5 6]

// 若需要降序排列
s := []int{5, 2, 6, 3, 1, 4} // 未排序的切⽚数据
sort.Sort(sort.Reverse(sort.IntSlice(s)))
fmt.Println(s) //将会输出[6 5 4 3 2 1]
```

- Float64Slice类型及[]float64排序

与Sort()、IsSorted()、Search()相对应的三个⽅法

```
func Float64s(a []float64)
func Float64sAreSorted(a []float64) bool
func SearchFloat64s(a []float64, x float64) int
```

- StringSlice类型及[]string排序

与Sort()、IsSorted()、Search()相对应的三个⽅法

```
func Strings(a []string)
func StringsAreSorted(a []string) bool
func SearchStrings(a []string, x string) int
```

####  container

该包实现了三个复杂的数据结构:堆(heap),链表(list),环(ring)。


##### heap(堆)

堆使⽤的数据结构是最⼩⼆叉树，即根节点⽐左边⼦树和右边⼦树的所有值都
⼩。 go的堆包只是实现了⼀个接⼝

```
type Interface interface {
    sort.Interface
    Push(x interface{}) // add x as element Len()
    Pop() interface{} // remove and return element Len() - 1.
}
```

示例

```
// 定义实现了相关接口的类型
type IntHeap []int
func (h IntHeap) Len() int { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int) { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}
func (h *IntHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

// 调用
h := &IntHeap{2, 1, 5}
heap.Init(h)
heap.Push(h, 3)
heap.Pop(h)
```
##### list(链表)

链表就是⼀个有prev和next指针的数组了

```

// list 包定义

type Element struct {
    next, prev *Element // 上⼀个元素和下⼀个元素
    list *List // 元素所在链表
    Value interface{} // 元素
}

type List struct {
    root Element // 链表的根元素
    len int // 链表的⻓度
}
```


示例

```
list := list.New()
list.PushBack(1)
list.PushBack(2)
fmt.Printf("len: %v\n", list.Len());
fmt.Printf("first: %#v\n", list.Front());
fmt.Printf("second: %#v\n", list.Front().Next());
```

list对应相关方法

```
type Element
    func (e *Element) Next() *Element
    func (e *Element) Prev() *Element
type List
    func New() *List
    func (l *List) Back() *Element // 最后⼀个元素
    func (l *List) Front() *Element // 第⼀个元素
    func (l *List) Init() *List // 链表初始化
    func (l *List) InsertAfter(v interface{}, mark *Element) *Element // 在某个元素前插
    ⼊
    func (l *List) InsertBefore(v interface{}, mark *Element) *Element // 在某个元素后
    插⼊
    func (l *List) Len() int // 在链表⻓度
    func (l *List) MoveAfter(e, mark *Element) // 把e元素移动到mark之后
    func (l *List) MoveBefore(e, mark *Element) // 把e元素移动到mark之前
    func (l *List) MoveToBack(e *Element) // 把e元素移动到队列最后
    func (l *List) MoveToFront(e *Element) // 把e元素移动到队列最头部
    func (l *List) PushBack(v interface{}) *Element // 在队列最后插⼊元素
    func (l *List) PushBackList(other *List) // 在队列最后插⼊接上新队列
    func (l *List) PushFront(v interface{}) *Element // 在队列头部插⼊元素
    func (l *List) PushFrontList(other *List) // 在队列头部插⼊接上新队列
    func (l *List) Remove(e *Element) interface{} // 删除某个元素
```

##### 环(ring)

环的结构有点特殊，环的尾部就是头部，所以每个元素实际上就可以代表⾃身的这个环.它不需要像list⼀样保持list和element两个结构，只需要保持⼀个结构就⾏

```
type Ring struct {
    next, prev *Ring
    Value interface{}
}
```

我们初始化环的时候，需要定义好环的⼤⼩，然后对环的每个元素进⾏赋值。环还提供⼀个Do⽅法，能便利⼀遍环，对每个元素执⾏⼀个function

示例

```
package main

import (
	"container/ring"
	"fmt"
)

func main() {
	ring := ring.New(3)
	for i := 1; i <= 3; i++ {
		ring.Value = i
		ring = ring.Next()
	}
	// 计算1+2+3
	s := 0
	ring.Do(func(p interface{}) {
		s += p.(int)
	})

	fmt.Println("sum is", s)

}

```

ring提供的⽅法

```
type Ring
    func New(n int) *Ring // 初始化环
    func (r *Ring) Do(f func(interface{})) // 循环环进⾏操作
    func (r *Ring) Len() int // 环⻓度
    func (r *Ring) Link(s *Ring) *Ring // 连接两个环
    func (r *Ring) Move(n int) *Ring // 指针从当前元素开始向后移动或者向前（n可以为负数）
    func (r *Ring) Next() *Ring // 当前元素的下个元素
    func (r *Ring) Prev() *Ring // 当前元素的上个元素
    func (r *Ring) Unlink(n int) *Ring // 从当前元素开始，删除n个元素
```