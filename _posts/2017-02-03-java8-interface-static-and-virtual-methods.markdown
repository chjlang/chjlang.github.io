---
layout: post
author: chjlang
title:  "Java 8接口中的静态方法和默认方法"
date:   2017-02-03 17:51:00 +8000
categories: java
tags:
    - java
    - study
---
在java 8之前，接口可以拥有静态域（隐式拥有public staitic final修饰符）和抽象方法（隐式拥有abstract public修饰符），接口中不能有任何实现。

从java 8开始，接口可以拥有默认方法和静态方法。

# 默认方法

默认方法用default关键字来修饰，默认方法可以拥有方法体，包含实现代码。接口的实现类不会被强制要求实现默认方法，实现类的对象自动获该方法的默认行为。同时默认方法也可以被具体的实现类重写来实现特定的功能。

如下代码所示，Fomula的一个实现类可以直接调用sqrt()方法来实现calculate这个抽象的方法

```java
interface Formula {

    int calculate(int number);

    default int sqrt(int number) {
        return (int) Math.sqrt((double) number);
    }
}

Formula formula = new Formula() {
    @Override
    public int calculate(int number) {
        return sqrt(number);
    }
};

int a = formula.caculate(9); // a = 3
```

默认方法带来的好处是可以方便扩展已有接口，如果需要在接口添加新方法，可以直接添加一个默认方法，这样不会影响已有的该接口的实现类（即不需要所有实现类都实现新添加的方法）。java 8就是利用了这样的好处来扩展了原来的Collection Framework（例如引入了stream api，增加了sort方法等）和引入了lambda表达式。

# 静态方法

从java 8开始，可以在接口中定义静态方法，而这些静态方法包含具体的实现。这样的好处是可以将该接口相关的静态方法都定义在该接口，而不需要定义一个静态类来实现相应的方法。同时这些静态方法也可以被默认方法使用，使得默认方法实现起来更为方便。见以下jdk 8中Comparator接口的代码片段：

```java
// default method
default <U extends Comparable<? super U>> Comparator<T> thenComparing(
        Function<? super T, ? extends U> keyExtractor)
{
    return thenComparing(comparing(keyExtractor));
}

// static method
public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
        Function<? super T, ? extends U> keyExtractor)
{
    Objects.requireNonNull(keyExtractor);
    return (Comparator<T> & Serializable)
        (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
}
```
