---
layout:     post
title:      "SpringBoot源码系列之SpringBoot Starter解析"
subtitle:   ""
date:       2019-01-17 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 基础解析

引入具体配置类
@EnableConfigurationProperties(xxxx.class)


##### conditional注解

含义：基于条件的注解
作用：根据是否满足某一个特定条件来决定是否创建某个特定的Bean
意义：SpringBoot实现自动配置的关键基础能力


常用的conditional注解

- @ConditionalOnBean
- @ConditionalOnMissingBean
- @ConditionalOnClass
- @ConditionalOnMissingClass
- @ConditionalOnWebApplication
- @ConditionalOnNotWebApplication
- @ConditionalOnProperty 存在某个特定属性
- @ConditionalOnJava 处于某个特定版本

自定义conditional注解的实现

1. 实现一个自定义注解并且引入Conditional注解
2. 写一个接口实现Condition接口重写matches方法，符合条件返回true
3. 在自定义注解引入Condition接口实现类

```

@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(MyCondition.class)
public @interface MyConditionAnnotation {

    String[] value() default {};

}



public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String[] properties = (String[])metadata.getAnnotationAttributes("com.mooc.sb2.condi.MyConditionAnnotation").get("value");
        for (String property : properties) {
            if (StringUtils.isEmpty(context.getEnvironment().getProperty(property))) {
                return false;
            }
        }
        return true;
    }
}

使用
@Component
//@ConditionalOnProperty("com.mooc.condition")
//@MyConditionAnnotation({"com.mooc.condition1", "com.mooc.condition2"})
public class A {

}
```

#### starter
一种可插拔的插件。与jar包的区别在于：starter能实现自动配置。比如：引入一个mybatis的jar
包，引入配置基本参数过后，还需要将其注入到具体容器中。使用mybatis-stater能够实现自动配置。
大幅提升开发效率。


常用的starter有：

spring-boot-starter-web: 构建Web、Restful框架、SpringMVC、默认嵌入式容器Tomcat
spring-boot-starter-data-redis: 通过Spring Data Redis,Jedis client使用Redis
spring-boot-starter-aop: 通过Spring AOP、Aspect面向切面编程
spring-boot-starter: Core starter 包括自动配置支持、logging和YAML
spring-boot-starter-mail: 使用Java mail、Spring email发送支持

自定义一个stater步骤

1. 新建SpringBoot项目
2. 引入spring-boot-autoconfigure
3. 编写属性源及自动配置类
4. 在spring.factories中添加自动配置类实现
5. maven打包


#### starter原理解析

1. 启动类上@SpringBootApplication
2. 引入AutoConfigurationImportSelector
3. 在ConfigurationClassParser中处理
4. 获取spring.factories中EnableAutoConfiguration实现


配置类过滤

1. @ConditionalOnProperty
2. @OnPropertyCondition
3. @getMatchOutcome
4. 遍历注解属性集判断environment中是否含有并值一致
5. 返回对比结果


#### 总结

- 介绍下熟悉的condition注解
- 回答下conditional注解的原理
- SpringBoot starter有什么作用？熟悉哪些？
- 自定义搭建starter?
- starter中的自动配置类是如何被引入到框架中的？
    1. spring.factories
- starter中自动配置类生效的原理？
    1. 引入
    2. 过滤

