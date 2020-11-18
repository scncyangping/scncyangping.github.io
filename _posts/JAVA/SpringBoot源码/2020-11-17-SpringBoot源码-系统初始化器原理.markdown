---
layout:     post
title:      "SpringBoot源码系列之系统初始化器原理"
subtitle:   ""
date:       2020-11-16 14:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

### 概要

用来加载系统初始化器

- 框架内部使用的通用工厂加载机制
- 从classpath下多个jar包特定的位置读取文件并初始化类
- 文件内容必须是kv形式，即properties类型
- key是全限定名(抽象类｜接口)、value是实现,多个实现用都好分割

#### 加载实现源码

1. spring.factories 系统初始化器加载过程

它的初始化在框架初始化过程中
SpringApplication.run() -> new SpringApplication()

```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 可配置资源加载器(文件、资源的配置读取) 手动指定资源加载器(类加载器) 不指定取默认加载器
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 配置primarySources(就是启动传递过去的类，一般都是启动类)
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 应用环境检测(springBoot2会检测环境是一个web环境还是reactnative环境)
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 配置系统初始化器
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 配置应用监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 配置main方法所在类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```


```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 获取类加载器，先获取当前线程设置的类加载器Thread.currentThread().getContextClassLoader()
    // 没有就获取当前类的类加载器ClassUtils.class.getClassLoader(),若还没有获取系统类加载器ClassLoader.getSystemClassLoader()
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 获取路径META-INF/spring.factories下的文件内容，并读取内容返回指定类名，有缓存直接返回缓存，没有就继续读取
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 通过类名生成相关实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 根据配置的order进行排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```
SpringFactoriesLoader加载在配置文件spring.factories中配置的系统初始化器。

首先扫描所有jar包中的resource目录下的META-INF目录下的spring.factories文件。然后将配置文件中的key-value键值对读取出来，存放在一个
LinkedMultiValueMap中。然后返回此对象。设置全局系统初始化器。然后返回。其中LinkedMultiValueMap底层是LinkedHashMap。是一个 Map<K, List<V>> 的数据结构。它的好处是
这样可以存放相同key值的多个value。然后根据类名，通过反射生成对应的对象。


2. main方法中编码的系统初始化器

```
SpringApplication application = new SpringApplication(Sb2Application.class);
application.addInitializers(new FirstInitializer());
application.run(args);
```
通过addInitializers方法手动添加到全局系统初始化器集合

3. 在application.properties环境变量中指定的类。被SpringBoot默认的DelegatingApplicationContextInitializer系统初始化器注册。
而DelegatingApplicationContextInitializer是定义在SpringBoot默认jar包的spring.factories文件中。并且，此默认系统初始话器配置
的order为0，所以，它会优先于其他方式配置的系统初始化器执行

#### 主要流程

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/springBoot%E6%BA%90%E7%A0%81/SpringFactories%E6%B5%81%E7%A8%8B.jpg)

#### 调用实现源码

SpringApplication.run()方法中调用 --> prepareContext()方法 --> applyInitializers()方法 -->获取到所有系统初始化器后调用其initialize方法