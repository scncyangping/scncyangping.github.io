---
layout:     post
title:      "SpringBoot源码系列之启动加载器解析"
subtitle:   ""
date:       2019-01-10 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### 什么是启动加载器

如果有一段代码，需要在SpringBoot框架启动后立即执行，就需要运用行一个启动加载器。

#### 实现方式

两种实现方式

第一种实现CommandLineRunner重写run方法

```
@Component
@Order(1)
public class FirstCommandlineRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println("\u001B[32m >>> startup first runner<<<");
    }
}
```

第二种实现ApplicationRunner重写run方法

```
@Component
@Order(1)
public class FirstApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("\u001B[32m >>> startup first application runner<<<");
    }
}
```

Order排序 order值相同ApplicationRunner优先


#### 启动加载器原理

启动加载器的启动，在SpringBoot容器启动过后(run方法中的最后一步)

callRunners(context, applicationArguments);


主要逻辑
```
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    // 获取所有ApplicationRunner的实现类
    // 根据类型获取当前容器中的实例
    // 有一个判断容器是否启动的参数 this.active
    // 在SpringApplication的run方法中的refreshContext方法内的refresh方法中的prepareRefresh方法中设置
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    // 获取所有CommandLineRunner的实现类
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```



1. 添加ApplicationRunner实现到runners
2. 添加CommandLineRunner实现到runners集合
3. 对runner集合排序
4. 遍历runner集合依次调用实现类run方法


两种实现方式的差异
1. 执行优先级不一样
2. run方法入参不一样，实现ApplicationRunner是ApplicationArguments类型参数，实现CommandLineRunner是String变量参数

相同点：
1. 调用点一样
2. 实现方法名一样




#### 计时器


主要对象
```
public class StopWatch {

	// ID
	private final String id;
    // 是否保存执行过的任务
	private boolean keepTaskList = true;
    // 执行过的任务列表
	private final List<TaskInfo> taskList = new LinkedList<>();
    // 任务开始时间
	private long startTimeMillis;
    // 任务名称
	@Nullable
	private String currentTaskName;

    // 最后执行任务名称
	@Nullable
	private TaskInfo lastTaskInfo;
    // 任务总数
	private int taskCount;
    // 任务总耗时
	private long totalTimeMillis;
}
```

1. 构建StopWatch对象： StopWatch stopWatch = new StopWatch();
2. 开始 stopWatch.start()
    2.1 业务校验，当前是否有执行中的任务
    2.2 保存任务名
    2.3 记录当前系统时间
3. 结束 stopWatch.stop()
    3.1 业务校验，当前是否有执行中的任务
    3.2 计算耗时
    3.3 将当前任务添加到任务列表中
    3.4 任务执行数加一
    3.5 清空当前任务