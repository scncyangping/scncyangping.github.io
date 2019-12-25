---
layout:     post
title:      "数据结构(五)树-二分搜索树"
subtitle:   ""
date:       2019-08-24 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---


#### 二叉树

- 二叉树具有唯一跟节点
- 二叉树中每个节点最多有两个孩子
- 没有孩子的节点称为叶子节点
- 二叉树中每个节点只能有一个父亲节点

二叉树不一定是满的

一个节点也是二叉树

NULL 空也是二叉树

#### 二分搜索树

- 二分搜索树是二叉树
- 二分搜索树的每个节点的值都大于其左子树的所有节点的值
- 二分搜索树的每个节点的值都小于其右子树的所有节点的值
- 每一棵子树也是二分搜索树
- 存储的元素必须有可比较性
- 二分搜索树是不能用来存放相同数据的数据结构，比较好用来实现Set集合
- 二分搜索树的增、查、删的时间复杂度是O(h) h是树的高度
- 同一组数据可以用创建出不同的二叉树，它有可能会退化成一个链表(在近乎有序的情况下，顺序添加)，解决这个问题-->创建平衡二叉树

```
已满二叉树为基础：


将树的根看作第0层，那么第0层1个节点，第1层2个节点，第三层8个节点...由此可得，第h-1层，有 2^(h-1)个节点

若有h层的话，它的节点个数就为 2^h - 1个

若将节点个数和数的高度建立关系的话

2^h - 1 = n

h = log2^(n+1)=O(log2^n) = O(logn)

```


 n | logn | n | 比较
---|--- | --- | ---
n = 16 | 4 | 16 | 相差4倍
n = 1024 | 10 | 1024 | 相差100倍
n = 100万 | 20 | 100万 | 相差5万倍






##### 二分搜索树的遍历

以当前节点的遍历顺序为基准,确定是哪一种遍历方式

- 前序遍历(DLR) 当前节点-->左子树-->右子树
- 中序遍历(LDR) 左子树-->当前节点-->-->右子树
- 前序遍历(LRD) 左子树-->右子树-->当前节点

二分搜索树的中序遍历结果是顺序的，所以也叫排序树
前序遍历的第一个元素为根节点，后续遍历最后一个节点是根节点。


后续遍历的一个应用: 为二分搜索树释放内存(c++)。

以上的遍历都是广度优先遍历，每一次都会先走到树最深处的位置，然后再返回


##### 广度优先遍历

需要借助队列实现，每次一层一层的遍历。即，一个节点出队的时候，先将其左右两个孩子入队，再出队。广度优先的优势在于你能更快的找到你需要的元素，因为深度优先遍历会先走到最深的地方。常用于算法设计中(无权图的最短路径)


#### 基础实现
```
/*
@date : 2019/09/05
@author : YaPi
@desc : 二分搜索树
*/
package tree

import (
	"dtSt/common"
	"dtSt/queue"
	"fmt"
)

// 二分搜索树
type Bst struct {
	node 	*Node
	size 	int
}

func NewBst() *Bst {
	return &Bst{}
}

// 二分搜索树前序遍历
func (b *Bst)DLR(){
	// 先遍历当前节点
	// 再遍历左子树
	// 再遍历右子树
	dlr(b.node)
}

func dlr(node *Node)  {
	if node == nil{
		return
	}
	fmt.Println(node.e)
	dlr(node.left)
	dlr(node.right)
}

// 二分搜索树中序遍历
func (b *Bst)LDR(){
	// 先遍历当前节点
	// 再遍历左子树
	// 再遍历右子树
	ldr(b.node)
}

func ldr(node *Node)  {
	if node == nil{
		return
	}
	ldr(node.left)
	fmt.Println(node.e)
	ldr(node.right)
}


// 二分搜索树后序遍历
func (b *Bst)LRD(){
	// 先遍历当前节点
	// 再遍历左子树
	// 再遍历右子树
	lrd(b.node)
}

func lrd(node *Node)  {
	if node == nil{
		return
	}
	lrd(node.left)
	lrd(node.right)
	fmt.Println(node.e)
}

func contains(node *Node, d common.E)bool{
	if node == nil {
		return false
	}
	r := node.e.CompareTo(d)
	if r == -1{
		return contains(node.right,d)
	}else if r == 1{
		// 当前节点的值比它大，那么需要在左子树中去查
		return contains(node.left,d)
	}else{
		return true
	}
}

func add(b *Node,d common.E)(*Node,bool){
	isAdd := false
	// 将nil当作一个为NULL的子节点,那么,判断当程序找到下一个节点是
	// NULL的时候，则代表需要新加一个节点
	if b == nil {
		return newNode(d),true
	}
	// 若b不为空的时候，则继续向下判断
	// 说明当前节点大于需要新增的节点
	res := b.e.CompareTo(d)
	if res == 1{
		b.left,isAdd = add(b.left,d)
	}else if res == -1{
		b.right,isAdd = add(b.right,d)
	}else{
		// 相等 不做操作
	}
	return b,isAdd
}

func removeMind(node *Node)*Node{
	if node.left == nil {
		rNode := node.right
		node.right = nil
		return rNode
	}
	node.left = removeMind(node.left)
	return node
}

func removeMax(node *Node)*Node{
	// 若右孩子为空，则当前节点为最大节点
	if node.right == nil {
		// 找到最大节点，找到其左孩子，左孩子为删除掉当前节点后的最大节点
		lNode := node.left
		node.left = nil
		return lNode
	}
	node.right = removeMax(node.right)
	return node
}


// 找二分搜索树的最小节点
func mind(n *Node)*Node{
	if n.left == nil{
		return n
	}
	// 获取节点的最右节点
	//lNode := mind(n.left)
	//if lNode != nil{
	//	return lNode
	//}else {
	//	return n
	//}
	return mind(n.left)
}

// 找二分搜索树的最大节点
func max(n *Node)*Node  {
	// 若当前节点为空 返回nil
	if n.right == nil{
		return n
	}
	// 获取节点的最右节点
	return max(n.right)
}

// 删除任意节点
func removeElement(node *Node,e common.E)*Node{
	// 非空判断
	if node == nil{
		return nil
	}
	// 和当前节点的值进行比较
	switch node.CompareTo(e) {
	case 1:
		// 当前节点的值比需要删除的值大
		// 需要在当前节点的左孩子进行查找
		node.left = removeElement(node.left,e)
	case -1:
		// 当前节点的值比需要删除的值小
		// 需要在当前节点的右孩子进行查找
		node.right = removeElement(node.right,e)
	case 0:
		// 需要删除的值就是当前值
		// 若当前节点没有左孩子和右孩子
		if node.left == nil && node.right == nil{
			node = nil
		}else if node.right != nil && node.left == nil {
			// 若当前节点只有右孩子 没有左孩子
			// 需要找到右子树当中最小的元素 用最小的元素当作当前节点的值
			n := mind(node.right)
			// 删除最小的元素 并返回删除过后的树
			rNode := removeMind(node.right)
			n.right = rNode
			node = n
		}else if node.left !=nil && node.right == nil {
			// 若当前节点只有左孩子 没有右孩子
			// 找到左子树当中最大的元素 用最大的元素当作当前节点值
			n := max(node.left)
			lNode := removeMax(node.left)
			n.left = lNode
			node = n
		}else if node.left != nil && node.right != nil {
			// 当前节点既有左孩子 也有右孩子
			// 找到左子树最小的值 作为当前节点的值
			n := mind(node.right)
			rNode := removeMind(node.right)
			n.left = node.left
			n.right = rNode
			node = n
		}
	default:
	}
	return node
}

func (b *Bst)RemoveElement(e common.E)  {
	removeElement(b.node,e)
}


func (b *Bst)RemoveMind()  {
	b.node = removeMind(b.node)
}

func (b *Bst)RemoveMax()  {
	b.node = removeMax(b.node)
}


func (b *Bst)Max()*Node  {
	return max(b.node)
}

func (b *Bst)Mind()*Node  {
	return mind(b.node)
}

// 层序遍历(又叫广度优先遍历) 借助队列
func (b *Bst)LevelOrder()(l []Node){
	// 借助队列实现
	q := queue.NewArrayQueue()
	node := b.node
	q.Enqueue(b.node)
	for !q.IsEmpty() {
		node = q.Dequeue().(*Node)
		// 将节点添加到数组里面返回
		l= append(l, *node)
		if node.left != nil {
			q.Enqueue(node.left)
		}
		if node.right != nil{
			q.Enqueue(node.right)
		}
	}
	return
}


func (b *Bst) Add(d common.E) {
	rNode,ok := add(b.node,d)
	b.node = rNode
	if ok {
		b.size ++
	}
}

func (b *Bst) Size() int {
	return b.size
}

func (b *Bst) IsEmpty() bool {
	return b.size == 0
}

func (b *Bst) Remove(d common.E) {
	panic("implement me")
}

func (b *Bst) Contains(d common.E) bool {
	return contains(b.node,d)
}
// 版本1
//func (b *Bst) Add(d int) {
//	if b.node == nil {
//		b.node = newNode(d)
//		b.size++
//		return
//	}else {
//		add(b.node, d)
//	}
//}
//func add(b *Node,d int){
//	// 判断当前节点和此根节点是否相等
//	// 若相等则不做操作 退出
//	if b.IsEqual(d){
//		return
//	}
//	// 判断当前值和传入值的比较
//	switch b.Compare(d) {
//	case 0:
//		// 相等就不做处理
//		return
//	case 1:
//		// 这个值比当前值小
//		// 应该插入左子树
//		// 判断左子树是否存在
//		if b.left != nil {
//			add(b.left,d)
//		}else {
//			// 不存在左子树
//			// 新建一个 并将当前左子树节点加入
//			b.left = newNode(d)
//		}
//	case -1:
//		// 这个值比当前值大
//		// 应该插入右子树
//		if b.right != nil {
//			add(b.right,d)
//		}else {
//			// 不存在左子树
//			// 新建一个 并将当前左子树节点加入
//			b.right = newNode(d)
//		}
//	default:
//		return
//	}
//}
```

