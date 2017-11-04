---
title: Guava-LinkedHashMultiset
date: 2017-11-04 12:34:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
> 前面了解了`HashMultiset`的实现和使用，接着学习`LinkedHashMultiset`的使用。

## LinkedHashMultiset

`LinkedHashMultiset`也实现了`AbstractMapBasedMultiset`抽象类，其是实现`Multiset`的子类，类继承图结构如下。

![](https://raw.githubusercontent.com/leesf/blogPhotos/master/guava/linkedhashmultiset/linkedhashmultiset-class.png)

`AbstractMapBasedMultiset`作为`HashMultiset`的父类，实现了具体的添加、删除、计数的逻辑。

### 示例

```java

package com.hust.grid.leesf.guavalearning;

import com.google.common.collect.LinkedHashMultiset;
import com.google.common.collect.Multiset;

public class LinkedHashMultisetTest {
    public static void main(String[] args) {
        LinkedHashMultiset<String> linkedHashMultiset = LinkedHashMultiset.create();
        linkedHashMultiset.add("leesf");
        linkedHashMultiset.add("leesf");
        traverse(linkedHashMultiset);
        linkedHashMultiset.remove("leesf");
        traverse(linkedHashMultiset);
        linkedHashMultiset.add("dyd");
        traverse(linkedHashMultiset);
        linkedHashMultiset.remove("leesf");
        traverse(linkedHashMultiset);
        linkedHashMultiset.remove("dyd");
        traverse(linkedHashMultiset);
    }

    public static void traverse(LinkedHashMultiset<String> multiset) {
        System.out.println("+++++++++++++++++++++++++++");
        for (Multiset.Entry entry : multiset.entrySet()) {
            System.out.println(entry.getElement() + " : " + entry.getCount());
        }
    }
}


```

运行结果如下：  
> +++++++++++++++++++++++++++  
leesf : 2  
+++++++++++++++++++++++++++  
leesf : 1  
+++++++++++++++++++++++++++  
leesf : 1  
dyd : 1  
+++++++++++++++++++++++++++  
dyd : 1  
+++++++++++++++++++++++++++            

可以看到其可以多次添加同一个`Key`，同一个`Key`只会被存储一次(Hash属性)，但是其值(`Count`)会自动添加，当进行移除时，也可以看到若元素的`Count`大于`1`，则只会减少`Count`，当`Count`减少到为`0`，就会移除该元素，并且元素是和插入顺序一致的。

### add方法

该方法首先会调用`AbstractMultiset`类的`add(@Nullable E element)`方法，然后该方法会调用`AbstractMapBasedMultiset`的`add(@Nullable E element, int occurrences)`方法，其源码如下。

```java

  @Override public int add(@Nullable E element, int occurrences) {
    if (occurrences == 0) { // 次数为0，直接返回现有集合中该element出现的次数
      return count(element);
    }
	// 保证出现次数不能为负数
    checkArgument(
        occurrences > 0, "occurrences cannot be negative: %s", occurrences);
	// 获取该元素对应的Count对象
    Count frequency = backingMap.get(element);
    int oldCount;
    if (frequency == null) { // 不存在，表示第一次添加该元素
	  // 之前出现次数为0
      oldCount = 0;
	  // 添加至backingMap中，该集合存放元素和出现次数(Count对象)
      backingMap.put(element, new Count(occurrences));
    } else { // 集合中存在该元素
	  // 获取之前出现的次数
      oldCount = frequency.get();
	  // 计算新的次数
      long newCount = (long) oldCount + (long) occurrences;
	  // 保证不超过最大阈值
      checkArgument(newCount <= Integer.MAX_VALUE,
          "too many occurrences: %s", newCount);
	  // 获取然后增加次数
      frequency.getAndAdd(occurrences);
    }
	// 改变集合大小
    size += occurrences;
	// 返回集合中元素之前的出现次数
    return oldCount;
  }

```

> 可以看到其主要是通过`backingMap`提供支持，其为`LinkedHashMap<String, Count>`类型，`Count`对象中封装了出现的次数和对次数进行获取和改变的操作，`LinkedHashMap<String, Count>`维护插入顺序。

### remove方法

该方法首先会调用`AbstractMultiset`类的`remove(@Nullable E element)`方法，然后该方法会调用`AbstractMapBasedMultiset`的`remove(@Nullable E element, int occurrences)`方法，其源码如下。

```java

    @Override public int remove(@Nullable Object element, int occurrences) {
    if (occurrences == 0) { //次数为0，直接返回现有集合中该element出现的次数
      return count(element);
    }
	// 保证出现次数不能为负数
    checkArgument(
        occurrences > 0, "occurrences cannot be negative: %s", occurrences);
	// 获取该元素对应的Count对象	
    Count frequency = backingMap.get(element);
    if (frequency == null) { // 不存在，表示第一次添加该元素
      return 0; // 直接返回0
    }
	// 获取之前出现次数
    int oldCount = frequency.get();
	// 移除的次数
    int numberRemoved;
    if (oldCount > occurrences) { // 集合之前出现的次数大于要删除的次数
	  // 移除的次数为删除的次数
      numberRemoved = occurrences;
    } else { // 集合之前出现的次数小于等于要删除的次数
	  // 移除的次数为之前出现的次数
      numberRemoved = oldCount;
	  // 移除该元素
      backingMap.remove(element);
    }
	// 更新存在的次数
    frequency.addAndGet(-numberRemoved);
	// 改变集合大小
    size -= numberRemoved;
	// 返回集合中元素之前的出现次数
    return oldCount;
  }

```

可以看到当需要删除的次数大于等于集合中已存在的次数时，会直接删除元素，否则，只会减少该元素的次数；当删除不存在的元素时，会直接返回`0`。


## 总结

本篇博文讲解了`LinkedHashMultiset`的使用，其与`HashMultiset`极为相似，其维护了插入顺序，如同`LinkedHashMap`和`HashMap`之间的区别。



