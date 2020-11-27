---
layout:     post
title:      "SpringBoot源码系列之异常报告器解析"
subtitle:   ""
date:       2019-01-14 14:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 流程

构建过程

run 方法
```
Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

...

exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
        new Class[] { ConfigurableApplicationContext.class }, context);
```

1. run方法中构建Collection<SpringBootExceptionReporter>集合
2. 运行getSpringFactoriesInstances方法。调用SpringFactoriesLoader.loadFactoryNames获取
SpringBootExceptionReporter的实现类。在SpringBoot中默认实现就一个FailureAnalyzers。将构建一个FailureAnalyzers对象。
并将结果填充到集合。在构建FailureAnalyzers对象时，会查找FailureAnalyzers对象实现，并添加到其analyzers变量当中。同时，
会判断其是否实现了BeanFactoryAware或EnvironmentAware。若实现了，将设置相关属性。

调用过程 run方法里面的handleRunFailure

1. handleExitCode
    1.1 exitCode: 退出状态码，为0代表正常退出，否则异常退出
    1.2 发布ExitCodeEvent事件
    1.3 记录exitCode
2. listeners.failed
    2.1 发布ApplicationFailedEvent事件
3. reportFailure
    3.1. 就是对FailureAnalyzers的reportException方法的调用。
    3.2. 其中有两个重要的方法analyze方法和report方法
    3.3. analyze方法,遍历analyzers集合，通过其实现类范型与异常类型的比较，找到能处理该异常的对象，组装其自有的对错误的描述。并返回此对象
    3.4. 若返回异常描述对象，同时，获取系统中FailureAnalysisReporter的实现类，并且依次调用其report方法。
    这也就表明，一个异常，可以被多次捕获处理。
4. context.close
    4.1 更该应用上下文状态，将AbstractApplicationContext.closed改为true，同时active设置为false
    4.2 发送ContextClosedEvent事件
    4.2 销毁单例bean
    4.3 将beanFactory置为空
    4.4 若处于web环境，关闭web容器
    4.5 若注册了关闭钩子方法，将其移除
5. ReflectionUtils.rethrowRuntimeException
    5.1 重新将异常抛出。SpringBoot异常处理流程结束


#### 关闭钩子方法(shutdownHook)
在jvm退出时执行的业务逻辑

```
System.out.println("hello");

Thread close = new Thread(()-> System.out.println("close jvm"));

Runtime.getRuntime().addShutdownHook(close);

System.out.println("world");

// 若需删除
// Runtime.getRuntime().removeShutdownHook(close);
```


#### 实例

##### 设置指定异常返回的 exitCode

```
@Component
public class MyExitCodeExceptionMapper implements ExitCodeExceptionMapper {
    @Override
    public int getExitCode(Throwable exception) {
        if (exception instanceof ConnectorStartFailedException) {
            return 10;
        }
        return 0;
    }
}
```

##### 自定义SpringBootExceptionReport实现

创建

```
public class MyExceptionReporter implements SpringBootExceptionReporter {

    // 构建需要的必须参数
    private ConfigurableApplicationContext context;

    public MyExceptionReporter(ConfigurableApplicationContext context) {
        this.context = context;
    }

    @Override
    public boolean reportException(Throwable failure) {
        if (failure instanceof UnsatisfiedDependencyException) {
            UnsatisfiedDependencyException exception = (UnsatisfiedDependencyException) failure;
            System.out.println("no such bean " + exception.getInjectionPoint().getField().getName() );
        }
        // 返回true，则不执行后续的异常处理，返回false，继续执行
        return false;
    }
}

```

引入：在spring.factories里面引入(因为它是通过SpringFactoriesLoader获取的)
org.springframework.boot.SpringBootExceptionReporter=com.mooc.sb2.except.MyExceptionReporter


#### 问题

1. 关闭钩子方法的作用及使用方法？
2. 介绍下SpringBoot异常报告器类结构？
3. 介绍下SpringBoot异常报告器实现原理？
4. SpringBoot异常处理流程？
    handleRunFailure

5. SpringBoot异常处理流程中有哪些注意事项？
    5.1 ExitCodeEvent事件不一定会发布，取决于返回的code值，需大于0，可自定义
    5.2 listeners.failed 发布监听器不一定发布，可能发生异常的时候，监听器还未初始化成功
6. 如何自定义实现SpringBoot异常报告器？