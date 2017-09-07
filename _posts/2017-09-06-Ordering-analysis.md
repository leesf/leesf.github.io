---
title: Guava-Ordering使用
date: 2017-09-06 20:53:16
categories:
- technique
tags:
- Guava
- Java
---

## 前言
> 在原生`Java`中，当对不同的类进行比较时，需要让类实现`Comparable`接口或者`Comparator`接口，其特别是在基于`Hash`实现的散列表中非常重要。而`Guava`实现了`Ordering`类，可以供开发者更方便地比较不同对象。

## Ordering

> `Ordering`实现了`Comparator`接口，在实现`compare`和`equals`方法的基础上，增强了多个方法，其可以构造非常复杂的比较器，以便在容器的排序和比较中使用。

### 示例

```java

package com.hust.grid.leesf.guavalearning;

import com.google.common.base.Function;
import com.google.common.collect.Ordering;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;


public class OrderingTest {

    static class Person {
        private String name;
        private int age;

        public Person(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String toString() {
            return "[name = " + name + ", age = " + age + "]";
        }
    }

    public static void main(String[] args) {
        Ordering<Person> ordering = Ordering.natural().onResultOf(new Function<Person, Comparable>() {
            public String apply(Person input) {
                return input.name;
            }
        }).compound(new Comparator<Person>() {
            public int compare(Person o1, Person o2) {
                return o2.age - o1.age;
            }
        });

        Person leesf = new Person("leesf", 25);
        Person leesf1 = new Person("leesf", 26);
        Person dyd = new Person("dyd", 25);
        Person ld = new Person("ld", 25);

        List<Person> persons = new ArrayList<Person>();
        persons.add(leesf);
        persons.add(leesf1);
        persons.add(dyd);
        persons.add(ld);

        System.out.println(ordering.compare(leesf, dyd));
        System.out.println(ordering.max(persons));
        System.out.println(ordering.min(persons));
        List<Person> sortPersons = ordering.sortedCopy(persons);
        System.out.println(sortPersons);
        System.out.println(persons);
    }
}




```

运行结果：


> 8
[name = leesf, age = 25]  
[name = dyd, age = 25]  
[[name = dyd, age = 25], [name = ld, age = 25], [name = leesf, age = 26], [name = leesf, age = 25]]  
[[name = leesf, age = 25], [name = leesf, age = 26], [name = dyd, age = 25], [name = ld, age = 25]]

### natural方法

> 当调用`natural`方法时，返回一个以自然顺序进行排序的比较器`NaturalOrdering`对象，`NaturalOrdering`类继承了`Ordering<Comparable>`，因此，其返回后可以继续调用父类`Ordering`的方法，形成链式调用。

### onResult方法

> 该方法实现比较的具体逻辑，需要传入`Function`对象，并实现`apply`方法添加具体的比较逻辑，在示例中是以`Person`的`name`属性进行比较。

### compound方法

> compound方法表示在比较`name`属性之后，若其相等，则再按照`age`进行比较。示例中将`age`大的排序在前面。

## 总结

> `Ordering`主要是用作比较，组合不同的比较器，可以对集合进行排序，通过`sortCopy`方法返回的是原始集合的副本，两者互不影响。