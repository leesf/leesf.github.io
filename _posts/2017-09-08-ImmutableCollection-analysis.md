---
title: Guava-不可变集合父类ImmutableCollection
date: 2017-09-08 19:14:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
> 经过前面几个基础点的学习，接着分析`Guava`中的核心，不可变集合，所有不变集合的父类为`ImmutableCollection`，其子类包括`EntryCollection`、`ImmutableSet`、`ImmutableList`、`ImmutableMultiSet`、`ImmutableMapValues`等，现分析其源码。

## ImmutableCollection
> `ImmutableCollection`实现了`Java`的`Collection`和`Serializable`接口，其重写了`Collection`中的很多方法，针对更改集合类型的操作，如`add`方法，则直接抛出异常。

### iterator方法
> `iterator`方法会返回一个不可修改的`UnmodifiableIterator`，其实现了`Java`的`Iterator`接口，并且重写了`remove`方法，直接抛出异常。

### toArray方法
> 该方法可将集合转化为`Array`数组，具体会调用`ObjectArrays`类的`fillArray`方法，其会遍历集合，并将集合元素放在`Object`类型中的数组中，针对不同的集合实现也会重写该方法。

### add方法
> `add`方法会修改集合，由于为不可变集合，会直接抛出`UpsupportedOperationException`异常。

### remove方法
> 同`add`方法一样会修改集合，该方法也是不允许的，需要抛出异常。

### asList方法
> 该方法将集合转化为`List`类型。

### EmptyImmutableCollection类
> 该类表示空集合，不存在任何元素。

### Builder类

> `Builder`类用于构建`ImmutableCollection`，其通过`add`方法添加元素，并通过`expandedCapacity`进行扩容，最后需要调用`build`方法完成`ImmutableCollection`的构建。

## 总结
> `ImmutalbeCollection`是所有不可变集合的父类，其定义了一些通用方法，在具体子类中对某些方法需要重写。