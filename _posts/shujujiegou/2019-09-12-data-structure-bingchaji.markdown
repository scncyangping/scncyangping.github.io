---
layout:     post
title:      "数据结构(九)并查集"
subtitle:   ""
date:       2019-08-25 02:45:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---

#### 并查集

并查集，在一些有N个元素的集合应用问题中，我们通常是在开始时让每个元素构成一个单元素的集合，然后按一定顺序将属于同一组的元素所在的集合合并，其间要反复查找一个元素在哪个集合中。这一类问题近几年来反复出现在信息学的国际国内赛题中，其特点是看似并不复杂，但数据量极大，若用正常的数据结构来描述的话，往往在空间上过大，计算机无法承受；即使在空间上勉强通过，运行的时间复杂度也极高，根本就不可能在比赛规定的运行时间（1～3秒）内计算出试题需要的结果，只能用并查集来描述。
并查集是一种树型的数据结构，用于处理一些不相交集合（Disjoint Sets）的合并及查询问题。常常在使用中以森林来表示。

主要用于解决连接问题，它支持两种操作

- union(p,q)
- isConnected(p,q)

#### 实现

```
/*
@date : 2019/09/16
@author : YaPi
@desc : 并查集
*/
package unionFind


// 将每一个元素，看作是一个节点，由孩子节点指向根节点
type UF struct {
	// 代表一系列根节点的数据
	parent 		[]int
	// 代表每一个根节点代表的树的节点总数
	sz 			[]int
	// 代表每一个根节点代表的树的高度
	rank 		[]int
}

// 初始化数据，每一个节点都是指向自身的一颗树
func NewUF(size int) *UF {
	ids := make([]int,size)
	szs := make([]int,size)
	rank := make([]int,size)

	for i:=0;i<size;i++{
		ids[i] = i
		szs[i] = 1
	}
	return &UF{parent: ids,sz:szs,rank:rank}
}

// 查找父亲节点
func (u *UF)find(p int)int{
	if p < 0 || p >= len(u.parent){
		panic("p is out of bound")
	}

	// 找到这个数据所处的根节点
	for p != u.parent[p]{
		// 路径压缩
		u.parent[p] = u.parent[u.parent[p]]
		// 添加了路径压缩过后，rank就不代表具体的高度了
		// 只是代表每一个节点的排名
		p = u.parent[p]
	}
	return p
}

// 递归方式的路径压缩 会将所有节点直接指向根节点
// 递归方式的开销比循环遍历要差一点
func (u *UF)find2(p int)int{
	if p < 0 || p >= len(u.parent){
		panic("p is out of bound")
	}
	if p != u.parent[p]{
		u.parent[p] = u.find2(u.parent[p])
	}
	return u.parent[p]
}


// 判断是否是连通
func (t *UF)IsConnected(p,q int)bool  {
	return t.find(p) == t.find(q)
}

// 合并
// 基于rank的优化
func (t *UF)UnionElements(p,q int)  {
	pRoot := t.find(p)
	qRoot := t.find(q)

	if pRoot == qRoot {
		return
	}

	// 基于size的优化
	// 每次将树节点少的那一棵树，合并到树节点多的那一棵树
	//if t.sz[pRoot] < t.sz[qRoot]{
	//	// 将 p 所在的根节点对应的id指向 q 所在的根节点
	//	t.parent[pRoot] = qRoot
	//	t.sz[qRoot] += t.sz[pRoot]
	//}else {
	//	t.parent[qRoot] = pRoot
	//	t.sz[pRoot] += t.sz[pRoot]
	//}

	// 基于rank的优化
	if t.rank[pRoot] < t.rank[qRoot]{
		// 将 p 所在的根节点对应的id指向 q 所在的根节点
		// qRoot所对应的根节点的高度比pRoot对应的大，合并了过后，此树
		// 的根节点还是qRoot，所以不需要改变qRoot或pRoot对应的树的高度值
		t.parent[pRoot] = qRoot
	}else if t.rank[qRoot] < t.rank[pRoot]{
		// 原理同上
		t.parent[qRoot] = pRoot
	}else {
		// 相等了随便指向一个都行，但是树的高度增加了1，需要重新维护树的高度
		t.parent[qRoot] = pRoot
		t.rank[pRoot] += 1
	}
}

// 路径压缩 Path Compression

```