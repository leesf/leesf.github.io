---
title: Guava-ImmutableTable
date: 2017-11-22 20:34:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
在学习了`HashBiMap`后，接着学习`ImmutableTable`，该类由行和列共同确定一个元素，类似于表格。


## ImmutableTable

`ImmutableTable`实现`Table`接口，`Table`接口中定义了表格操作规范，需要具体的子类实现。

### 示例

```java

package com.hust.grid.leesf.guavalearning;

import com.google.common.collect.ImmutableSet;
import com.google.common.collect.ImmutableTable;

import java.util.Map;

public class ImmutableTableTest {
    public static void main(String[] args) {
        ImmutableTable<String, String, Integer > immutableTable = ImmutableTable.<String, String, Integer>builder()
                .put("0", "0", 0)
                .put("50", "50", 50)
                .put("100", "100", 100)
                .build();
        ImmutableSet<String> rows = immutableTable.rowKeySet();
        for (String row : rows) {
            for (Map.Entry<String, Integer> entry : immutableTable.row(row).entrySet()) {
                System.out.println(entry.getKey() + " -> " + entry.getValue());
            }
        }

    }
}


```

运行结果

> 0 -> 0  
50 -> 50  
100 -> 100  

### ImmutableTable#Builder

在调用`builder`方法时会生成一个`Builder`对象，`Builder`类结构如下。

```java

  public static final class Builder<R, C, V> {
    private final List<Cell<R, C, V>> cells = Lists.newArrayList();
    private Comparator<? super R> rowComparator;
    private Comparator<? super C> columnComparator;
  }

```

可以看到`Builder`类中使用`cells`对象存放表格元素，`Cell`对象表示表单元素，由行和列以及值组成；同时，还包含了行列比较器。


### put

在构造完`Builder`对象后，调用`put`方法添加元素，`put`方法源码如下。

```java

    public Builder<R, C, V> put(R rowKey, C columnKey, V value) {
	  // 构造表单元素，并添加至列表中
      cells.add(cellOf(rowKey, columnKey, value));
      return this;
    }

```

其中，`cellof`方法会构造一个`ImmutableCell`对象，`ImmutableCell`实现了`Cell`接口。

### forCellsInternal

在添加完元素后，调用`build`方法，该方法根据`cells`大小确定构造何种`Table`，本例中，会构造`RegularImmutableTable`，并且最后会调用`forCellsInternal`方法，其源码如下。

```java

  private static final <R, C, V> RegularImmutableTable<R, C, V>
      forCellsInternal(Iterable<Cell<R, C, V>> cells,
          @Nullable Comparator<? super R> rowComparator,
          @Nullable Comparator<? super C> columnComparator) {
	// 行构建器
    ImmutableSet.Builder<R> rowSpaceBuilder = ImmutableSet.builder();
	// 列构建器
    ImmutableSet.Builder<C> columnSpaceBuilder = ImmutableSet.builder();
	// 拷贝表格元素列表
    ImmutableList<Cell<R, C, V>> cellList = ImmutableList.copyOf(cells);
    for (Cell<R, C, V> cell : cellList) { // 遍历
	  // 将行添加至行构建器中
      rowSpaceBuilder.add(cell.getRowKey());
	  // 将列添加至列构建器中
      columnSpaceBuilder.add(cell.getColumnKey());
    }

	// 行Set
    ImmutableSet<R> rowSpace = rowSpaceBuilder.build();
    if (rowComparator != null) { // 行比较器不为空
	  // 将Set转化为List
      List<R> rowList = Lists.newArrayList(rowSpace);
	  // 根据比较器进行排序
      Collections.sort(rowList, rowComparator);
	  // 重新赋值排序后的List
      rowSpace = ImmutableSet.copyOf(rowList);
    }
	// 对列进行同样处理
    ImmutableSet<C> columnSpace = columnSpaceBuilder.build();
    if (columnComparator != null) {
      List<C> columnList = Lists.newArrayList(columnSpace);
      Collections.sort(columnList, columnComparator);
      columnSpace = ImmutableSet.copyOf(columnList);
    }

    // use a dense table if more than half of the cells have values
    // TODO(gak): tune this condition based on empirical evidence
	
	// 构建稠密或稀疏表，当元素个数超过行列积一半时，构建稠密表，否则构建稀疏表
    return (cellList.size() > ((rowSpace.size() * columnSpace.size()) / 2)) ?
        new DenseImmutableTable<R, C, V>(cellList, rowSpace, columnSpace) :
        new SparseImmutableTable<R, C, V>(cellList, rowSpace, columnSpace);
  }

```

可以看到，最后会构造出`DenseImmutableTable`或者`SparseImmutableTable`表，两个类中都存在`rowMap(行 -> (列，value))`和`columnMap(列 -> (行，value))`。而后调用`rowKeySet`或者`columnKeySet`时，`SparseImmutableTable`会从`rowMap`或`columnMap`中取出`keySet`，而`DenseImmutableTable`从`rowKeyToIndex(R -> Integer)`或`columnKeyToIndex(C -> Integer)`中取出`keySet`，其中`DenseImmutableTable`存在行列索引主要是为了加快检索速度。两个类的核心是`rowMap`和`columnMap`，其提供底层存储和访问支持。

## 总结

本篇博文讲解了`ImmutableTable`，其相当于表格，通过行列确定元素，与数据结构中的图密不可分，特别是稠密和稀疏表，该类可用于图类场景中。并且简单分析了源码实现，有兴趣的读者可详细研读源码。


