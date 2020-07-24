---
title: 枚举
tags:
  - effective java 3rd
  - 读书笔记
declare: true
abbrlink: 633a03ca
date: 2020-03-25 16:41:57
---
### 使用枚举类型替代整型常量
+ 在枚举类型添加到java之前，表示枚举的常见模式是一组名为int的常量，每个类型的成员都有一个常量，没有简单的方法将 int 枚举常量转换为可打印的字符串。如果你打印这样一个常量或者从调试器中显示出来，你看到的只是一个数字。 没有可靠的方法来迭代组中的所有 int 枚举常量，甚至无法获得int 枚举组的大小(String枚举亦然)。
<!-- more -->
```java
public static final int APPLE_FUJI = 0;
public static final int APPLE_PIPPIN = 1;
public static final int APPLE_GRANNY_SMITH = 2;
```
+ enum是java.lang.Enum的子类，它实现了Comparable和Serializable，并提供了valueof(),values()等方法便于使用。
+ 要将数据与枚举常量相关联，需声明实例属性并编写一个构造方法，**构造方法带有数据并将数据保存在属性中；由于枚举不变的，所以所有的属性都应该是final的；最好将属性私有化并提供公共访问方法。**
+ 在枚举类型中声明一个抽象的方法，并用常量特定的类主体中的每个常量的具体方法重写它。这种方法被称为特定于常量（constant-specific）。
```java
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;
    Operation(String symbol) { 
        this.symbol = symbol; 
    }
    public String getSymbol() { 
        return symbol; 
    }
    public abstract double apply(double x, double y);
}
```

### 使用实例属性替代序数
```java
public enum Ensemble {
    SOLO, DUET, TRIO, QUARTET, QUINTET,
    SEXTET, SEPTET, OCTET, NONET, DECTET;
    public int numberOfMusicians() { 
        return ordinal() + 1; 
    }
}
```
+ 所有枚举都有一个 ordinal 方法，它返回每个枚举常量类型的数值位置，上面的枚举能正常工作，但对于维护来说则是一场噩梦。如果常量被重新排序，调用者将受到影响。
+ **永远不要从枚举的序号中得出与它相关的值; 请将其保存在实例属性中。**
```java
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
    SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
    NONET(9), DECTET(10), TRIPLE_QUARTET(12);
    private final int numberOfMusicians;

    Ensemble(int size) {
        this.numberOfMusicians = size;
    }

    public int numberOfMusicians() {
        return numberOfMusicians;
    }
}
```