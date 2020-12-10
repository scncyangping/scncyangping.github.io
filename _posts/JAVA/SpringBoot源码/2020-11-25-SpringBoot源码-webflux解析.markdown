---
layout:     post
title:      "SpringBoot源码系列之webflux解析"
subtitle:   ""
date:       2019-01-19 12:00:00
author:     "YaPi"
header-img: ""
tags:
    - SpringBoot
---

#### webflux理论

传统的Spring MVC是采用的同步阻塞式IO模型，即是每一个请求，容器都会新开一个线程去处理。
在处理完成之前，不会接收其他的请求。

webflux是一个异步阻塞式IO模型。当容器内发生了一个线程密集型的请求，就会将这些请求交给
一个worker线程组去处理。这样，这个线程本身就可以去处理另外的请求，达到容器只需使用少量
线程就可处理大量的请求。

可以提升吞吐量和伸缩性，但是本身线程处理的时间不会减少，还是会等到worker线程组执行完成。

可以运用在IO密集型，即磁盘IO密集，网络IO密集的服务场景

比如：微服务网关(Spring Cloud gateWay就使用了此模型)

webflux与springmvc异同点

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/springBoot%E6%BA%90%E7%A0%81/webflux%E4%B8%8Espringmvc%E7%9A%84%E5%BC%82%E5%90%8C%E7%82%B9.jpg)

#### webflux技术依赖

- Reactive Streams : 反应式编程标准和规范

1. 具有处理无限数量数据的能力
2. 按序处理数据
3. 异步非阻塞的传递数据
4. 必须实现非阻塞的背压(如果数据源发送数据的数量大于消费端消费数据的数量，那么消费端会发送
消息到数据源，减少发送数量或者直接停止发送)

它定义了一系列api规范

1. publisher: 数据发布者

```
public interface Publisher<T>{
    public void subscribe(Subscriber<? super T> s);
}
```

2. subscriber: 数据订阅者

```
public interface Subscriber<T> {

    public void onSubscribe(Subscription s);


    public void onNext(T t);


    public void onError(Throwable t);


    public void onComplete();
}

```

3. subscription: 订阅信号

```
public interface Subscription {

    public void request(long n);

    public void cancel();
}
```

4. processor: 处理器(包含了发布者和订阅者的混合体)

- Reactor : 基于Reactive Streams的反应式编程框架
- WebFlux : 以Reactor为基础实现Web领域的反应式编程



#### Reactor框架

Spring公司开发，符合Reactive Streams规范。侧重于server端的响应式框架。由
两个模块组成，reactor-core和reactor-ipc

java原油的异步编程方式

Callbacks: 异步方法采用一个callback作为参数，当结果出来后，调用这个callback.
例如：swings的EventListener

局限性：回调地狱

Futures: 异步方法返回一个Future<T>，此时结果并不是可以立即拿到，需处理结束后才可用

局限性：多个Future组合不易，调用Future#get时仍然会阻塞，缺乏对多个值以及进一步的出错
处理



##### Reactor的publisher
实现：Flux Mono

Flux: 代表一个包含0...N个元素的响应式序列
Mono: 代表一个包含0/1个元素的结果

```
// 创建及调用方式
void test(){
    Flux<Integer> flux = Flux.just(1,2,3,4,5,6);
    Flux<Integer> arrayFlux = Flux.fromArray(new Integer[]{1,2,3,4,5,6});
    Flux<Integer> streamFlux = Flux.fromStream(Stream.of(1,2,34,5,6));
    Flux<Integer> listFlux = Flux.fromIterable(Arrays.asList(1,2,3));
    Flux<Integer> fluxFlux = Flux.from(flux);
    Mono<Integer> mono = Mono.just(1);

    // 消费 不做任何事
    flux.subscribe();
    // 消费
    flux.subscribe(System.out::println);
    // 加上出错处理
    flux.subscribe(System.out::println,System.err::println);
    // 完成处理
    flux.subscribe(System.out::println,System.err::println,
            ()->{System.out.println("完成处理");});
    // 指定处理数量
    flux.subscribe(System.out::println,System.err::println,
            ()->{System.out.println("完成处理");},
            subscription -> subscription.request(3));

    flux.subscribe(new DemoSubscribe());
}

class DemoSubscribe extends BaseSubscriber<Integer> {
    @Override
    protected void hookOnSubscribe(Subscription subscription) {
        System.out.println("subscribe");
        subscription.request(1);
    }

    @Override
    protected void hookOnNext(Integer value) {
        if (value == 3){
            cancel();
        }
    }
}
```
#### reactor操作符
需要对数据源做一些处理，就需要用到reactor操作符

- map操作符

```
// 对数据进行某些操作
flux.map(i -> i * 3).subscribe(System.out::println);

arrayFlux.flatMap(i -> flux).subscribe(System.out::println);
```

- filter操作符

```
// 过滤掉一部分数据
listFlux.filter(i ->i>3).subscribe(System.out::println);
```

- zip操作符

```
// 将两个flux里面的数据进行相加操作输出
Flux.zip(flux,listFlux).subscribe(zip -> System.out.println(zip.getT1() + zip.getT2()));
```

#### reactor和java8 stream区别

- 形似而神不似
- reactor: push模式，服务端推送数据客户端，对应的异步的反应式程序
- stream: pull模式，客户端主动向服务端请求数据，对应的是同步的程序

#### reactor线程模型

- Schedulers.immediate(): 当前线程
- Schedulers.single(): 可重用的单线程
- Schedulers.elastic(): 弹性线程池
- Schedulers.parallel(): 固定大小线程池
- Schedulers.fromExecutorService(): 自定义线程池

如何指定线程

- 使用publishOn指定

```
flux.map(i -> {
        System.out.println(Thread.currentThread().getName()+"-map1");
        return i *3;
    }).publishOn(Schedulers.elastic()).map(
            i ->{
                System.out.println(Thread.currentThread().getName() + "-map2");
                return i /3;
            }
    ).subscribe(it -> System.out.println(Thread.currentThread().getName()+"-"+it));
```

它将上游信号传给下游，同时改变后续的操作符的执行所在线程，直到下一个
publishOn出现在这个链上

- 使用subscribeOn指定

```
flux.map(i -> {
            System.out.println(Thread.currentThread().getName()+"-map1");
            return i *3;
        }).publishOn(Schedulers.elastic()).map(
                i ->{
                    System.out.println(Thread.currentThread().getName() + "-map2");
                    return i /3;
                }
        ).subscribeOn(Schedulers.parallel()).subscribe(it -> System.out.println(Thread.currentThread().getName()+"-"+it));

无输出，主线程执行完成，直接退出

```

作用于向上的订阅链，无论处于操作链的什么位置，它都会影响到源头的线程执行
环境，但不会影响到后续的publishOn

#### webflux实践

- 与SpringMVC结合

```
@RestController
public class DemoController {

    @GetMapping("/demo")
    public Mono<String> demo(){
        return Mono.just("demo");
    }
}
```

注意点：

- 使用springMVC注解
- 使用ServletReq/Resp 切换成ServerReq/Resp
- 返回Mono对象

- webflux

```
// 定义
@Component
public class DemoHandler {

    public Mono<ServerResponse> hello(ServerRequest request){
        return ok().contentType(MediaType.TEXT_PLAIN)
                .body(Mono.just("hello"),String.class);
    }

    public Mono<ServerResponse> world(ServerRequest request){
        return ok().contentType(MediaType.TEXT_PLAIN)
                .body(Mono.just("world"),String.class);
    }

    // time没隔一秒发送数据到客户端
    public Mono<ServerResponse> times(ServerRequest request){
        return ok()
                .contentType(MediaType.TEXT_EVENT_STREAM)
                .body(Flux.interval(Duration.ofSeconds(1))
                           .map(it -> new SimpleDateFormat("HH:mm:ss")
                                   .format(new Date())),String.class);
    }
}

// 路由

@Configuration
public class RouterConfig {
    @Autowired
    private DemoHandler demoHandler;


    @Bean
    public RouterFunction<ServerResponse> demoRouter(){
        return route(GET("/hello"),demoHandler::hello)
                .andRoute(GET("/world"),demoHandler::world)
                .andRoute(GET("/times"),demoHandler::times);
    }
}

```

- SpringMVC处理流程

![avatar](https://blog-1257627424.cos.ap-chengdu.myqcloud.com/springBoot%E6%BA%90%E7%A0%81/SpringMVC%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.jpg)


- WebFlux处理流程

1. DispatcherHandler准备
    1.1 setApplicationContextAware
    1.2 initStrategies
    1.3 获取容器中HandlerMapping及子接口实现
    1.4 获取容器中HandlerAdapter及子接口实现
    1.5 获取容器中HandlerResultHandler及子接口实现

RouterFunctionMapping实例化
因为它实现了InitializingBean,所以在实例化的时候先调用
afterPropertiesSet。然后调用initRouterFunctions方法，然后调用routerFunctions方法。
获取系统中所有的RouterFunction。通过RouterFunction::andOther将对象合并。
返回SameComposedRouterFunction对象。

2. DispatcherHandler#handle
    2.1 构建基于handlerMappings集合的Flux对象
    2.2 通过concatMap将其转换成handler对象
    2.3 取出第一个handler对象，若为空，则抛错
    2.4 调用invokeHandler获得response
        2.4.1 遍历handlerAdapters集合
        2.4.2 依次调用集合元素supports方法
        2.4.3 获得具体实现类调用handle方法
        2.4.4 进入具体url对应处理类请求
        2.4.5 返回Mono<HandlerResult>对象

3. DispatcherHandler#handleResult
    3.1 遍历resultHandlers集合
    3.2 依次调用集合元素supports方法
    3.3 获得具体实现类调用handleResult方法
    3.4 将请求结果信息写入ServerWebExchange对象





若需引入与MongoDB

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>

比在web项目中多一个reactive后缀

使用

@Repository
public interface CityRepository extends ReactiveMongoRepository<City,String>{
}


handler添加

public Mono<ServerResponse> listCity(ServerRequest request){
    return ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(cityRepository.findAll(),City.class);
}


public Mono<ServerResponse> saveCity(ServerRequest request){
    String province = request.pathVariable("province");
    String city = request.pathVariable("city");

    City record = new City();
    record.setProvince(province);
    record.setCity(city);
    Mono<City> mono = Mono.just(record);
    return ok().build(cityRepository.insert(mono).then())
}

```

#### 总结

1. webflux出现的意义
2. webflux与springmvc异同
3. 介绍一下reactive stream规范
4. 介绍一下reactor
5. flux和mono对象的区别，如何创建
    一个是0到N，一个是0或者1
6. 简述下reactor操作符&线程池
7. publishOn和subscribeOn区别
8. reactor和java8 stream的区别
    java8 stream还是一个阻塞的方式，reactor是异步非阻塞的方式
9. RouterFunctionMapping的作用以及何时被加载
    作用是路由url到对应处理的RouterFunction。在DispatcherHandler初始化的时候加载