---
layout:     post
title:      Axon Framework 初探
subtitle:   CQRS and Event-sourcing 实践
date:       2020-04-29
author:     sincosmos
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 领域驱动设计, CQRS, Event Sourcing
---  

## What is Axon Framework
Axon Framework 旨在帮助开发者运用 CQRS 开发模式，开发出可伸缩，易扩展，便于维护的应用程序。它提供了 CQRS 开发模式中最重要的一些组件，例如 Aggregates, Repositories 和 Event bus，并提供了基于注解的接入方式，便于使用。Axon 保证 CQRS 中事件传递的可靠性，保证事件处理的并发问题及执行顺序。



## 参考资料
1. [Axon Reference Guide](https://docs.axoniq.io/reference-guide/)

