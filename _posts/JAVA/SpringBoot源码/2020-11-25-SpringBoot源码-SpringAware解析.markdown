---
layout:     post
title:      "SpringBoot源码系列之SpringAware原理解析"
subtitle:   ""
date:       2019-01-12 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### Aware使用场景

在Spring框架中，Bean是感知不到容器的存在的，里面不存在任何容器相关的信息。
但是在某些场景需要使用到Spring容器的功能或资源，这个时候就需要使用到
Spring Aware。

#### 使用方式

直接实现相关接口，通过方法注入就能直接使用。

常用的Aware：

类名 | 作用
---|---
BeanNameAware | 获取容器中Bean名称
BeanClassLoaderAware | 获取类加载器
BeanFactoryAware | 获取bean创建工厂
EnvironmentAware | 获得环境变量
EmbeddedValueResolverAware | 获取spring容器加载的properties文件属性值
ResourceLoaderAware | 获得资源加载器
ApplicationEventPublisherAware | 获得应用时间发布器
MessageSourceAware | 获得文本信息(国际化)
ApplicationContextAware | 获得当前应用上线哦


#### 调用原理

AbStractBeanFactory#getBean->doGetBean->AbstractAutowireCapableBeanFactory#createBean->doCreateBean(创建Bean的方法里面)-> initializeBean -> invokeAwareMethods(在这里处理一部分)

->applayBeanPostProcessorsBeforeInitialization -> ApplicationContextAwareProcessor



#### 自定义实现

1. 定义一个接口继承Aware接口
2. 定义setX方法
3. 写一个BeanPostProcessor实现
4. 改写其中postProcessorsBeforeInitialization方法


```

public interface MyAware extends Aware {
    void setFlag(Flag flag);
}


@Component
public class MyAwareProcessor implements BeanPostProcessor {

    private final ConfigurableApplicationContext configurableApplicationContext;

    public MyAwareProcessor(ConfigurableApplicationContext configurableApplicationContext) {
        this.configurableApplicationContext = configurableApplicationContext;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (bean instanceof Aware) {
            if (bean instanceof MyAware) {
                ((MyAware) bean).setFlag((Flag) configurableApplicationContext.getBean("flag"));
            }
        }
        return bean;
    }
}

使用

@Component
public class ResultCommandLineRunner implements CommandLineRunner, EnvironmentAware, MyAware {

    private Environment env;

    private Flag flag;

    @Override
    public void run(String... args) throws Exception {
        System.out.println(env.getProperty("mooc.defalut.name"));
        System.out.println(env.getProperty("mooc.active.name"));
    }

    @Override
    public void setEnvironment(Environment environment) {
        env = environment;
    }

    @Override
    public void setFlag(Flag fla) {
        flag = fla;
    }
}

```