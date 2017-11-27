---
title: Guava-ImmutableMap
date: 2017-09-29 19:18:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
> 分析完了`ImmutableSet`后，接着分析`ImmutableMap`，其使用与`ImmutableSet`大致类似，下面对其源码进行分析。

## ImmutableMap

> `ImmutableMap`实现了`Map`接口和`Serializable`接口，表示不可变的`Map`，在构建完成后不允许进行结构性的修改。

### 示例

```java

import com.google.common.collect.ImmutableMap;

public class ImmutableMapTest {
    public static void main(String[] args) {
        ImmutableMap<String, Integer> maps = ImmutableMap.of("dyd", 25, "leesf", 25);
        System.out.println(maps);

        ImmutableMap<String, Integer> maps1 = ImmutableMap.<String, Integer>builder().put("leesf", 25).put("dyd", 25).build();
        System.out.println(maps1);
    }
}


```

运行结果：

> {dyd=25, leesf=25}  
{leesf=25, dyd=25}

### of方法

> 在`of`方法中会首先使用`K、V`调用`entriesOf`方法生成键值对，该方法会进行空值检查，要求键值都不能为空，然后调用`RegularImmutableMap`构造函数进行构造，其构方法是核心，源码如下

```java

  RegularImmutableMap(Entry<?, ?>... immutableEntries) {
	// 可变参数大小
    int size = immutableEntries.length;
	// 创建指定大小的数组，用于维护插入顺序
    entries = createEntryArray(size);
	// 根据参数大小和可变因子确定表大小
    int tableSize = Hashing.closedTableSize(size, MAX_LOAD_FACTOR);
	// 创建指定大小的数组
    table = createEntryArray(tableSize);
	// 用于确定元素存放位置
    mask = tableSize - 1;
	
    for (int entryIndex = 0; entryIndex < size; entryIndex++) { // 遍历元素
      // each of our 6 callers carefully put only Entry<K, V>s into the array!
      @SuppressWarnings("unchecked")
	  // 获取元素
      Entry<K, V> entry = (Entry<K, V>) immutableEntries[entryIndex];
	  // 获取键
      K key = entry.getKey();
	  // 获取键的hash值
      int keyHashCode = key.hashCode();
	  // 确定存放位置
      int tableIndex = Hashing.smear(keyHashCode) & mask;
	  // 将存放位置的元素取出
      @Nullable LinkedEntry<K, V> existing = table[tableIndex];
      // prepend, not append, so the entries can be immutable
	  // 创建LinkedEntry节点，会根据元素是否存在创建不同类型的节点，
	  // 并且将新创建的节点的next域设置为已存在头部节点。
      LinkedEntry<K, V> linkedEntry =
          newLinkedEntry(key, entry.getValue(), existing);
	  // 设置新创建的节点为头节点
      table[tableIndex] = linkedEntry;
	  // 赋值给entries数组，用于维护插入顺序
      entries[entryIndex] = linkedEntry;
      while (existing != null) { // 如果存在元素，则需要循环遍历链表，判断是否存在元素重复
        checkArgument(!key.equals(existing.getKey()), "duplicate key: %s", key);
        existing = existing.next();
      }
    }
  }

```

> 可以看到其会使用额外的数组来维护元素插入顺序，这样在遍历时与插入顺序相同。

### put方法

> 在调用`builder`方法后，需要使用`put`方法写入元素，其源码相对简单。

```java

    public Builder<K, V> put(K key, V value) {
	  // 生成Entry后放入ArrayList中
      entries.add(entryOf(key, value));
      return this;
    }

```

### build方法

> 在`put`完数据后，需要使用`build`方法完成构建操作，其会调用`fromEntryList`方法，其源码如下。

```java

    private static <K, V> ImmutableMap<K, V> fromEntryList(
        List<Entry<K, V>> entries) {
	  // 获取列表大小
      int size = entries.size();
      switch (size) {
        case 0: // 为空
          return of();
        case 1: // 只有一个元素
          return new SingletonImmutableBiMap<K, V>(getOnlyElement(entries));
        default: // 多个元素，转化为数组后调用构造函数
          Entry<?, ?>[] entryArray
              = entries.toArray(new Entry<?, ?>[entries.size()]);
          return new RegularImmutableMap<K, V>(entryArray);
      }
    }


```

> 可以看到，其会根据`entries`大小来构建不同类型的`Map`，有多个元素时，其调用之前分析的构造函数，否则，生成空或者只有一个元素的`Map`。

## 总结

> 可以看到，对于`ImmutableMap`而言，也使用了内部数组来维护插入顺序，其与`JDK`中的`Map`相似，但是值得注意的是每个数组元素对应的链表元素过多时，遍历效率相对较低，而在`JDK 1.8`中当元素较多时，会使用红黑树进行存储，这样可以提高访问效率。