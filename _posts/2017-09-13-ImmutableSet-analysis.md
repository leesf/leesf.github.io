---
title: Guava-ImmutableSet
date: 2017-09-13 19:18:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
> 介绍完`ImmutableCollection`后，接着看看`ImmutableSet`，其表示不可变哈希集合，即创建后无法对集合再进行修改操作，并且`ImmutableSet`会使用一个数组(可变参数数组)维护插入顺序，即当遍历时会按照插入顺序输出。

## ImmutableSet

> `ImmutableSet`类实现了`ImmutableCollection`接口和`Set`接口，可以使用`of`方法创建`ImmutableSet`实例。

### 示例

```java

import com.google.common.collect.ImmutableSet;

public class ImmutableSetTest {
    public static void main(String[] args) {
        ImmutableSet<String> strings = ImmutableSet.of("leesf", "dyd", "ld");
        System.out.println(strings);

        strings = ImmutableSet.<String>builder().add("db").build();
        System.out.println(strings);
    }
}


```

运行结果：
> [leesf, dyd, ld]  
[db]

值得注意的是输出次序与插入顺序一致，这是由于底层遍历时会维护插入顺序，可以使用`of`方法直接构造`ImmutableSet`实例，也可通过`builder`方法进行构造。

### construct方法
> `ImmutableSet`对外提供`of`方法构建实例，`of`方法会调用`construct`方法。`construct`方法源码如下

```java

  private static <E> ImmutableSet<E> construct(int n, Object... elements) {
    switch (n) {
      case 0:
        return of();
      case 1:
        @SuppressWarnings("unchecked") // safe; elements contains only E's
        E elem = (E) elements[0];
        return of(elem);
      default:
        // continue below to handle the general case
    }
    int tableSize = chooseTableSize(n); // 确定表大小
    Object[] table = new Object[tableSize]; // 创建表
    int mask = tableSize - 1; // 用于确定元素在数组中的位置
    int hashCode = 0;
    int uniques = 0;
    for (int i = 0; i < n; i++) { // 遍历元素
      Object element = ObjectArrays.checkElementNotNull(elements[i], i); // 传入的参数不为空
      int hash = element.hashCode(); // 获取哈希码
      for (int j = Hashing.smear(hash); ; j++) { // 进行插入表操作
        int index = j & mask; // 确定插入位置
        Object value = table[index]; // 获取该位置的表元素
        if (value == null) { // 不存在元素
          // Came to an empty slot. Put the element here.
          elements[uniques++] = element; // 将数据存放在传入的表中
          table[index] = element; // 将数据存放在table中
          hashCode += hash;
          break;
        } else if (value.equals(element)) { // 存在相同的元素，表示重复，不插入
          break;
        }
      }
    }
    Arrays.fill(elements, uniques, n, null); // 多余的数组空间存放null
    if (uniques == 1) { // 只有一个元素
      // There is only one element or elements are all duplicates
      @SuppressWarnings("unchecked") // we are careful to only pass in E
      E element = (E) elements[0];
      return new SingletonImmutableSet<E>(element, hashCode); // 生成只包含一个元素的集合
    } else if (tableSize != chooseTableSize(uniques)) { // 表示传入的参数存在过多的重复元素
      // Resize the table when the array includes too many duplicates.
      // when this happens, we have already made a copy
      return construct(uniques, elements);
    } else { // 生成常规的集合
      Object[] uniqueElements = (uniques < elements.length)
          ? ObjectArrays.arraysCopyOf(elements, uniques)
          : elements;
      return new RegularImmutableSet<E>(uniqueElements, hashCode, table, mask);
    }
  }

```

对于`construct`方法已经进行了详细注释，需要注意的是其会使用`elements`数组来保存元素的顺序。

### builder方法

> `builder`方法会生成一个`Builder`实例，在实例中包含了`Object[]`类型的`contents`属性，其用来存放`add`方法中传入的元素，在生成实例后，可以调用实例方法，如`add`方法添加元素。

### add方法

> `add`方法会首先检查`contents`数组的容量是否够，然后再进行添加操作，最后返回`Builder`实例，方便进行链式调用。

### build方法

> 在调用完`add`后，需要调用`build`方法完成`ImmutableSet`实例的创建，在`build`方法中会调用`construct`方法构建`ImmutableSet`并将`contents`做为参数传入。

## 总结

> `ImmutableSet`用于添加元素，并且不能修改集合，其核心方法为`construct`方法，其与`Java`的`Set`方法不同，其会通过一个数组维护插入顺序。


