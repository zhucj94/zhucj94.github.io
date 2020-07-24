---
title: Lambdas和Streams
tags:
  - effective java 3rd
  - 读书笔记
declare: true
abbrlink: '33749838'
date: 2020-03-28 17:40:14
---
###  优先使用标准的函数式接口
+ 有了lambda后，编写 API 的最佳实践已经发生了很大的变化，例如模板方法模式，其中一个子类重写原始方法以专门化其父类的行为，变得没有那么吸引人。现代替代的选择是提供一个静态工厂或构造方法来接受函数对象以达到相同的效果。
例如 LinkedHashMap，可以通过重写removeEldestEntry 方法将此类用作缓存，每次将新的key 值加入到 map时都会调用该方法，当此方法返回true时，map将删除传递给该方法的最久条目。
<!-- more -->
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > 100;
}
```
如果LinkedHashMap是现在编写，可以用lambda来实现，增加一个构造或静态工厂方法
```java
private BiPredicate<Map<K,V>, Map.Entry<K,V>> removeEldestEntry;
public LinkedHashMap(BiPredicate<Map<K,V>, Map.Entry<K,V>> param) {
    removeEldestEntry = param;
}
```
+ java.util.Function中有43个几口，但是只有6个基本接口，其余的均是其派生接口

接口 | 方法 |  示例
:-: | :-: | :-: 
UnaryOperator&lt;T&gt; | T apply(T t) |  String::toLowerCase
BinaryOperator&lt;T&gt; | T apply(T t1, T t2) |  BigInteger::add
Predicate&lt;T&gt; | boolean test(T t) |  Collection::isEmpty
Function&lt;T,R&gt; | R apply(T t) |  Arrays::asList
Supplier&lt;T&gt; | T get()  |  Instant::now
Consumer&lt;T&gt; | void accept(T t) |  System.out::println

+ 始终使用 @FunctionalInterface 注解标注你的函数式接口。
它告诉读者该接口是为了实现lambda表达式而设计的。 
它使接口可靠，因为除非只有一个抽象方法，否则接口不会编译。 
它可以防止维护人员在接口发生变化时不小心地将抽象方法添加到接口中。 

// TODO
