---
layout: post
author: chjlang
title:  "Java集合类问题总结"
date:   2017-02-18 17:51:00 +8000
categories: java
tags:
    - java
    - study
---

# 

## ConcurrentModificationException

对于java.util.collection下的集合类，迭代时修改集合可能会会抛出ConcurrentModificationException
以下为ConcurrentModificationException的java doc说明

>
This exception may be thrown by methods that have detected concurrent modification of an object when such modification is not permissible.
For example, it is not generally permissible for one thread to modify a Collection while another thread is iterating over it. In general, the results of the iteration are undefined under these circumstances. Some Iterator implementations (including those of all the general purpose collection >implementations provided by the JRE) may choose to throw this exception if this behavior is detected. Iterators that do this are known as fail-fast iterators, as they fail quickly and cleanly, rather that risking arbitrary, non-deterministic behavior at an undetermined time in the future.
>
Note that this exception does not always indicate that an object has been concurrently modified by a different thread. If a single thread issues a sequence of method invocations that violates the contract of an object, the object may throw this exception. For example, if a thread modifies a >collection directly while it is iterating over the collection with a fail-fast iterator, the iterator will throw this exception.
>
Note that fail-fast behavior cannot be guaranteed as it is, generally speaking, impossible to make any hard guarantees in the presence of unsynchronized concurrent modification. Fail-fast operations throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write >a program that depended on this exception for its correctness: ConcurrentModificationException should be used only to detect bugs.

由以上说明可以知道，这个异常是在集合类在迭代时检测到底层储层有改动后而抛出的。如果迭代时对原来的集合结构有改动（增加或删除），那么在迭代时可能出现不确定的行为。

```java
List<Integer> list = Lists.newArrayList(1, 2, 3);
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
    if (i == 1) {
        list.add(i, 10);
    }
}
```
例如以上代码在我的机器的输出结果是1 2 2 3。为了限制这种不确定性，java的集合类采取一种fast-fail的方式来处理这个问题，即当迭代时检测到改动时就搜出ConcurrentModificationException，不再继续迭代下去。

然而需要注意的是，改动并不保证一定能及时检测到，所以就算代码在迭代过程中有改动的操作，也不一定会抛出ConcurrentModificationException异常，例如以上的代码就顺利执行完毕，但是结果是不正确的。

所以抛出该异常表示代码有问题，但不抛出该异常也不一定表示代码一定没有问题。使用者应该自己注意。正如java doc里说的

>
Fail-fast operations throw ConcurrentModificationException on a best-effort basis. Therefore, it would be wrong to write a program that depended on this exception for its correctness: ConcurrentModificationException should be used only to detect bugs.

* 若需要在迭代过程删除元素，可以使用iterator来遍历集合，在遍历过程中使用`remove()`方法来删除迭代器当前指向的元素（注意不能用当前的迭代器删除其它元素）

* 对List.subList()方法返回的子列表作增加或删除操作都会抛出ConcurrentModificationException

* ConcurrentHashMap和CopyOnWriteArray采用的是fail-safe策略
    * 对于CopyOnWriteArrayList, 由于每次写操作都会复制一份当前的数据到新的数组中去，修改操作的结果只会反映到新的数组上，迭代继续在原数组上进行
    * 对于ConcurrentHashMap而言，修改后的结果是不确定的，所以应尽量避免在迭代的过程中修改结构

