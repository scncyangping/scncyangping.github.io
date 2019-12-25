---
layout:     post
title:      "数据结构(六)堆和优先队列"
subtitle:   ""
date:       2019-08-25 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---

### 优先队列

普通队列：先进先出、后进后出
优先队列：出队顺序和入队顺序无关；和优先级有关

操作系统中进行任务的调用就是优先队列，会动态的选择优先级最高的任务执行。

#### 优先队列经典问题

在N个元素中选出M个元素

使用优先队列，维护当前看到的前M的元素

使用最小堆，每次将根节点的那个最小的元素提出来

java 中的PriorityQueue默认实现就是最小堆

### 堆

#### 堆的基本结构

##### 最主流的实现方法 二叉堆

满二叉树：除了根节点和叶子节点，每个节点都有两个孩子

完全二叉树：不一定是满二叉树，但是缺少的那一部分一定在树的右下部分，同时，元素按照从左到右，一层一层的顺序的排列成树的形状

二叉堆的特点

1. 是一颗完全二叉树
2. 堆中某个节点的值总是不大于其父节点的值(最大堆，相应的可以定义出最小堆)

```
将元素按顺序一层一层的排列，可以使用数组来表示这一结构

                        62
                41              30
            28       16      22      13
         19    17  15

数组对应

0   1   2   3   4   5   6   7   8   9   10
-   62  41  30  28  16  22  13  19  17  15

此时，每一个节点在数组中对应的下表i会存在如下规律：

若从1 开始
parent(i) = i/2  整数除法
left child (i) = 2 * i
right child (i) = 2 * i + 1

若从0开始
parent(i) = (i-1)/2  整数除法
left child (i) = 2 * i + 1
right child (i) = 2 * i + 2

```

##### 实现

```
/*
@date : 2019/09/09
@author : YaPi
@desc :
*/
package heap

import "dtSt/common"

type MaxHeap struct {
	data		[]common.E
	size 		int
}

func NewMaxHeap() *MaxHeap {
	return &MaxHeap{data : make([]common.E,0)}
}

// 数组从0开始
// 父亲节点对应数组下标
func parent(n int)int  {
	if n == 0{
		return n
	}
	return (n-1)/2
}
// 左孩子节点对应数组下标
func leftChild(n int)int{
	return 2 * n +1
}
// 右孩子节点对应数组下标
func rightChild(n int)int{
	return 2 * n +2
}

func (m *MaxHeap)get(i int) common.E {
	if i < 0 || i > len(m.data){
		return nil
	}
	return m.data[i]
}

func (m *MaxHeap)swap(i ,j int){
	if i < 0 || j < 0 || i >= len(m.data) || j >= len(m.data){
		return
	}
	m.data[i],m.data[j] = m.data[j],m.data[i]
}

func (m *MaxHeap)Add(e common.E)  {
	// 添加元素到末尾
	m.data = append(m.data, e)
	// 对数组进行shiftUp操作
	m.shiftUp(len(m.data) - 1)
	m.size ++
}

// 向上重新组装二叉堆
func (m *MaxHeap)shiftUp(i int)  {
	for i >0 && m.get(i).CompareTo(m.get(parent(i))) == 1 {
		m.swap(i,parent(i))
		i = parent(i)
	}
}

func (m *MaxHeap)findMax()common.E  {
	if len(m.data) == 0{
		return nil
	}
	return m.data[0]
}

func (m * MaxHeap)ExtractMax()common.E{
	e := m.findMax()
	if e == nil {
		return e
	}
	// 将最后一个元素的值设置到最大值的位置
	m.swap(0,len(m.data)-1)
	// 将最大值删除
	m.data = m.data[:len(m.data)-1]
	// 从根节点执行down操作
	m.shiftDown(0)
	m.size --
	return e
}

func (m *MaxHeap)shiftDown(k int)  {
	for leftChild(k) < len(m.data){
		// 设置j的值为左孩子对应的数组下标值
		j := leftChild(k)
		// j + 1 是右孩子的数组中的下标
		// 如果右孩子对应的值大于左孩子对应的值，设置j的值为右孩子的下标
		if j + 1 < len(m.data) && m.get(j + 1).CompareTo(m.get(j)) == 1{
				j ++
		}
		// 判断当前节点的值是不是比j对应的孩子节点大，如果大于或等于，就跳过，否则就替换
		if m.get(k).CompareTo(m.get(j)) >= 0{
			break
		}

		m.swap(k,j)
		k = j
	}
}
```

Test

```
/*
@date : 2019/09/10
@author : YaPi
@desc :
*/
package heap

import (
	"dtSt/common"
	"fmt"
	"math/rand"
	"testing"
)

func TestMaxHeap(t *testing.T) {
	maxHeap := NewMaxHeap()

	for i:=0;i<10;i++{
		t := rand.Intn(1000)
		fmt.Println(t)
		maxHeap.Add(common.NewInt(t))
	}
	fmt.Println()
	test := make([]common.E,0)
	for i:=0;i<10;i++{
		test = append(test, maxHeap.ExtractMax())
	}

	for _,v := range test{
		fmt.Println(v)
	}
	// output
	//		81
	//		887
	//		847
	//		59
	//		81
	//		318
	//		425
	//		540
	//		456
	//		300
	//
	//		887
	//		847
	//		540
	//		456
	//		425
	//		318
	//		300
	//		81
	//		81
	//		59

}

```