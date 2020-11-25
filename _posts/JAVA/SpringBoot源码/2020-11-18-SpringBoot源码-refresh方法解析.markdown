---
layout:     post
title:      "SpringBoot源码系列之Refresh方法解析"
subtitle:   ""
date:       2019-01-07 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 方法主要作用

1. bean配置读取
2. spring框架启动流程

#### refresh方法

refresh方法是SprintBoot启动的一个重要步骤，其中包含了很多独立的方法

##### prepareRefresh
在这个方法中，会进行如下设置：

- 容器状态设置，比如启动时间
- 初始化属性设置，比如业务监听器
- 检查必备属性是否存在

```
必备属性的设置：
系统初始化器中设置
@Order(1)
public class FirstInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        environment.setRequiredProperties("mooc");
//        Map<String, Object> map = new HashMap<>();
//        map.put("key1", "value1");
//        MapPropertySource mapPropertySource = new MapPropertySource("firstInitializer", map);
//        environment.getPropertySources().addLast(mapPropertySource);
//        System.out.println("run firstInitializer");
    }
}
```

##### obtainFreshBeanFactory方法

```
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

1. 设置beanFactory序列化ID
2. 获取beanFactory

##### prepareBeanFactory方法

主要对beanFactory做一些基础配置

1. 设置beanFactory一些属性
2. 添加后置处理器
3. 设置忽略的自动装配接口
4. 注册一些组建

##### postProcessBeanFactory方法

子类重写以在BeanFactory完成创建后做进一步设置。比如设置web环境的作用域及环境信息

##### invokeBeanFactoryPostProcessors方法

此方法主要做两件事情

1. 调用BeanDefinitionRegistryPostProcessor实现向容器添加bean的定义

```
@Component
public class MyBeanRegister implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition();
        rootBeanDefinition.setBeanClass(Monkey.class);
        registry.registerBeanDefinition("monkey", rootBeanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

    }
}
```

2. 调用BeanFactoryPostProcessor实现向容器内bean的定义添加属性

```
@Component
public class MyBeanFactoryPostprocessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition teacher = beanFactory.getBeanDefinition("teacher");
        MutablePropertyValues propertyValues = teacher.getPropertyValues();
        propertyValues.addPropertyValue("name", "wangwu");
    }
}
```

##### registerBeanPostProcessors方法

1. 找到BeanPostProcessor的实现
2. 排序后注册进容器内

##### initMessageSource方法

初始化国际化相关属性

##### initApplicationEventMulticaster方法
初始化事件广播器


##### onRefresh方法
留给子类实现的，如果是web容器，就会去初始化相关容器，比如tomacat，jetty等


##### registerListeners方法

1. 添加容器内事件监听器至事件广播器中
2. 派发早期事件(没有创建完监听器的时候就有的事件)

##### finishBeanFactoryInitialization 方法

1. 初始化所有剩下的单实例bean


##### finishRefresh 方法

1. 初始化生命周期处理器
2. 调用生命周期处理器onRefresh方法
3. 发布ContextRefreshedEvent事件
4. JMX相关处理




