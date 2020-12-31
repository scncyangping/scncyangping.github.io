---
layout:     post
title:      "数据结构(二)时间复杂度分析+自定义数组"
subtitle:   ""
date:       2019-08-21 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---

#### 简单的时间复杂度分析

- O(1)、 O(n)、 O(lgn)、 O(nlogn)、
- 大O描述的是算法的运行时间和输入数据之间的关系

例：

```
public static int sum(int[] nums){
    int sum = 0;
    for(int num : nums)sum += num;
    return sum;
}
```
上述的时间复杂度是 O(n)的，n是nums中的元素个数。上述这个程序运行时间的多少是和元素个数n成线性关系的。即n数字越大时间就越长。

为什么要用大O，叫做O(n)?

忽略常数。线性方程是 T = c1*n + c2。其中c1 * n 代表for循环，即主要运行时间。c2代表读取数据、返回数据等额外开销。

所以，并不是所有的O(n)的算法会快于O(n2)的算法。这个和n有很大的关系。

#### 均摊复杂度

以数组扩容来说，一个添加操作，当数据量达到了数组容量的时候就会扩容为原来的二倍。比如，初始容量为10，当达到10的时候扩容一倍，容量变为20，当继续添加到20时，才会又扩容。扩容的时间复杂度为O(n)，每一次添加的时间复杂度为O(1)。相当于，添加 n次，会有1次扩容。将这一次操作的耗时均摊到所有添加操作中，那么这个复杂度就变成了n(1)了

##### 复杂度的震荡

以数组扩容来说，当数组容量达到规定容量过后，需要扩容，当数组中减少一个元素时，需要缩容，若，存在一种情况: 每添加一次元素后，立马删除一个元素，那么就会不断的扩容、缩容、扩容、缩容。这样时间复杂度就会一直会O(n)，而不能均摊成O(1)。这种情况下就需要对缩容方式进行优化，当缩小到容量减少为原数组容量为四分之一时，再进行所容。

#### 自定义数组实现

```
/*
@date : 2019/08/30
@author : YaPi
@desc :
*/
package array

import "strconv"

// 定义传入数组类型
// type E interface {}

// 自定义数组类型需满足的方法
type E interface {
	// 0 相等 1 大于 -1 小于
	CompareTo(e E)int
	String()string
}

// 自定义数组结构
type sArray struct {
	size 		int
	data 		[]E
}

func (s *sArray) String() string {
	str := "数组长度 : "+ strconv.Itoa(s.size) + " 【 ";
	for i:=0;i<len(s.data);i++{
		str += s.data[i].String() + "  "
	}
	str += " 】"
	return str
}

func NewSArray() *sArray {
	return &sArray{}
}


// 自定义数组实现的方法
type SArrayInterface interface {
	// 添加
	Add(e E)
	// 删除指定位置的元素
	Remove(i int)E
	// 数组容量(不是数组中元素的总数)
	Capacity()int
	// 数组中元素的总数
	Size()int
	// 查找元素E在数组中的位置
	Find(e E)int
	// 数组最后一个位置添加元素
	AddLast(e E)
	// 数组头添加元素
	AddFirst(e E)
	// 获取指定位置的元素
	Get(i int)E
	// 设置指定位置的元素
	Set(i int,e E)
	// 删除第一个元素
	RemoveFirst()
	// 删除最后一个元素
	RemoveLast()
	// 删除找到的第一个元素
	RemoveElement(e E)
}


func (s *sArray) Add(e E) {
	s.data = append(s.data, e)
	s.size ++
}

func (s *sArray) Remove(i int) E {
	if i < 0 || i >= s.size {
		return nil
	}
	e := s.data[i]
	s.data = append(s.data[:i], s.data[i+1:]...)
	s.size--
	return e
}

func (s *sArray) Capacity()int {
	return cap(s.data)
}

func (s *sArray) Size()int {
	return s.size
}

func (s *sArray) Find(e E) int {
	index := -1
	for i,d := range  s.data{
		if d.CompareTo(e) == 0{
			index = i
			break
		}
	}
	return index
}

func (s *sArray) AddLast(e E) {
	s.data = append(s.data, e)
	s.size ++
}

func (s *sArray) AddFirst(e E) {
	lNode := s.data[s.size-1]
	for i:=s.size-2;i>=0;i--{
		s.data[i+1] = s.data[i]
	}
	s.data[0] = e
	s.size ++
	s.data = append(s.data, lNode)
}

func (s *sArray) Get(i int) E {
	if i < 0 || i >= s.size {
		return nil
	}
	return s.data[i]
}

func (s *sArray) Set(i int, e E) {
	if i < 0 || i >= s.size {
		return
	}
	s.data[i] = e
}

func (s *sArray) RemoveFirst() {
	s.Remove(0)
}

func (s *sArray) RemoveLast() {
	s.Remove(s.size-1)
}

func (s *sArray) RemoveElement(e E) {
	index := s.Find(e)
	if index != -1{
		s.Remove(index)
	}
}

```

测试

```
type NewInt int

func (ee NewInt) String() string {
	return strconv.Itoa(int(ee))
}

func (ee NewInt) CompareTo(e E) int {

	eeInt := int(ee)

	eInt := int(e.(NewInt))

	if eeInt > eInt {
		return 1
	}else if eeInt == eInt{
		return 0
	}else {
		return -1
	}
	return 0
}

func TestSArray(t *testing.T) {
	a := NewSArray()
	for i:=0;i<5;i++{
		newInt := NewInt(rand.Intn(100))
		a.Add(newInt)
	}
	fmt.Println(a)
	fmt.Println("添加元素1")
	a.Add(NewInt(1))
	fmt.Println(a)
	fmt.Println("删除第二个元素")
	a.Remove(2)
	fmt.Println(a)
	fmt.Println("查找元素为81的元素: ",a.Find(NewInt(81)))
	fmt.Println("获取第二个元素: ",a.Get(2))
	fmt.Println("数组头添加一个元素111")
	a.AddFirst(NewInt(111))
	fmt.Println(a)
	fmt.Println("数组尾添加一个元素123")
	a.AddLast(NewInt(123))
	fmt.Println(a)
	fmt.Println("删除第一个元素")
	a.RemoveFirst()
	fmt.Println(a)
	fmt.Println("删除最后一个元素")
	a.RemoveLast()
	fmt.Println(a)
	fmt.Println("删除指定为1的元素")
	a.RemoveElement(NewInt(1))
	fmt.Println(a)
}
```