---
title: Guava-ImmutableList
date: 2017-09-20 19:58:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
> 分析完`ImmutableSet`后，接着分析`ImmutableList`，从字面上可知其是不可变的列表，可根据索引获取对应项，由于其在创建后不可变，底层可以使用数组来存储，这样会访问效率。

## ImmutableList
> `ImmutableList`类实现了`ImmutableCollection`接口和`List`接口，可以使用`of`方法或者`builder`配合`build`方法创建`ImmutableList`实例。

### 示例

```java

package com.hust.grid.leesf.guavalearning;

import com.google.common.collect.ImmutableList;

public class ImmutableListTest {
    public static void main(String[] args) {
        ImmutableList<String> strings = ImmutableList.of("dyd", "leesf", "ld");
        System.out.println(strings);

        strings = ImmutableList.<String>builder().add("dyd").add("leesf").add("ld").build();
        System.out.println(strings);
    }
}


```

运行结果：
> [dyd, leesf, ld]  
[dyd, leesf, ld]

示例展示了通过`of`和`builder`及`build`方法构建`ImmutableList`对象。

### construct方法

> 调用`of`方法会调用`ImmutableList`的`construct`方法，其源码如下

```java

  private static <E> ImmutableList<E> construct(Object... elements) {
    for (int i = 0; i < elements.length; i++) {
      ObjectArrays.checkElementNotNull(elements[i], i);
    }
    return new RegularImmutableList<E>(elements);
  }

```

首先会检查传入的数组是否有值为空，若为空，则会抛出异常，否则，调用`RegularImmutableList`构造函数，其中，`RegularImmutableList`继承`ImmutableList`类，其内部使用数组和偏移值来保存具体的值。

### builder方法

> 调用`builder`方法会生成新的`Builder`实例，`Builder`类的源码大致如下

```java

public static final class Builder<E> extends ImmutableCollection.Builder<E> {
	private Object[] contents;
	private int size;
}


```

构造函数会生成默认大小的数组，并设置`size = 0`，`size`用于指示已经添加了多少个元素。

### add方法

> 当使用`builder`方法返回`Builder`对象后，可使用`add`方法添加元素，其源码如下

```java

    @Override public Builder<E> add(E element) {
      checkNotNull(element);
      ensureCapacity(size + 1);
      contents[size++] = element;
      return this;
    }

```

可以看到首先检查传入的元素是否为空，然后保证新添加元素后保证数组不会溢出。

### build方法

> 当使用`add`方法添加完所有的元素后，需要调用`build`方法完成`ImmutableList`的构造，其源码如下

```java

    @Override public ImmutableList<E> build() {
      switch (size) {
        case 0:
          return of();
        case 1:
          @SuppressWarnings("unchecked") // guaranteed to be an E
          E singleElement = (E) contents[0];
          return of(singleElement);
        default:
          if (size == contents.length) {
            // no need to copy; any further add operations on the builder will copy the buffer
            return new RegularImmutableList<E>(contents);
          } else {
            return new RegularImmutableList<E>(ObjectArrays.arraysCopyOf(contents, size));
          }
      }
    }

```

可以看到其会根据添加的元素个数来生成对应的`List`。

## 总结

> `ImmutableList`类在完成构造后便不可再添加元素，所有修改集合的操作均会抛出异常，其使用和源码相对简单。