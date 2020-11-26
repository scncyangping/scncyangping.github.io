---
layout:     post
title:      "SpringBoot源码系列之bean"
subtitle:   ""
date:       2019-01-06 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### IOC思想

将组建对象的控制权从代码本身转移到外部容器。使用的时候在需要引入的类依赖注入使用。
可以达到松耦合、灵活、可维护的好处。

bean的配置方式有两种
1. xml方式，可使用注解@ContextConfiguration("classpath:xxx")指定xml位置. (配置相对集中，清晰，但是复杂度提升)
2. 注解的方式. (配置分散，对象关系不清晰，修改过后需要重新打包部署)


#### xml方式

同时，具有四种构造方法

```
// 定义简单实体
public class Student {

    private String name;

    private Integer age;

    private List<String> classList;

    public Student(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
    ...
}
```


- 无参构造



```
xml配置方式：
<bean id="student" class="com.mooc.sb2.ioc.xml.Student">
        <property name="name" value="zhangsna"/>
        <property name="age" value="3"/>
        <property name="classList">
            <list>
                <value>math</value>
                <value>english</value>
            </list>
        </property>
</bean>

使用@Autowired注入使用

```



- 有参构造
```
xml配置方式：
<bean id="student" class="com.mooc.sb2.ioc.xml.Student">
        <constructor-arg index="0" value="zhangsan"/>
        <constructor-arg index="1" value="13"/>
        <property name="classList">
            <list>
                <value>math</value>
                <value>english</value>
            </list>
        </property>
</bean>
```


- 静态工厂方法



```
定义一个静态工厂方法，传入type，返回对应类型实体


public class AnimalFactory {
    public static Animal getAnimal(String type) {
        if ("dog".equals(type)) {
            return new Dog();
        } else {
            return new Cat();
        }
    }
}


xml配置方式：
<bean name="animalFactory" class="com.mooc.sb2.ioc.xml.AnimalFactory"/>

<bean id="dog" class="com.xxx.sb2.AnimalFacotry" factory-method="getAnimal">
    <constructor-arg value="dog"/>
</bean>

<bean id="cat" class="com.xxx.sb2.AnimalFacotry" factory-method="getAnimal">
    <constructor-arg value="cat"/>
</bean>

```


- 实例工厂方法



```

public class AnimalFactory {
    public  Animal getAnimal(String type) {
        if ("dog".equals(type)) {
            return new Dog();
        } else {
            return new Cat();
        }
    }
}

<bean name="animalFactory" class="com.mooc.sb2.ioc.xml.AnimalFactory"/>

<bean id="dog" factory-bean="animalFactory" factory-method="getAnimal">
    <constructor-arg value="dog"/>
</bean>

<bean id="cat" factory-bean="animalFactory" factory-method="getAnimal">
    <constructor-arg value="cat"/>
</bean>
```


#### 注解配置方式

- @Component声明
- 配置类中使用@Bean



```
@Configuration
public class BeanConfiguration{
    @Bean("dog")
    Animal getDog(){
        return new Dog();
    }
}

注册一个id为dog的Bean

```


- 继承FactoryBean


```
@Component
public class MyCat implements FactoryBean<Animal> {
    @Override
    public Animal getObject() throws Exception {
        return new Cat();
    }

    @Override
    public Class<?> getObjectType() {
        return Animal.class;
    }
}

若同一类型的实例有多个，这时需要用@Qualifier("bean名称指定")

```


- 继承BeanDefinitionRegistryPostProcessor


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

- 继承ImportBeanDefinitionRegistrar


```
public class MyBeanImport implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition();
        rootBeanDefinition.setBeanClass(Bird.class);
        registry.registerBeanDefinition("bird", rootBeanDefinition);
    }
}

使用:

@Import(MyBeanImport.class)
```



