---
layout:     post
title:      "SpringBoot源码系列之Servlet容器启动解析"
subtitle:   ""
date:       2019-01-16 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 全局流程解析

在SpringBoot2.x之前，只能区分web环境和非web环境。2.x之后，新增了一个REACTIVE环境。

在执行SpringApplication.run()方法的时候，会首先新建一个SpringApplication对象。在此
对象的初始化过程中，会设置容器当前运行环境属于哪一种。具体判断的方法是判断某些类文件是否存在

```
static WebApplicationType deduceFromClasspath(){
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
                    && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
                    && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
                return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
    }
    return WebApplicationType.SERVLET;
}

其中SERVLET_INDICATOR_CLASSES即判断当前环境是否是Servlet环境，若是，则需要包含类：
"javax.servlet.Servlet"
"org.springframework.web.context.ConfigurableWebApplicationContext"
```

当判断好当前环境属于某一种环境过后，在SpringApplication.run()方法中，对其进行创建，其创建的
具体对象，是更具上述所设置的WebApplicationType类型决定的。

```
context = createApplicationContext();

```
创建好过后，会调用refreshContext方法中的refresh方法。在此方法中有一个onRefresh方法，
它是一个空实现，运行时会调用实现类方法。比如：若是Servlet环境,
就会调用ServletWebServerApplicationContext的onRefresh方法中运行，并调用createWebServer
创建容器。

```
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

```

接下来，在createWebServer方法中创建

```
private void createWebServer() {
    WebServer webServer = this.webServer;
    // 获取当前环境ServletContext，若无则新建
    ServletContext servletContext = getServletContext();
    if (webServer == null && servletContext == null) {
        // 会先获取一个WebServerFactory的工厂类
        // 会去寻找类ServletWebServerFactory.class的所有实现
        ServletWebServerFactory factory = getWebServerFactory();
        // 用工厂类创建一个并设置给webServer
        // 默认情况下是一个tomcat服务
        this.webServer = factory.getWebServer(getSelfInitializer());
    }
    else if (servletContext != null) {
        try {
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context", ex);
        }
    }
    // 判断environment中是否存在servletContextInitParams和servletConfigInitParams属性集，若有的话
    // 就对其做一个替换
    initPropertySources();
}
```

创建好了过后，会在refresh方法中调用finishRefresh方法。这个方法也会根据类型调用。比如：若当前是Servlet环境，
会调用ServletWebServerApplicationContext的finishRefresh方法。然后调用其中的startWebServer方法，调用
webServer.start()方法。启动成功过后，会在finishRefresh中调用publishEvent方法，
发布一个ServletWebServerInitializedEvent事件。


#### Web容器工厂类加载解析

上述提到创建ServletWebServerFactory的方法

```
ServletWebServerFactory factory = getWebServerFactory();

```

直接通过类型获取的容器类它的实现类，那他是什么时候加载进去的呢？

在注解@SpringBootApplication中有一个注解叫做@EnableAutoConfiguration。它包含一个子注解
@Import(AutoConfigurationImportSelector.class)。
在此后初始化过程中，通过此注解配置，就会去加载spring.factories文件中EnableAutoConfiguration.class的实现类。
其中有一个叫做ServletWebServerFactoryAutoConfiguration。此类有一个注解@Import。
它引入类很多的类，其中就有ServletWebServerFactoryConfiguration.EmbeddedTomcat.class。
这个类也是一个配置类，加载的时候也会加载此类中的配置。此类，有一个@Bean注解，定义返回了TomcatServletWebServerFactory。

```
@Configuration
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
public static class EmbeddedTomcat {
    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory();
    }

}
```

#### Web容器个性化配置解析
我们可以在配置文件中，对容器的基础属性做任意修改，比如，修改端口号。那么这些配置是在什么时候生效的呢？


上述提到，在web容器创建的过程中，会先获取一个工厂类，然后调用AbstractBeanFactory的getBean方法。在这个
方法中完成创建Bean及初始化，初始化的过程中会去调用执行包含该Bean的Aware、BeanPostProcessor方法。其中
WebServerFactoryCustomizerBeanPostProcessor这个类定义了当Bean的类型是WebServerFactory实现类时，需要
执行的方法。而ServletWebServerFactory是其实现类，所以就会执行这个类的postProcessBeforeInitialization方法。

在此方法中，会去获取容器内部所有WebServerFactoryCustomizer.class的实现类。其中有一个叫
ServletWebServerFactoryAutoConfiguration(即上面引入EmbeddedTomcat.class那个类)。它初始化了一个配置类ServerProperties。
它就解析了属性中以server.开头的属性。并在初始化过程中，将其注入到容器工厂类当中。

再比如TomcatWebServerFactoryCustomizer.class就定义类server.tomcat.*的属性。


#### 总结
启动流程解析

1. 首先在SpringApplication的新建方法当中判断当前容器的环境，判断的方法是根据是否
存在某些确定的类来实现的。
2. 然后在SpringApplication的run方法中，refreshContext()->refresh()->
onRefresh()。onRefresh方法交由具体的实现类复写。根据当前环境配置，如，Servlet环境
就会去调用创建Servlet容器
3. 创建的过程会去获取容器中所有ServletWebServerFactory的实现类。然后配置相关属性
4. 创建完成过后，用工厂类获取一个实例，然后启动，发送相应事件。启动流程完成。


工厂类加载解析

1. 首先启动类引入了@EnableAutoConfiguration注解，这个注解又引入了@Import(AutoConfigurationImportSelector.class)。
这个配置会在ConfigurationClassParse里面处理。在这个处理过程当中就回去引入SpringFactoriesLoader加载的META-INFO/spring.factories
文件中的配置EnableAutoConfiguration的实现。
2. 其中有一个叫做ServletWebServerFactoryAutoConfiguration类，其又引入了一个EmbeddedTomcat类。这个配置类又用Bean注解定义返回了
一个ServletWebServerFactory的实现类TomcatServletWebServerFactory。所以，在getWebServerFactory()方法中，可以通过查找ServletWebServerFactory
的实现类，获取容器工厂类

属性注入流程

1. 配置web容器属性如：server.xxx=xyz
2. 注入到ServerProperties
3. 自动配置会导入WebServerFactoryCustomizer实现类
4. ServerProperties成为其实现类的属性
5. 在getWebServerFactory获取具体web服务工厂类时，会对具体实现类
调用doGetBean进行初始化
6. 初始化过程中，遍历BeanPostProcessor实现对bean处理
7. 进入WebServerFactoryCustomizerBeanPostProcessor实现方法,调用
postProcessBeforeInitialization
8. 调用getCustomizers(),获取所有WebServerFactoryCustomizer实现类，
然后依次调用实现类的customize方法进行定制处理

比如：ServletWebServerFactoryCustomizer#customize()。构建PropertyMapper
工具类，将ServerProperties定义设置到具体属性，然后再赋给WebServerFactory。
在创建WebServer时容器就有了具体的属性。


#### 问题

1. 描述一下Servlet容器启动流程
2. 介绍下AutoConfigurationImportSelector的作用
通过SpringFactoriesLoader,将classpath路径下META-INFO/spring.factories文件中的
EnableAutoConfiguration的实现类，给引入到容器当中。如servlet相关的类，这些类又引入了
容器工厂类，容器配置类等
3. AutoConfigurationImportSelector何时被处理？
在SpringBoot启动过程中，ConfigurationClassParse会对配置类做一些解析。其中有一步骤
就是处理@Import注解导入的类。这个时候就会对这个类做一些处理
4. 为什么SpringBoot启动默认的是tomcat容器？
相应文件是否存在确定哪种环境。确定环境后引入web工厂类。此配置类引入了tomcat、jetty等。但是
只有tomcat符合条件(某些特定类是否不存在或不存在：@ConditionalOnClass、@ConditionalOnMissingBean判断)
5. 常见的web容器自定义配置参数？
server.port、server.address、server.tomcat.basedir
6. 讲解下自定义配置参数生效的原理



