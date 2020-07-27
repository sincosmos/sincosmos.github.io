---
layout:     post
title:      Spring WebFlux 概述
subtitle:   学习笔记
date:       2020-01-14
author:     sincosmos
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Reactive, Spring WebFlux, WebClient
---  

## 什么是 Reactive
Reactive Programming 一种异步编程模型，事件-响应模式，例如网络组件响应 I/O 事件。其核心实现可以简单理解为，在应用主线程之外，有一个线程非阻塞式地等待数据，对这些数据感兴趣的模块注册到这个线程的 Observer 集合中，数据到达时，该线程将数据 push 给集合中每个 Observer。RP 在上述思想的基础上，增加解决 non-blocking backpressure 问题的能力。  
Backpressure 是一种现象：当数据的生产者生产数据的速度大于下游的消费速度，那么下游就需要 buffer （大量）数据乃至最终 buffer 容量到上限后，消费端需要丢弃部分数据，这种现象即为 backpressure。  
一般的 Push 模型中，例如 Java 中的 Futrue, CompletableFuture, 生产者不关心消费者的处理速度，生产者发送给消费者的数据，由消费者自行决定是否丢弃。RP 模型中，通过 backpressure 反馈使生产者“感受”到消费者的消费能力，从而使生产者只发送合适的数据量给消费者。 

## Java 与 Reactive Programming 
在 Java 领域，Reactive Programming 的基础组件是 reactive-streams.jar 包。而从 jdk 1.9 开始，该 jar 包的功能被集成到 java.util.concurrent.Flow 类中。主要包括 4 个简单的接口的定义。  
- `Publisher<T>` 组件负责生产、发布类型为 T 的数据元素流，该流可以是有界的，也可以持续不断无界的流。它提供一个 `subscribe(Subscriber<? super T> s)` 方法，供消费者订阅它产生的数据；
- `Subscriber<T>` 组件订阅 `Publisher`，`Subscriber` 主要提供四个回调方法：`onSubscribe(Subscription s), onNext(T t), onError(Throwable t), onComplete()`。  
  `publisher.subscribe(subscriber)` 调用成功后，`subscriber` 会回调执行 `subscriber.onSubscribe(Subscription s)` 方法。但如果在该回调函数中没有执行 `subscription.request(long n)`，即使订阅成功，`subscriber` 也不会收到 `publisher` 的任何数据（Spring 默认的 Subscriber 基类，默认执行 request(Long.MAX_VALUE) 操作）；  
  `subscriber` 收到数据后会回调执行 `subscriber.onNext(T t)` 方法，去处理处理数据；
  类似地，`onError(Throwable t), onComplete()` 也是在接到相应的数据信号后，`subscriber` 需要执行的回调函数。
- `Subscription` 组件代表 `publisher` 和 `subscriber` 之间的一次性连接，它提供 `request(long n), cancel()` 方法，作为 backpressure 的反馈能力的体现，前者限定本次订阅预期接收的数据的数量，后者可以使 `subscriber` 随时取消接收数据。  
  
reactive-streams.jar 或 jdk 1.9 中对 RP 的实现是比较底层的。RxJava ([RxJava: Reactive Extensions for the JVM](https://github.com/ReactiveX/RxJava)) 和 Reactor ([Reactor 3](https://projectreactor.io/docs/core/release/reference/)) 是 Java 领域主流的 RP 编程库，提供了 RP 编程常用的高级 API。 

## Spring WebFlux 
Spring 5 推出的 WebFlux 使用 Reactor 作为其底层的 reactive 库。Reactor 提供了 Mono 和 Flux 两个数据流管理的 API 类型。Mono 是 0..1 个数据元素异步到达 -> push 管理工具；Flux 是 0..N 个数据元素持续到达 -> push 的管理工具。二者都是 `Publisher<T>` 的具体实现。
  
## Spring WebClient
WebFlux 是对 SpringMVC 的挑战，是对 web 服务实现方式的改变。WebClient 是对 RestTemplate 的挑战，是对调用 web 服务的方式的改变。  
WebClient 默认使用 Reactor Netty 作为 http 连接库。与 RestTemplate 相比，WebClient 有以下特点。
- 非阻塞、reactive，支持更高的并发能力
- 支持函数式编程
- 同时支持同步和异步调用场景
- 支持流式地从服务器获取数据或流式地向服务器发送数据

### WebClient 常见用法





