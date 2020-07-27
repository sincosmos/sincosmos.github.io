---
layout:     post
title:      Java Bean 有效校验
subtitle:   
date:       2020-06-24
author:     sincosmos
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java Valid, Hibernate Validator
---  

## Bean Validation
对用户的输入或其它内容进行有效性验证是非常常见的需求，JSR-380 是 Java 一项标准规范，规定了对 Java Bean 进行有效性校验的相关技术。对此规范的实现，是辅助性的 Java Bean Validation framework。
JDK 在包 javax.validation 中定义了该规范的接口，我们在项目中使用时还需要对各种 Validator 进行具体的实现。Hibernate Validator 是对 JSR-380 的参考实现，是目前最流行的（某种程度上是唯一的） Java Bean Validation framework。  
Spring 中就默认使用 Hibernate Validator 进行 Bean 校验。Spring 自行定义了 Validator 等相关接口，同时为了适配 javax.validation.Validator，Spring 提供 SpringValidatorAdapter 实现了其自定义 Validator 接口和 javax.validation.Validator 以对二者进行适配。Spring 的 Validator 本身并不用于 Bean 有效检查，而是组合了一个 javax.validation.Validator 的对象（一般使用 Hibernate Validator 的具体实现）进行有效检查，Spring 的 Validator 会将  javax.validation.Validator 的检查结果封装为 BindingResult 便于返回给客户端。




