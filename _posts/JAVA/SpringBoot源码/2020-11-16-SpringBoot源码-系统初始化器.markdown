---
layout:     post
title:      "SpringBoot源码系列(三)系统初始化器实战"
subtitle:   ""
date:       2020-11-16 14:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

### 概要

- 系统初始化器介绍
- SpringFactoriesLoader介绍
- 系统初始化器原理解析


#### 系统初始化器介绍

系统初始化器介绍在SpringBoot中的类名是ApplicationContextInitializer。它是Spring容器
刷新之前执行的一个回调函数。它能够向SpringBoot容器中注册属性。可以通过继承接口自定义实现。
也就是说，它其实是SpringBoot开放的能够自定义注册属性的工具。

实现方式

1. 实现ApplicationContextInitializer,并添加自定义属性
```
@Order(1)
public class FirstInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        Map<String, Object> map = new HashMap<>();
        map.put("key2", "value2");
        MapPropertySource mapPropertySource = new MapPropertySource("secondInitializer", map);
        environment.getPropertySources().addLast(mapPropertySource);
        System.out.println("run secondInitializer");
    }
}
```

2. 注册到Spring容器

```
三种方式

第一种：
1. resources目录下面创建META-INF目录
2. 创建spring.factories文件
3. 添加属性 ： org.springframework.context.ApplicationContextInitializer=com.mooc.sb2.initializer.FirstInitializer


第二种：

在启动类中使用

    SpringApplication application = new SpringApplication(Sb2Application.class);
    application.addInitializers(new FirstInitializer());
    application.run(args);

第三种：
直接在application.properties配置文件中配置

context.initailizer.classes=com.mooc.sb2.initializer.FirstInitializer

```

3. 使用

```
@Component
public class TestService implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    public String test() {
        return applicationContext.getEnvironment().getProperty("key3");
    }

}
```

#### 总结

- 三种自定义系统初始化器都需要实现ApplicationContextInitializer接口
- 客定义多个初始化器，用Order确定其执行顺序，Order值越小，越先执行(注意：application.properties中定义的优先于其他方式)
