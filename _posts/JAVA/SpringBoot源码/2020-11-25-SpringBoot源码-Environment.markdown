---
layout:     post
title:      "SpringBoot源码系列之Environment属性集添加"
subtitle:   ""
date:       2019-01-13 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 设置流程

入口 SpringApplication.run()方法中

```
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
```

主要业务逻辑

```
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // 根据当前容器环境，实例化相关对象，比如：web环境(SERVLET)、reactive环境(REACTIVE)，普通开发环境等

    // 在web环境中，新建对象的时候会先读取servlet相关属性源配置，有JNDI服务也会读取相关配置
    // servletConfigInitParams、servletContextInitParams、jndiProperties
    // 然后添加上系统的两种属性源 systemProperties,systemEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();


    // 判断是否有默认的配置，有就加上defaultProperties
    // 判断是否有commandLineArgs配置，存在就添加上springApplicationCommandLineArgs，读取相关配置
    // 否则添加上启动参数 args的参数配置
    // 然后添加上配置文件里面定义的参数
    configureEnvironment(environment, applicationArguments.getSourceArgs());

    // 发送环境变量准备好的事件到监听器
    // 当监听器监听到相应事件过后，会实例化EnvironmentPostProcessor实现类对象并调用其postProcessEnvironment方法
    // 比如
    // 1. spring.application.json的属性源的读取就是在一个监听器里面实现的
    // 2. 随机属性源的添加也是在其中一个监听器实现的
    // 3. 如果是Spring Cloud环境，就会添加对应vcap属性集
    // 4. 添加当前系统活动的profile即application-profile.(application|yml)属性集
    listeners.environmentPrepared(environment);


    // 从当前的environment对象中，获取所有以 spring.main开头的属性值，赋予到当前SpringApplication对象中
    bindToSpringApplication(environment);

    // 判断一下当前的环境变量对象是不是符合当前环境(web环境(SERVLET)、reactive环境(REACTIVE)，普通开发环境)
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    // 添加ConfigurationProperties属性集
    ConfigurationPropertySources.attach(environment);
    return environment;
}


对于 @PropertySources()注解指定的属性集合是在ConfigurationClassParser里面获取的
```


```
// getOrCreateEnvironment() 特点
创建环境变量实体主要实现主要有三个对象，主要目的是添加相应的属性集。
StandardServletEnvironment、StandardEnvironment、以及AbstractEnvironment

其中，StandardServletEnvironment继承StandardEnvironment,StandardEnvironment继承AbstractEnvironment

在 new StandardServletEnvironment()这个过程中，首先会运行父类的构造方法。会先运行AbstractEnvironment
的构造方法，在这里初始化 propertySources集合，同时，运行子类StandardServletEnvironment复写的customizePropertySources方法，
加载web相关的属性集配置，servletConfigInitParams、servletContextInitParams、以及JNDI相关的jndiProperties。
同时，在运行完过后，调用super.customizePropertySources()，运行StandardEnvironment
的customizePropertySources,加载系统属性集systemProperties、systemEnvironment


简单的例子：
抽象初始函数

public abstract class AbstractClass {
    public AbstractClass(){
        System.out.println("抽象函数构造方法");
        doSomeTh();
    }
    public void doSomeTh(){};
}

子类1
public class ChildClass extends AbstractClass {
    @Override
    public void doSomeTh() {
        System.out.println("ChildClass -- doSomeTh");
    }
}

子类2
public class Child2 extends ChildClass {
    @Override
    public void doSomeTh() {
        System.out.println("child2 -- doSomeTh");
        super.doSomeTh();
    }
}

运行：
public class main {
    public static void main(String[] args){
        Child2 childClass = new Child2();
    }
}

输出：
抽象函数构造方法
child2 -- doSomeTh
ChildClass -- doSomeTh
```


#### 获取

1. AbstractEnvironment#getProperty
2. PropertySourcesPropertyResolver#getProperty
3. 遍历propertySources集合获取属性


