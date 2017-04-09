---
layout: post
author: chjlang
title:  "Java自动装箱和拆箱"
date:   2017-04-08 17:51:00 +8000
categories: java
tags:
    - java
    - study
---
从JDK 1.5开始，java开始引入范型。由于在Java中使用范型必须使用类而不能是原始数据类型，于是同时引入了原始类型对应的包装类，即int -> Integer, char -> Character等。为了进一步方便包装类的使用，JDK 1.5也同时引入了自动装箱（即自动把原始数据类型转换成对应的包装类）和自动拆箱（即自动把包装类转换成相应的原始数据类型）机制，使得以下语句能够编译，避免需要程序员显式来转换类型，提高了开发效率。

```java
Integer c = 1 + 2;  // auto-boxing
int cc = c;         // auto-unboxing
List<Integer> li = new ArrayList<>();
```

以上的自动装箱和拆箱是Java提供的一个语法糖，自动装箱和拆箱的过程实际在编译期间由编译器自动加上一些`Integer.valueOf()`或者`Integer.intValue()`这些语句来完成数据类型转换的工作。

## 自动装箱的时机

### 1. 当传递一个原始类型的数据到一个需要包装类的方法时

```java
List<Integer> list = new ArrayList<>();
list.add(1);
``` 

### 2. 赋值到一个包装类（或任何非原始类型）的时候

```java
Integer a = 1 + 2;
a.equals(3); 
```

## 自动拆箱时机
### 1. 赋值到一个原始数据类型时

```java
int a = new Integer(1);
```

### 2. 当进行算术运算

```java
int c = new Integer(1) + new Integer(2);
```

### 3. 当比较布尔值时

```java
if (new Boolean(true)) {...}
```

### 4. 当在同一个表达式中出现原始类型与包装类混合的情况时，包装类会进行拆箱

```java
int a = new Integer(1) + 1;
```

### 5. 当==运算遇到算术运算的情况下：

```java
Integer a = 1, b = 2, c = 3
c == a + b;
```

为了验证以上所说的自动装箱和拆箱时机，见以下程序：

```java
public class Boxing {

    public static void main(String[] args) {
        new ArrayList<>().add(1);
        Integer a = 1;
        int aa = a;
        aa = aa + a;
        Integer b = 2, c = 3, d = 3, e = 321, f = 321;
        Long g = 3L;
        System.out.println(c == d); // true
        System.out.println(e == f); // false
        System.out.println(c == (a + b)); // true
        System.out.println(c.equals(a + b)); // true
        System.out.println(g == (a + b)); // true
        System.out.println(g.equals(a + b)); // false, a + b装箱后是Integer类，与Long类型比较由于是不同类型，所以是false
        if (new Boolean(true)) {
            System.out.println(true);
        }
    }
}
```

先对Boxing.java编译后，再反编译Boxing.class文件，可以得到编译器自动装箱和拆箱后的代码：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import java.util.ArrayList;

public class Boxing {
    public Boxing() {
    }

    public static void main(String[] var0) {
        (new ArrayList()).add(Integer.valueOf(1));
        Integer var1 = Integer.valueOf(1);
        int var2 = var1.intValue();
        int var10000 = var2 + var1.intValue();
        Integer var3 = Integer.valueOf(2);
        Integer var4 = Integer.valueOf(3);
        Integer var5 = Integer.valueOf(3);
        Integer var6 = Integer.valueOf(321);
        Integer var7 = Integer.valueOf(321);
        Long var8 = Long.valueOf(3L);
        System.out.println(var4 == var5);
        System.out.println(var6 == var7);
        System.out.println(var4.intValue() == var1.intValue() + var3.intValue());
        System.out.println(var4.equals(Integer.valueOf(var1.intValue() + var3.intValue())));
        System.out.println(var8.longValue() == (long)(var1.intValue() + var3.intValue()));
        System.out.println(var8.equals(Integer.valueOf(var1.intValue() + var3.intValue())));
        if((new Boolean(true)).booleanValue()) {
            System.out.println(true);
        }

    }
}
```

## 注意事项

### ==比较运算
当两个包装类进行==运算时，是对对象的引用进行比较，即判断两个引用是否指向同一个对象，以下代码因为创建了两个不同的对象，所以进行==比较得到的是false

```java
Long a = new Long(1), b = new Long(1);
System.out.println(a == b);     // false
```

所以一般情况下如果为了比较包装类对应的原始类型的值是否相等，应使用用Object中的`equals(Object)`方法。但由于JVM在自动装箱过程中会使用常量池的方式，在一定数值范围内进行的装箱操作总是会返回同一个对象，所以会出现以下代码情况：

```java
Long a = 1, b = 1;
System.out.println(a == b); // true;
```

因为自动装箱使用的是`valueOf(long)`方法，在Long.java中的valueOf(long)代码如下，即在-128到127之间总是返回同一个对象

```java
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

### 尽量避免用包装类
自动装箱和拆箱只是模糊了原始类型和包装类使用，使得开发者无需显式来进行类型转换，但这并没有消除两者间的区别，在混合使用原始类型和包装类的时候，心里要清楚自动装箱拆箱的存在。
在<<Effective Java>>书中建议尽量少用包装类，尤其是在大量循环计算的情况下，自动装箱和拆箱会耗费大量的性能，见以下代码：

```java
Integer sum = 0;
for (int i = 0; i < 100000; i++) {
    sum += i;
}
```

在以上代码上，仅仅是因为声明了sum为Integer类型，导致在每一次循环中，sum都要先进行拆箱成int，进行了加法运算后又装箱成Integer，这样代码执行的效率很低。

### 包装类拆箱时需要注意`NullPoiterException`
当包装类为null时，拆箱会报NPE，这是一个在自动装箱和拆箱的情况容易忽视的问题：

```java
List<String> words = Arrays.asList("a", "b");
Map<String, Integer> wordCnt = new HashMap<>();
for (String w : words) {
    wordCnt.put(w, wordCnt.get(w) + 1); // wordCnt.get()得到的是null，拆箱会报NPE
}
```
