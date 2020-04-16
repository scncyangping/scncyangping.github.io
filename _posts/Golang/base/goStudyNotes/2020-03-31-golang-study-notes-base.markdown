---
layout:     post
title:      "Go学习笔记 - (一)类型"
subtitle:   ""
date:       2020-03-31 02:40:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

### 类型
Go语言是一门静态编译型语言，是一门强类型语言，Go语言中类型分为两种：命名类型(已定义类型)和未命名类型(组合类型)

1. 命名类型

```
uint8(byte) uint16 uint32 uint64 int int8 int16 int32(rune) int64 bool string
float32 float64 complex64 complex128
```
上面举例类型归为三大类：，数值类型，字符串类型， 布尔值类型，我们使用type定义的任何类型也被称为命名类型

```
//也是命名类型
type MyBool bool
```

2. 非命名类型

```
slice map chan function interface struct pointer
```


#### 类型转换和断言

类型转换是用来在类型不同但相互兼容的类型之间的相互转换的方式，如果不兼容，则无法相互转换，编译会报错,通常写法是 a(b),把b转换成a

类型断言是在接口之间进行，本质也是类型转换，写法是a.(b),含义是把a转换成b




#### 变量
Go是静态类型语言，不能再运行期改变变量的类型

使用var定义变量，自动初始化为零值。如果提供初始化值，可省略变量类型，由编译器⾃自动推断

```
var x int
var f float32 = 1.6
var s = "abc"
```

在函数内部，可以使用 " := "的方式定义变量,也可同时赋值多个

```
var x, y, z int
var s, n = "abc", 123
var (
a int
b float32 )
func main() {
    n, s := 0x1234, "Hello, World!"
    println(x, s, n)
}
```

#### 常量
常量值必须是编译期可确定的数字、字符串、布尔值

```
// 多常量初始化
const x, y int = 1, 2
// 类型推断
const s = "Hello, World!"
// 常量组
const ( 
    a, b = 10, 100
    c bool = false
)
```

itoa 自动增长变量

```text
const(
    _   = itoa
    KB  int64 = 1 << (10 * iota)
    MB
    GB
    TB
)

在同⼀一常量组中，可以提供多个 iota，它们各⾃自增⻓长

const(
   A,B=iota,iota<<10 //0,0<<10
   C, D // 1, 1 << 10 
)

如果 iota ⾃自增被打断，须显式恢复

const(
  A =iota   // 0
  B         // 1
  C = "c"   // c
  D         // c 与上一行相同
  E = iota  // 4 显示恢复,这里就变成了4
  F         // 5   
)

```

#### 引用类型
引⽤用类型包括 slice、map 和 channel。它们有复杂的内部结构，除了申请内存外，还需要初始化相关属性

内置函数 new 计算类型⼤大⼩小，为其分配零值内存，返回指针。⽽而 make 会被编译器翻译 成具体的创建函数，由其分配内存和初始化成员结构，返回对象⽽而⾮非指针

#### 字符串

字符串是不可变值类型，内部⽤用指针指向 UTF-8 字节数组

- 默认值是空字符串 ""
- 用索引号访问某字节，如 s[i]
- 不能用序号获取字节元素