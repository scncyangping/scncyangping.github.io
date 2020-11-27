---
layout:     post
title:      "SpringBoot源码系列之配置类解析"
subtitle:   ""
date:       2019-01-15 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---


#### 流程

执行入口
SpringBootApplication#run->refreshContext->refresh->invokeBeanFactoryPostProcessors调用

ConfigurationClassPostProcessor类的processConfigBeanDefinitions方法

此方法中的循环即时对配置类解析的重要逻辑

```
do {
    parser.parse(candidates);
    ...
   }while (!candidates.isEmpty());
```

1. 首先调用ConfigurationClassParser#parse方法，完成对注解的处理
    1.1 在此parse方法中，最重要的业务逻辑在doProcessConfigurationClass方法中
    1.2 首先判断是否是@Component注解，若是，判断其内部类是否是配置类，若是，递归执行上述解析方法
    1.3 其次，解析@PropertySources注解。若包含此注解，解析其配置的值，读取配置文件并解析，并添加到环境变量当中
    1.4 解析@ComponentScans注解。先解析注解中basePackages配置的值再解析basePackageClasses配置的值获取其文件所在包地址。
        若都没配置，则默认取注解类所在包的地址
    1.5 处理Import
        1.5.1. 判断类是否是ImportSelector.class & DeferredImportSelector.class
        1.5.2. 处理以上两个接口实现selectImports返回的类名数组
        1.5.3. DeferredImportSelector接口调用优先级低于其他接口
        1.5.4. 处理ImportBeanDefinitionRegistrar实现中注册的Bean
        1.5.5. 处理@Import(A.class)

    1.6 处理ImportSource
        1.6.1. @ImportResource("xyz.xml")
        1.6.2. 将注解属性值放入importedResources中
        1.6.3. 后续loadBeanDefinitionsForConfigurationClass中加载定义的Bean

    1.7 处理@Bean注解

2. 调用ConfigurationClassParser#validte方法
3. 读取BeanMethod注册BeanDefinition
4. 处理新引入的BeanDefinition


#### 重要逻辑

注解@PropertySource处理

1. @PropertySource({demo.properties})
2. 遍历指定路径，替换占位符，加载资源
3. 将资源添加到environment中


ComponentScan处理

1. @ComponentScan(basePackages={"pkgA","pkgB},
    basePackageClasses={A.class,B.class})
2. 没设置骚麦哦路径的话使用配置累所在路径
3. 过滤顺序：excludeFilters->includeFilters->false

Import注解处理
1. 判断类是否是ImportSelector.class & DeferredImportSelector.class
2. 处理以上两个接口实现selectImports返回的类名数组
3. DeferredImportSelector接口调用优先级低于其他接口
4. 处理ImportBeanDefinitionRegistrar实现中注册的Bean
5. 处理@Import(A.class)

ImportResource注解处理

1. @ImportResource("xyz.xml")
2. 将注解属性值放入importedResources中
3. 后续loadBeanDefinitionsForConfigurationClass中加载定义的Bean

BeanMethod处理

```
@Configuration
public class BeanConfiguration{
    // 处理此种返回的Bean
    @Bean("dog")
    Animal getDog(){
        return new Dog();
    }
}

// 接口默认方法的处理同上
public interface BeanConfiguration{
    // 处理此种返回的Bean
    @Bean("dog")
    default Animal getDog(){
        return new Dog();
    }
}
```



1. 配置类是什么？起到什么作用？
2. 常用的配置注解？
3. 介绍下SpringBoot框架对配置类的一个处理流程？
4. 配置类的处理一般包括哪些内容？
