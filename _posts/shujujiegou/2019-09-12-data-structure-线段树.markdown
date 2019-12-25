---
layout:     post
title:      "数据结构(七)线段树"
subtitle:   ""
date:       2019-08-25 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---


### 线段树

###### 平衡树

平衡树，即平衡二叉树（Balanced Binary Tree），具有以下性质：它是一棵空树或它的左右两个子树的高度差的绝对值不超过1，并且左右两个子树都是一棵平衡二叉树。
平衡二叉树的常用算法有红黑树、AVL、Treap、伸展树、SBT等


线段树不是一个完全二叉树，但是是一颗平衡二叉树，如果把最下层空节点看成空的节点，那么它也是一颗满二叉树。

对于一颗满二叉树来说有以下特点
- h层，一共有2^h - 1个节点(大约是2^h)
- 最后一层 h-1层，有2^(h-1)个节点
- 最后一层的节点数大致等于前面所有层节点之和

若我们想用一个数组去存储线段数，如果区间有n个元素，那么，如果线段树的节点数量n是2的整数倍，因为最后一层的节点数等于最后一层节点数减1，所以，就需要2n的数组空间才能存储下来，在最坏的情况下，如果n的数量为基数，那么就需要新加一层才能存储下来，所以需要4n的空间。

我们的线段树不考虑添加元素，即区间固定，使用4n的静态空间即可。会存在浪费


对于给定的区间支持相应的两种操作

- 更新: 更新区间种的一个元素或者一个区间的值
- 查询一个区间[i,j]的最大值，最小值或者区间数字和

不考虑添加元素和删除元素


#### 实现

```
/*
@date : 2019/09/11
@author : YaPi
@desc :
*/
package tree

import "dtSt/common"

type Segment interface {
	// 输入两个节点a,b，根据业务需求返回合适的节点，如：
	// 若线段树是用于存储一段区间的最大值，那么就应该返回阿a 和 b中的最大值
	// 若线段树是用于存储一段区间的最小值，那么就应该返回阿a 和 b中的最小值
	// 根据具体的结果返回
	Merge(b Segment)Segment
	String()string
}


type SegmentTree struct {
	data		[]common.Segment
	tree 		[]common.Segment
}

func (s *SegmentTree) String() string {
	ss := "[ "
	for _,v := range s.tree{
		if v != nil {
			ss += " "+ v.String()
		}
	}
	ss += " ]"
	return ss
}

func NewSegmentTree(d []common.Segment) *SegmentTree {
	// 需要 4n的元素才能存储完长度为len的数组所生成的线段树
	tre := &SegmentTree{
		data: d,
		tree: make([]common.Segment,len(d) * 4)}

	tre.buildSegmentTree(0,0,len(d) - 1)
	return tre
}

func (s *SegmentTree)IsEmpty()bool{
	return len(s.data) == 0
}

func (s *SegmentTree)GetSize()int{
	return len(s.data)
}

func (s *SegmentTree)Get(index int)common.Segment{
	if index < 0 || index >= len(s.data){
		return nil
	}
	return s.data[index]
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

// 构建线段树
// 在位置为i的地方构建区间为[l,r]的线段树
func (s *SegmentTree)buildSegmentTree(i,l,r int)  {
	// 说明已经不能再分了，就是一个单元素了
	if l == r{
		s.tree[i] = s.data[l]
		return
	}
	leftTressIndex := leftChild(i)
	rightTreeIndex := rightChild(i)
	// 不使用 (l + r) /2 避免越界
	mid := l + (r-l) / 2
	s.buildSegmentTree(leftTressIndex,l,mid)
	s.buildSegmentTree(rightTreeIndex,mid+1,r)

	s.tree[i] = s.tree[leftTressIndex].Merge(s.tree[rightTreeIndex])
}

func (s *SegmentTree)query(queryL,queryR int)common.Segment  {
	if queryL < 0 || queryR < 0 || queryL >=len(s.data) || queryR >= len(s.data){
		return nil
	}

	return s.queryInTree(0,0,len(s.data)-1,queryL,queryR)
}

// 在以treeIndex为根的，范围为l到r的线段树中查找区间为queryL,到queryR的数据
func (s *SegmentTree)queryInTree(treeIndex,l,r,queryL,queryR int)common.Segment  {
	if l == queryL && r == queryR{
		return s.tree[treeIndex]
	}
	mid := l + (r -l)/2
	leftTressIndex := leftChild(treeIndex)
	rightTreeIndex := rightChild(treeIndex)

	// 需要查找的和左子树没关系
	if queryL >= mid +1 {
		return s.queryInTree(rightTreeIndex,mid + 1,r,queryL,queryR)
	}else if queryR <= mid {
		// 右子树没关系 在左孩子中查找
		return s.queryInTree(leftTressIndex,l,mid,queryL,queryR)
	}else {
		// 意味着所关注的区间一部分在左孩子，一部分在右孩子

		leftResult := s.queryInTree(leftTressIndex,l,mid,queryL,mid)

		rightResult := s.queryInTree(rightTreeIndex,mid+1,r,mid+1,queryR)

		return leftResult.Merge(rightResult)
	}
}

func (s *SegmentTree)Set(index int,e common.Segment)  {

	if index <0 || index >= len(s.data){
		return
	}
	s.set(0,0, len(s.data)-1,index,e)
}


func (s *SegmentTree)set(treeIndex,l,r,index int,e common.Segment) {
	if l == r {
		s.tree[treeIndex] = e
		return
	}
	mid := l + (r -l)/2

	leftTreeIndex := leftChild(treeIndex)
	rightTreeIndex := rightChild(treeIndex)
	if index >= mid +1 {
		s.set(rightTreeIndex,mid + 1,r,index,e)
	}else {
		s.set(leftTreeIndex,l,mid,index,e)
	}
	s.tree[treeIndex] = s.tree[leftTreeIndex].Merge(s.tree[rightTreeIndex])
}

```
##### 经典题目 区间染色

有一面墙，长度为n，每次选择一段儿墙进行染色，每一次染色过后，可以覆盖染色，在m次操作过后，我们可以看见多少种颜色？m次操作过后，我们可以在[i,j]这段种有多少种颜色


若使用数组，在某一段之间染色，将对应数组设置为对应颜色，那么染色，和查询的时间复杂度都是O(n)

##### 经典题目 区间查询

查询一个区间[i,j]的最大值、最小值、和等。
