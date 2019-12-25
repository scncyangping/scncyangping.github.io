---
layout:     post
title:      "Go标准库-testing"
subtitle:   ""
date:       2019-08-18 02:30:00
author:     "YaPi"
header-img: ""
tags:
    - golang
---

#### test - Go包的⾃动化单元测试

测试文件的文件名需要以_test.go结尾,测试用例需要以TestXxxx的样式存在

- testing 基本用例测试
- testing/iotest - io测试
- testing/quick 进⾏⿊盒测试的辅助功能包
- net/http/httptest 提供测试 HTTP 的⼯具

##### Test

- TestXxxx(t *testing.T) 基本测试用例
- BenchmarkXxxx(b *testing.B) // 压⼒测试的测试⽤例
- Example_Xxx() // 测试控制台输出的例⼦
- TestMain(m *testing.M) // 测试Main函数

##### gotest变量

- test.short:⼀个快速测试的标记，在测试⽤例中可以使⽤testing.Short()来绕开⼀些测试
- test.outputdir:输出⽬录
- test.coverprofile:测试覆盖率参数，指定输出⽂件
- test.run:指定正则来运⾏某个/某些测试⽤例
- test.memprofile:内存分析参数，指定输出⽂件
- test.memprofilerate:内存分析参数，内存分析的抽样率
- test.cpuprofile:cpu分析输出参数，为空则不做cpu分析
- test.blockprofile:阻塞事件的分析参数，指定输出⽂件
- test.blockprofilerate:阻塞事件的分析参数，指定抽样频率
- test.timeout:超时时间
- test.cpu:指定cpu数量
- test.parallel:指定运⾏测试⽤例的并

##### testing包内结构

- B:压⼒测试
- BenchmarkResult:压⼒测试结果
- Cover:代码覆盖率相关结构体
- CoverBlock:代码覆盖率相关结构体
- InternalBenchmark:内部使⽤的结构
- InternalExample:内部使⽤的结构
- InternalTest:内部使⽤的结构
- M:main测试使⽤的结构
- PB:Parallelbenchmarks并⾏测试使⽤结果
- T:普通测试⽤例
- TB:测试⽤例的

##### testing包的通用方法

T结构内部是继承⾃common结构，common结构提供集中⽅法

当我们遇到⼀个断⾔错误的时候，我们就会判断这个测试⽤例失败，就会使⽤到

```
Fail : case失败，测试⽤例继续
FailedNow : case失败，测试⽤例中断
```
当我们遇到⼀个断⾔错误，只希望跳过这个错误，但是不希望标示测试⽤例失败,会使⽤到

```
SkipNow : case跳过，测试⽤例不继续
```

当我们只希望在⼀个地⽅打印出信息,我们会⽤到

```
Log : 输出信息
Logf : 输出有format的信息
```

当我们希望跳过这个⽤例,并且打印出信息

```
Skip : Log + SkipNow
Skipf : Logf + SkipNow
```

当我们希望断⾔失败的时候，测试⽤例失败，打印出必要的信息，但是测试⽤例继
续

```
Error : Log + Fail
Errorf : Logf + Fail
```

当我们希望断⾔失败的时候，测试⽤例失败，打印出必要的信息，测试⽤例中断

```
Fatal : Log + FailNow
Fatalf : Logf + FailNow
```

> 参考 go语言标准库