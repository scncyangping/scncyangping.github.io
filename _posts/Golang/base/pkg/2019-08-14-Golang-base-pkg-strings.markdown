---
layout:     post
title:      "Golang标准库-strings"
subtitle:   ""
date:       2019-08-13 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

##### 是否存在某个字符或者字串

- func Contains(s, substr string) bool
- func ContainsAny(s, chars string) bool
- func ContainsRune(s string, r rune) bool


##### 字串出现次数(字符串匹配)

- 朴素匹配算法
- KMP算法
- Rabin-Karp算法
- Boyer-Moore算法

Go语言中查找字串出现次数实现的是Rabin-Karp算法

```
func Count(s, substr string) int
```

###### 特殊处理

```
fmt.Print(strings.Count("five", "")) // 5

// Count仅计算字串在字符串中出现的无重叠次数
fmt.Print(strings.Count("fivevev", "vev")) // 1
```

##### 字符串分割为 []string

###### Fields 和 FieldsFunc

Fields ⽤⼀个或多个连续的空格分隔字符串 s，返回⼦字符串的数组（slice）。如果
字符串 s 只包含空格，则返回空列表([]string的⻓度为0）

```
func Fields(s string) []string
```


FieldsFunc  自定义分割函数,其实Fields就是调用了指定函数为unicode.IsSpace的FieldsFunc
```
func FieldsFunc(s string, f func(rune) bool) []string
```

###### Split、SplitAfter 和 SplitN、SplitAfterN

它们都是通过⼀个同⼀个内部函数来实现的,若分隔符(sep)为空,则分割成
一个一个的字符
```
func Split(s, sep string) []string { return genSplit(s, sep, 0, -1) }

func SplitAfter(s, sep string) []string { return genSplit(s, sep, len(sep), -1) }

func SplitN(s, sep string, n int) []string { return genSplit(s, sep, 0, n) }

func SplitAfterN(s, sep string, n int) []string { return genSplit(s, sep, len(sep), n)}
```

Split 会将 s 中的 sep(分隔符) 去掉，⽽ SplitAfter 会保留 sep


```
带 N 的⽅法可以通过最后⼀个参数 n 控制返回的结果中的 slice 中的元素个数，当 n
< 0 时，返回所有的⼦字符串；当 n == 0 时，返回的结果是 nil；当 n > 0 时，表示返
回的 slice 中最多只有 n 个元素，其中，最后⼀个元素不会分割，

fmt.Printf("%q\n", strings.SplitN("foo,bar,baz", ",", 2))
输出 : ["foo" "bar,baz"]
```

##### 是否已某个字符串开始或结尾

- func HasPrefix(s, prefix string) bool
- func HasSuffix(s, suffix string) bool

```
// s 中是否以 prefix 开始
func HasPrefix(s, prefix string) bool {
	return len(s) >= len(prefix) && s[0:len(prefix)] == prefix
}


// s 中是否以 suffix 结尾
func HasSuffix(s, suffix string) bool {
	return len(s) >= len(suffix) && s[len(s)-len(suffix):] == suffix
}
```

##### 字符或字串在字符串中的位置函数

```
// 在 s 中查找 sep 的第⼀次出现，返回第⼀次出现的索引
func Index(s, sep string) int

// chars中任何⼀个Unicode代码点在s中⾸次出现的位置
func IndexAny(s, chars string) int

// 查找字符 c 在 s 中第⼀次出现的位置，其中 c 满⾜ f(c) 返回 true
func IndexFunc(s string, f func(rune) bool) int

// Unicode 代码点 r 在 s 中第⼀次出现的位置
func IndexRune(s string, r rune) int

// 有三个对应的查找最后⼀次出现的位置
func LastIndex(s, sep string) int

func LastIndexAny(s, chars string) int

func LastIndexFunc(s string, f func(rune) bool) int

```

##### 字符数组连接

- func Join(a []string, sep string) string

```
GO语言中str是不可变的,所有用byte数组避免⼤量的字符串连接操作

func Join(a []string, sep string) string {
    // 特殊情况处理
	switch len(a) {
	case 0:
		return ""
	case 1:
		return a[0]
	}
	// 计算分隔符所占据的空间
	// 加上原有字符的空间,用于生成byte数组
	n := len(sep) * (len(a) - 1)
	for i := 0; i < len(a); i++ {
		n += len(a[i])
	}

	var b Builder
	// 生成数组
	b.Grow(n)
	// 添加第一个元素
	b.WriteString(a[0])
	// 若数组不止一个,添加后续数据
	for _, s := range a[1:] {
	    // 写入分隔符
		b.WriteString(sep)
		// 写入数据
		b.WriteString(s)
	}
	return b.String()
}
```

##### 字符串重复

- func Repeat(s string, count int) string

```
// 输出 hehexixi
fmt.Println("hehe" + strings.Repeat("xi", 2))
```

##### 字符串替换

进⾏字符串替换时，考虑到性能问题，能不⽤正则尽量别⽤.应该⽤函数

- func Replace(s, old, new string, n int)
- func ReplaceAll(s, old, new string) string

```
// ⽤ new 替换 s 中的 old，⼀共替换 n 个。
// 如果 n < 0，则不限制替换次数，即全部替换
func Replace(s, old, new string, n int) string



func ReplaceAll(s, old, new string) string {
	return Replace(s, old, new, -1)
}
```

若需要一次替换调多个值,可以用Replacer 类型

```
func NewReplacer(oldnew ...string) *Replacer {
	if len(oldnew)%2 == 1 {
		panic("strings.NewReplacer: odd argument count")
	}
	return &Replacer{oldnew: append([]string(nil), oldnew...)}
}


使用

r := strings.NewReplacer("<", "&lt;", ">", "&gt;")
fmt.Println(r.Replace("This is <b>HTML</b>!"))


// 也可调用 r.WriteString()方法将替换过后的内容放在io.Writer中
func (r *Replacer) WriteString(w io.Writer, s string) (n int, err error)
```