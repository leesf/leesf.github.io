---
title: Spring-框架介绍
date: 2017-09-15 18:18:16
categories:
- technique
tags:
- Spring
- Java
---

## 前言
> 在开发`Java Web`时，需要使用`Spring`的知识，可以说没有`Spring`，很难出现这么便捷的`Web`开发技术，现在工作内容虽然编写的很多都是业务层代码，但也需要适时从业务层抽离出来，系统学习`Spring`框架知识。

## Spring框架

![Markdown](https://raw.githubusercontent.com/leesf/blogPhotos/master/spring-framework.jpg)

* 核心容器：容器提供`Spring`框架的基本功能。主要组件是`BeanFactory`，它是工厂模式的实现，提供`DI`功能，其管理`Bean`创建、配置、管理。
* AOP：`Spring`对面向切面编程提供了丰富的支持，该模块是`Spring`应用系统中开发切面的基础。
* Data Access/Integration：`JDBC`模块简化了访问数据库的样板代码，`ORM`模块建立在对`DAO`支持之上，如`MyBatis`、`Hibernate`等。`JMS`模块使用消息异步的方式与其他应用集成。
* Web：使用`Spring MVC`框架开发`Web`项目。
* Instrumentation：提供为`JVM`添加代理的功能。
* Test：提供测试模块致力于`Spring`应用的测试。

## 总结

> `Spring`致力于简化`Java`开发，促进代码的松散耦合，简化主要依赖于`DI`和`AOP`，后面会详细介绍。

