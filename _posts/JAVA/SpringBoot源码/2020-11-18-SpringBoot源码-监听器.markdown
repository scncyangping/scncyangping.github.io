---
layout:     post
title:      "SpringBoot源码系列之监听器"
subtitle:   ""
date:       2019-01-04 14:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

### 概要
监听器，字面上的理解就是监听观察某个事件（程序）的发生情况，当被监听的事件真的发生了的时候，事件发生者（事件源） 就会给注册该事件的监听者（监听器）发送消息

监听器要素
1. 事件
2. 监听器
3. 广播器
4. 触发机制

#### 在SpringBoot中的实现

- 监听器 ApplicationListener

```
// 声明此类只有一个方法
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
	void onApplicationEvent(E event);
}
```

- 系统广播器 ApplicationEventMulticaster

```
public interface ApplicationEventMulticaster {

	void addApplicationListener(ApplicationListener<?> listener);

	void addApplicationListenerBean(String listenerBeanName);

	void removeApplicationListener(ApplicationListener<?> listener);

	void removeApplicationListenerBean(String listenerBeanName);

	void removeAllListeners();

	void multicastEvent(ApplicationEvent event);

	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);
}
```

- 系统事件 ApplicationEvent



#### 监听器的触发机制

- 获取监听器列表

1. getApplicationListeners,判断缓冲中，是否有该监听器感兴趣的监听器列表，若有则直接返回
2. retrieveApplicationListeners，遍历所有监听器实现
3. 遍历监听器
4. supportsEvent
5. 加入符合条件的监听器列表

```

#### 流程分析

Spring容器运行到某些关键节点，就会触发监听器。比如启动监听器在SpringApplication.run()方法中会发送starting事件

```
// 获取现有监听器列表
SpringApplicationRunListeners listeners = getRunListeners(args);
// 启动
listeners.starting();
```




SpringBoot将一些事件封装成了SpringApplicationRunListener的实现类。将事件的创建，发送封装在一个单独的类中。
将业务逻辑和具体的调用隔离。

```
public interface SpringApplicationRunListener {

    // SpringBoot首次启动立即调用，用于早期初始化
    void starting();

    // 在环境准备好之后，且上下文(ApplicationContext)准备好之前调用
    void environmentPrepared(ConfigurableEnvironment environment);

    // 上线文准备好之后，且资源加载之前
    void contextPrepared(ConfigurableApplicationContext context);

    // 上下文准备好，未刷新
    void contextLoaded(ConfigurableApplicationContext context);

    // 上下文已刷新，但CommandLineRunner及ApplicationRunners未被调用
    void started(ConfigurableApplicationContext context);

    // 上下文刷新，且CommandLineRunner及ApplicationRunners都执行
    void running(ConfigurableApplicationContext context);

    // 出错时
    void failed(ConfigurableApplicationContext context, Throwable exception);
}



使用staring事件进行分析 EventPublishingRunListener implements SpringApplicationRunListener { ...

```

public void starting() {
    // initialMulticaster 是 SimpleApplicationEventMulticaster(广播器)的实例对象
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```
在multicastEvent方法中，主要目的是找到监听器列表中，对此事件感兴趣的监听器。其主要流程为：

1. 将此事件构建为ResolvableType类型
2. 若线程有线程池，获取线程池，以便执行后续监听器。若没有，则同步执行
3. 根据当前事件，及类型，找到所有监听器当中，对此事件感兴趣的监听器。

```
判断是否感兴趣的源码中，最重要的逻辑在 supportsEvent方法中

if (supportsEvent(listener, eventType, sourceType)) {
    ...
}


protected boolean supportsEvent(
			ApplicationListener<?> listener, ResolvableType eventType, @Nullable Class<?> sourceType) {
    // 构建成GenericApplicationListener对象，4.2后更新，区别于SmartApplicationListener,提供更多的功能
    GenericApplicationListener smartListener = (listener instanceof GenericApplicationListener ?
            (GenericApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
    return (smartListener.supportsEventType(eventType) && smartListener.supportsSourceType(sourceType));
}


// 首先判断 eventType。事件类型

supportsEventType方法,判断待发送事件与当前监听器监听的事件类型是否匹配

1. 此监听器是否是SmartApplicationListener类型。

若是，判断实现类的supportsEventType方法。
若不是，判断监听器实现的接口的指定的类型变量是否是待发送事件的同类或子类

2. 若上诉判断通过,调用supportsSourceType方法。判断监听器类型是否是SmartApplicationListener

若是，判断实现类supportsSourceType方法
若不是，返回true

```

4. 使用上述获得的监听器调用分发事件(onApplicationEvent 方法)。





#### 自定义实现
创建监听器两种方式
1. 实现 ApplicationListener 接口。针对单一事件进行监听
```
@Order(1)
public class FirstListener implements ApplicationListener<ApplicationStartedEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        System.out.println("zzz hello first");
    }
}
```
2. 实现 SmartApplicationListener 接口。针对多种事件进行监听
```
@Order(4)
public class FourthListener implements SmartApplicationListener {
    @Override
    public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
        return ApplicationStartedEvent.class.isAssignableFrom(eventType) || ApplicationPreparedEvent.class.isAssignableFrom(eventType);
    }

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("hello fourth");
    }
}
```

TIPS: Order值越小越先执行，但是在application.properties中配置的优先执行
#### 注册

同注册系统初始化器一样，有三种创建方式
1. 通过在spring.factories文件中配置 org.springframework.context.ApplicationListener=全路径名
2. 在启动类中配置

```
SpringApplication application = new SpringApplication(Sb2Application.class);
application.addListeners(new FirstListener());
application.run(args);

```
3. 在application.properties中配置context.listener.classes=全路径名。也是有一个默认的DelegatingApplicationListener会去加载


#### SpringBoot框架事件发送顺序

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/springBoot%E6%BA%90%E7%A0%81/SpringBoot%E4%BA%8B%E4%BB%B6%E5%8F%91%E9%80%81%E9%A1%BA%E5%BA%8F.jpg)

#### SpringBoot监听事件触发机制

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/springBoot%E6%BA%90%E7%A0%81/%E7%9B%91%E5%90%AC%E5%99%A8%E9%80%9A%E7%94%A8%E8%A7%A6%E5%8F%91%E6%9D%A1%E4%BB%B6.jpg)

#### 总结

1. 介绍下监听器模式
2. SpringBoot关于监听器相关实现类有哪些？
可以看spring.factories文件中的类。比如DelegatingApplicationListener、AnsiOutputApplicationListener、ConfigFileApplicationListener等
3. SpringBoot框架有哪些框架事件以及它们的实现顺序？
上图
4. 监听事件的触发机制
上图
5. 如何自定义系统监听器及注意事项？
实现ApplicationListener接口与SmartApplicationListener。order顺序问题。
6. 实现ApplicationListener接口与SmartApplicationListener接口的区别？
ApplicationListener需指定监听类型范型，只能监听一种。后者可监听多种



