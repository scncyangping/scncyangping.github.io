---
layout:     post
title:      "SpringBoot源码系列之Spring Profile"
subtitle:   ""
date:       2019-01-14 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 用处

不同的环境，采用不同的配置

一般来说，默认的是使用application.properties文件中的配置。还有一个默认的激活的文件
application-default.properties。

#### 使用

若有多个配置文件，如何配置其中一个为默认配置文件？

application-default.properties
application-defaults.properties

若需指定默认激活的文件。可指定参数spring.profiles.default=defaults

注：此参数配置不能配置在application.properties文件中。可在命令行指定，如：

--spring.profiles.default=defaults


如何激活某个文件？

```
spring.profiles.active=xxx
```

注：spring.profiles.active与default互斥

如何激活某几个文件？

在application.properties中指定

```
spring.profiles.include=xxx,yyy
```

如何修改文件的前缀？

--spring.config.name=xxx

注：也不能定义相关配置文件内

#### 原理解析

前面说到，当系统环境准备好过后，会像消息队列发布相关事件。监听此事件的监听器有很多。如：
解析spring.application.json参数的监听器，监听springCloud相关的监听器等。其中也还有处理
文件相关的监听器，ConfigFileApplicationListener。这里主要处理一下其原理

onApplicationEvent->postProcessEnvironment->addPropertySources


1. 首先添加一个随机属性集到环境变量参数中
2. 调用监听器的load方法，处理文件类容

load流程：

1. 加载 initializeProfiles()
构建profiles集合 ->添加null对象到集合(目的是用于占位，若后续判断此集合只有一个对象，就去添加默认的配置文件)
->添加激活的profile到profiles(spring.profiles.include、spring.profiles.active指定)->判断集合profiles大小是否
为1，若是则添加默认的profile合同否则，返回对应的profiles

这也就解释了，spring.profiles.default不能定义在分散文件里面，定义在文件里面此时是没有解析的。就不会将其加载到激活的文件集里面。
可以定义在其他如命令行配置集里面

2. 读取addProfileToEnvironment()

    2.1 获取文件配置路径(getSearchLocations()可活动配置路径如：classpath:/,classpath:/config/,file:./,file:./config/)
        这里可以用spring.config.location、spring.config.additional-location配置扫描路径
    2.2 获取文件名(getSearchNames()获取) 这里可以用spring.config.name指定文件名(同样，这个定义需要定义在分散文件之外)，若没有，则使用默认的：application
    2.3 调用加载器(propertySourceLoaders)，加载文件。有两个加载：PropertiesPropertySourceLoader、YamlPropertySourceLoader
3. load
    3.1 读取application-profile.xx文件，若存在，loadDocuments读取文件属性，将文件内激活的profile添加到profiles当中
        将文件类定义的属性放入loaded中

4. 调用addLoadedPropertySources
    4.1 获取environment的propertySource集合对象destination
    4.2 遍历loaded集合，将集合内属性添加到destination中



```
public void load() {
    this.profiles = new LinkedList<>();
    this.processedProfiles = new LinkedList<>();
    this.activatedProfiles = false;
    this.loaded = new LinkedHashMap<>();

    // 1. 判断配置项是否有spring.profiles.active、spring.profiles.include配置的属性
    //    若有，将其配置生成profile对象，添加到激活的文件列表 activeProfiles。
    // 2. 将上一步生成的profile集合进行去重复，若添加了就去掉，并且，判断是否是默认的配置集，若是则去掉
    //    (这里就体现了active配置与默认配置互斥的说法)
    // 3. 根据上述获取到的文件名，进行加载，加载路径为classpath:/,classpath:/config/,file:./,file:./config/下的文件
    initializeProfiles();

    while (!this.profiles.isEmpty()) {
        Profile profile = this.profiles.poll();
        if (profile != null && !profile.isDefaultProfile()) {
            addProfileToEnvironment(profile.getName());
        }
        // 处理获取到的文件，若其中有指定新的文件，如：spring.profiles.include参数指定的数据，将其添加进processedProfiles
        load(profile, this::getPositiveProfileFilter, addToLoaded(MutablePropertySources::addLast, false));
        this.processedProfiles.add(profile);
    }
    // 在此获取相关文件属性，并加载
    resetEnvironmentProfiles(this.processedProfiles);
    load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
    // 将获取到的属性之，添加到系统环境中，完成参数读取
    addLoadedPropertySources();
}
```

#### 总结

问题
1. SpringBoot属性配置方式有哪些?
17种
2. 介绍下Spring Aware的作用及常见的有哪些?
3. Spring Aware注入原理?
4. Environment对象如何加载属性集的?
5. 部分属性集如spring.application.json什么时候加载的?
6. Spring Profile配置方式有哪些注意事项?
7. Spring Profile处理逻辑?
