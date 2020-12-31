---
layout:     post
title:      "数据结构(八)Trie 字典树"
subtitle:   ""
date:       2019-08-25 02:40:00
author:     "YaPi"
header-img: ""
tags:
    - 数据结构
---


#### 什么是前缀树 Trie

多用来进行字符串匹配

- 多叉树

```
// 如果不仅仅是英文单词的话，还会加上特殊字符等
// 所以一般不会设置next为指定容量的结构
type Node struct {
    // 当前节点的值
    s       string
    // 是否代表了一个单词的结尾
    isWord  bool
    next    []Node
}
```

缺点：

空间需求增大，通常来说，对于单词来说，每一个节点，至少需要26个空间的子节点，所以是原单词的27倍空间

解决方法：

- 压缩Trie
    > 将cat这种单词放在一个节点，而不是三个节点，维护成本增大
- Ternary Search Trie(三分搜索Trie)
    > 所有节点子孩子只有三个节点， 通常大于在左，等于在中，小于在右

#### 实现

```
/*
@date : 2019/09/16
@author : YaPi
@desc : 字典树 本质就是一个多叉树
*/
package tree

import "strings"

type TrieNode struct {
	// 表示该节点是否是单词节点
	isWord		bool
	// 该节点的值
	value 		string
	// 该节点的子节点指针数组
	// nodes		[]*TrieNode
	// 使用map方便获取指定节点
	nodes 		map[string]*TrieNode
}

func NewTrieNode() *TrieNode {
	return &TrieNode{isWord: false,nodes:make(map[string]*TrieNode)}
}

func NewTrieNodeByValue(value string,isWord bool) *TrieNode {
	return &TrieNode{isWord: isWord,nodes:make(map[string]*TrieNode),value:value}
}

type Trie struct {
	root 		*TrieNode
	size 		int
}

func Constructor() *Trie {
	return &Trie{root:NewTrieNode()}
}

func (t *Trie)GetSize()int{
	return t.size
}

func (t *Trie)Add(word string)  {
	node := t.root
	for _,v := range word{
		n := node.nodes[string(v)]
		if n == nil {
			node.nodes[string(v)] = NewTrieNodeByValue(string(v),false)

		}
		node = node.nodes[string(v)]
	}
	// 该节点是插入单词对应的最后一个节点
	if ! node.isWord {
		node.isWord = true
		t.size ++
	}
}

// 查看是否包含某个单词
func (t *Trie)Contains(word string)bool  {
	node := t.root
	for _,v := range word{
		n := node.nodes[string(v)]
		if n == nil {
			return false
		}
		node = node.nodes[string(v)]
	}
	// 判断该节点是否是一个单词节点
	if node.isWord {
		return true
	}else {
		return false
	}
}

// 用.号匹配任意字符
func (t *Trie)Match(word string)bool  {
	return t.match(t.root,strings.Split(word,""),0)
}
// 查看是否包含某个单词
func (t *Trie)match(node *TrieNode,word []string,index int)bool  {
	if index == len(word){
		return node.isWord
	}
	s := word[index]

	if s != "."{
		if node.nodes[s] == nil {
			return false
		}
		return t.match(node.nodes[s],word,index + 1)
	}else {
		for _,v := range node.nodes{
			if t.match(v,word,index + 1){
				return true
			}
		}
		return false
	}
}

// 查询是否有以prefix前缀的单词
func (t *Trie)IsPrefix(prefix string)bool  {
	node := t.root
	for _,v := range prefix{
		n := node.nodes[string(v)]
		if n == nil {
			return false
		}
		node = node.nodes[string(v)]
	}
	return true
}

```

#### 扩展

##### 后缀树

后缀树，就是把一串字符的所有后缀保存并且压缩的字典树。相对于字典树来说，后缀树并不是针对大量字符串的，而是针对一个或几个字符串来解决问题。比如字符串的回文子串，两个字符串的最长公共子串等等。
性质：一个字符串构造了一棵树，树中保存了该字符串所有的后缀。
操作：就是建立和应用。

后缀树能解决大多数字符串的问题

（1）查找某个字符串s1是否在另外一个字符串s2中。这个很简单，如果s1在字符串s2中，那么s1必定是s2中某个后缀串的前缀。理解以下后缀串的前缀这个词，其实每个后缀串也就是起始地点不同而已，前缀也就是从开头开始结尾不定。后缀串的前缀就可以组合成该原先字符串的任意子串了。比如banana，anan是anana这个后缀串的前缀。
（2）指定字符串s1在字符串s2中重复的次数
比如说banana是s1，an是s2，那么计算an出现的次数实际上就是看an是几个后缀串的前缀。上图的a节点是保存所有起始为a字母的后缀串，我们看a字母后的n字母的引用计数即可。
（3）两个字符串S1，S2的最长公共部分（广义后缀树）
（4）最长回文串（广义后缀树）