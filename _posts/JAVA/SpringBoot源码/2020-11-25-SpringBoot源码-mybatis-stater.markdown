---
layout:     post
title:      "SpringBoot源码系列之Mybatis Starter解析"
subtitle:   ""
date:       2019-01-18 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

### 使用

- pom.xml文件引入

```
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.39</version>
</dependency>
```

- maven-plugin引入

```
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.3.5</version>
    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>
    </dependencies>
</plugin>
```

- resource目录下新建generatorConfig.xml配置文件

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
    <!-- context 是逆向工程的主要配置信息 -->
    <!-- id：起个名字 -->
    <!-- targetRuntime：设置生成的文件适用于那个 mybatis 版本 -->
    <context id="default" targetRuntime="MyBatis3">

        <!--optional,旨在创建class时，对注释进行控制-->
        <commentGenerator>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true" />
        </commentGenerator>

        <!--jdbc的数据库连接 8.0.11-->
        <jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"

        <!--jdbc的数据库连接 5.x-->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"



                        connectionURL="jdbc:mysql://localhost:3306/demotest"
                        userId="yp"
                        password="88888,Yp">
            <property name="nullCatalogMeansCurrent" value="true" />

        </jdbcConnection>

        <!--非必须，类型处理器，在数据库类型和java类型之间的转换控制-->
        <javaTypeResolver>
            <!-- 默认情况下数据库中的 decimal，bigInt 在 Java 对应是 sql 下的 BigDecimal 类 -->
            <!-- 不是 double 和 long 类型 -->
            <!-- 使用常用的基本类型代替 sql 包下的引用类型 -->
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>

        <!-- targetPackage：生成的实体类所在的包 -->
        <!-- targetProject：生成的实体类所在的硬盘位置 -->
        <javaModelGenerator targetPackage="com.test.bean"
                            targetProject="src/main/java">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false"/>
        </javaModelGenerator>

        <!-- targetPackage 和 targetProject：生成的 mapper 文件的包和位置 -->
        <sqlMapGenerator targetPackage="mapper"
                         targetProject="src/main/resources">
            <!-- 针对数据库的一个配置，是否把 schema 作为字包名 -->
            <property name="enableSubPackages" value="false" />
        </sqlMapGenerator>

        <!-- targetPackage 和 targetProject：生成的 interface 文件的包和位置 -->
        <javaClientGenerator type="XMLMAPPER"
                             targetPackage="com.test.mapper" targetProject="src/main/java">
            <!-- 针对 oracle 数据库的一个配置，是否把 schema 作为字包名 -->
            <property name="enableSubPackages" value="false" />
        </javaClientGenerator>

        <table schema="" tableName="usertest" >
        </table>
    </context>
</generatorConfiguration>
```

- 配置数据库相关属性

```
server.port=8080
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.type-aliases-package=com.mooc.sb2.bean
mybatis.configuration.map-underscore-to-camel-case=true
```

- 测试

```
@RunWith(SpringRunner.class)
@SpringBootTest(classes = DemoApplication.class)
class DemoApplicationTests {

    @Autowired
    private UsertestMapper mapper;

    @Test
    void contextLoads() {
        Usertest usertest = new Usertest();
        usertest.setAge(Byte.parseByte("1"));
        usertest.setId(1L);
        usertest.setName("ergouzi");
        mapper.insert(usertest);
    }
}
```


### 源码解析


#### 配置类引入原理

引入mybatis-spring-boot-starter jar包，这个jar包又间接引入了
mybatis-spring-boot-autoconfigure jar包，引入此jar包，会读取配置在
META-INF/spring.factories文件中的自动注入类。其中包含两个主要的配置类

1. MybatisLanguageDriverAutoConfiguration 处理注解sql相关数据
2. MybatisAutoConfiguration 处理xmL文件相关数据


- MybatisAutoConfiguration

这个配置类里面，注入了两个关键的Bean

1. SqlSessionFactory.class
    1.1 单个数据库映射关系经编译后的内存镜像
2. sqlSessionTemplate.class (执行数据库操作的工具类)


同时，定义又MapperScannerRegistrarNotFoundConfiguration类。在符合条件的情况下会
(没有MapperFactoryBean类及MapperScannerConfigurer类)引入AutoConfiguredMapperScannerRegistrar
类。加载MapperScannerConfigurer类。在此类中，转化定义的Bean，生成MybatisProxy代理类，后续调用则为
调用其代理类的方法。

扫描mapper接口注册到容器

#### mapper类生成原理

ClassPathMapperScanner->processBeanDefinitions->将beanClass替换成MapperFactoryBean.class
->MapperFactoryBean#getBean->MapperProxy对象

#### mapper类执行原理

调用MapperProxy的invoke方法，实际会去调用MapperMethod的execute方法。此方法会
根据数据库操作类型，调用sqlSession操作


