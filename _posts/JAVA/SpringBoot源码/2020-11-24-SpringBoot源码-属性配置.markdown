---
layout:     post
title:      "SpringBoot源码系列之属性配置方式"
subtitle:   ""
date:       2019-01-11 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 属性配置17中方式

优先级逐级下降
若前一优先级配置了，后续级别配置的将不起作用

1. Devtools全局配置
2. 测试环境@TestPropertySource注解
3. 测试环境properties属性
4. 命令行参数
5. SPRING_APPLICATION_JSON属性
6. ServletConfig初始化参数
7. ServletContext初始化参数
8. JNDI属性
9. JAVA系统属性
10. 操作系统环境变量
11. RandomValuePropertySource随机值属性
12. jar包外的application-{profile}.properties
13. jar包内的application-{profile}.properties
14. jar包外的application.properties
15. jar包内的application.properties
16. @PropertySource绑定配置
17. 默认属性

除了上述方式添加参数意外，还可以在系统初始化器、或者监听器准备好了过后，像容器里面添加属性


.properties 文件的优先级高于.yml文件

#### 配置实现

```
// RandomValuePropertySource随机值属性
mooc.avg.age=${random.int[20,30]}

// JAVA系统属性
mooc.website.path=${PATH}

// ServletConfig初始化参数、ervletContext初始化参数
一般通过
server.xxxxx = xxxx 来配置

// SPRING_APPLICATION_JSON属性
在启动的时候添加参数:
--SPRING_APPLICATION_JSON={\"test.param.one\":\"test000002\"}

// 命令行参数
启动的时候直接指定
--test.param.one=test000002

// 测试环境properties属性
@SpringBootTest(properties = {"test.param.one=test000002"})

// 测试环境@TestPropertySource注解
@TestPropertySource({demo.properties})
```


- 默认属性配置

```
启动类当中设置默认数据

SpringApplication springApplication = new SpringApplication(Sb2Application.class);
Properties properties = new Properties();
properties.setProperty("test_param_one","test_0000001");
springApplication.setDefaultProperties(properties);
springApplication.run(args);

---------------------------

// 使用方式demo
@Component
public class ResultProperties implements CommandLineRunner, EnvironmentAware {
    private Environment environment;
    @Override
    public void run(String... args) throws Exception {
        System.out.println(environment.getProperty("test_param_one"));
    }
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}

```

- @PropertySource绑定

```
@SpringBootApplication
@MapperScan("com.mooc.sb2.mapper")
@PropertySource(value = {"demo.properties","demo2.properties"})
public class Sb2Application {
    ...
}
```