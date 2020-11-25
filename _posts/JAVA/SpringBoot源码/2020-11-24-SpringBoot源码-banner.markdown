---
layout:     post
title:      "SpringBoot源码系列之banner解析"
subtitle:   ""
date:       2019-01-09 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 设置banner的几种方式

1. 默认banner: 什么都不用管
2. 文字banner:
    2.1 resources目录下直接新建banner.txt文件
    2.2 配置参数 spring.banner.location=banner124.txt
3. 图片banner(gif｜png|jpg):
    2.1 resources目录下直接新建banner.jpg文件
    2.3 配置参数 spring.banner.image.location=favorite.jpg
4. 设置兜底图案

```
// 设置兜底banner
SpringApplication springApplication = new SpringApplication(Sb2Application.class);
springApplication.setBanner(new ResourceBanner(new ClassPathResource(
        "banner_bak.txt"
)));
springApplication.run(args);
```
5. 关闭banner

spring.main.banner-mode=off

#### 设置banner的原理

方法的入口在run方法中

```
Banner printedBanner = printBanner(environment);
```

#### 主要步骤 获取banners

```
private Banner getBanner(Environment environment) {
    Banners banners = new Banners();
    // 获取图片banner
    banners.addIfNotNull(getImageBanner(environment));
    // 获取文字banner
    banners.addIfNotNull(getTextBanner(environment));

    if (banners.hasAtLeastOneBanner()) {
        return banners;
    }
    // 判读是否有兜底banner
    if (this.fallbackBanner != null) {
        return this.fallbackBanner;
    }
    // 都没有，返回默认banner，定义在springboot默认jar包里面
    return DEFAULT_BANNER;
}
```

- 获取banner，添加到banners变量

1. 通过 getImageBanner方法获取图片banner。
    1.1 在此方法中，会获取spring.banner.image.location指定路径上的文件
    1.2 图片格式三种 gif、png、jpg
    1.3 默认的图片文件名为 banner.jpg、banner.png、banner.gif

2. 通过getTextBanner方法获取文字banner
    1.1 可以通过spring.banner.location指定
    1.2 默认banner.txt

- 判断banners是否为空

1. 若不为空，返回banners
2. 若为空，判读是否设置了兜底banner
    2.1 若设置了，返回兜底banner
    2.2 没设置，返回默认banner


获取到多个banners都会返回，后续会依次打印

#### 主要步骤 打印banner

- 默认输出

1. 先输出banner指定内容
2. 获取version信息
3. 文本内容前后对齐
4. 文本内容染色
5. 输出文本内容

- 文字banner输出

1. 循环获取到的文字banners
2. 可以通过spring.banner.charset指定字符集
3. 获取文本内容
4. 替换占位符(比如可定义 ${test}参数，在配置文件中指定test=123,就会替换banner文件中的参数)
5. 输出文本内容

- 图片banner输出

1. 可以通过spring.banner.image.* 设置图片属性
2. 读取图片文件流
3. 输出图片内容


