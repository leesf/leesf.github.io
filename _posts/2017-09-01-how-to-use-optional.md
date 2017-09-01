---
title: Guava-Optional使用
date: 2017-09-01 19:23:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言

> 在`Java`开发中，`null`就像给黑洞，给错误的排查带来极大的难度，而`Guava`对`null`进行了优化，可以方便我们更优雅的处理`null`。

## Optional

> `Guava`中使用`Optional`来处理`null`，如使用`Optional(null)`时，则会快速报错，而使用`Java`原生方式处理时，则会在具体使用到`null`时报错，下面通过一个示例说明。

### 示例

### 原生Java示例

```java

package com.hust.grid.leesf.optional;

import com.google.common.base.Optional;

public class OptionalTest {
    public static void main(String[] args) {
        Integer invalidInput = null;
        Integer validInput = 10;

        sum(invalidInput, validInput);
    }

    public static int sum(Integer i1, Integer i2) {
        return i1 + i2;
    }
}

```

运行结果如下：

> Exception in thread "main" java.lang.NullPointerException  
	at com.hust.grid.leesf.optional.OptionalTest.sum(OptionalTest.java:18)  
	at com.hust.grid.leesf.optional.OptionalTest.main(OptionalTest.java:10)

可以看到，其会在`sum`方法中报错，即当运行`i1 + i2`时报错，若不采取合适的判断非`null`方法，则该程序会报错。

### Guava示例

```java

public class OptionalTest {
    public static void main(String[] args) {
        Integer invalidInput = null;
        Integer validInput = 10;

        Optional<Integer> i1 = Optional.of(invalidInput);
        Optional<Integer> i2 = Optional.of(validInput);

        sum(i1, i2);
        //sum1(invalidInput, validInput);
    }

    public static int sum(Optional<Integer> i1, Optional<Integer> i2) {
        return i1.get() + i2.get();
    }

```

运行结果如下：

> Exception in thread "main" java.lang.NullPointerException  
	at com.google.common.base.Preconditions.checkNotNull(Preconditions.java:191)  
	at com.google.common.base.Optional.of(Optional.java:86)  
	at com.hust.grid.leesf.optional.OptionalTest.main(OptionalTest.java:10)  

可以看到在运行`Optional.of(invalidInput)`时报错了，不会调用`sum`方法，使用`Optional`来规避方法调用时的非`null`判断，使得能够更快速的抛出异常。

### 源码分析

#### 创建非null对象

> 根据`Guava`示例，来简单分析`Optional`的源码，其有**`Absent`**和**`Present`**两个子类，`Absent`代表`null`，`Present`则代表非`null`，当调用`Optional.of`方法时，其会创建`Present`对象。

* `Optional.of`方法首先调用`checkNotNull(reference)`方法。当`reference`为`null`时，则直接抛出`NullPointerException`异常，否则创建`Present`对象。
* `Present`对象中有`T reference`的变量用于保存非`null`值。

#### 创建null对象
> 除了创建非`null`对象外，还可以特意创建`null`对象，而创建`null`对象有两种方法，一种通过`fromNullable`方法，另一种通过`absent`方法。


##### fromNullable方法

> 可以通过`Optional.fromNullable(null)`方法创建，其会创建一个`Absent`对象表示`null`，而当调用`Optional.fromNullable`方法并且传递的参数不为`null`时，其会正常创建`Present`对象。

##### absent方法

> 直接通过调用`Optional.absent`方法即可创建一个`Absent`对象表示`null`。


**值得注意的是，所有的`null`对象都为同一个对象(哈希码相同)。**

### 获取元素

> 上述方法可以通过`Optional`来保存传递的变量，再使用时，可调用`Optional`的`get`或者`or`方法获取值。

#### get方法

> 对于`Absent`对象而言，调用`get`会抛出异常；对于`Present`对象而言，调用`get`会返回`reference`变量。

#### or方法

> 对于`Absent`对象而言，调用`or`会返回传入的参数；对于`Present`对象而言，调用`or`会返回原本传入的变量值。

## 总结

> `Optional`源码比较简单，主要是为了避免运行时`null`进行的优化。
