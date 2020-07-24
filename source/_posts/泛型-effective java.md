---
title: 泛型
tags:
  - effective java 3rd
  - 读书笔记
declare: true
abbrlink: 7678a22a
date: 2020-03-19 13:42:15
---
### 不要使用原始类型
+ 一个类或接口，它的声明有一个或多个类型参数（type parameters ），统称为泛型类型（generic types）。没有任何类型参数的泛型类型称为原始类型（List是List &lt;T&gt;的原始类型）。它们的存在主要是为了与没有泛型之前的代码相兼容。
+ 使用原始类型是合法的，但如果你使用原始类型，则会丧失泛型的所有安全性和表达上的优势
```java
private final List stamps = ... ; //警告
stamps.add(new Coin());// 如果有泛型，此处无法编译
(Stamp)stamps.get(0);//编译通过，运行ClassCastException
```

### 列表优于数组
1.1. 数组是协变的(covariant)，**即如果Child是Super的子类型，那么Child[]亦是Super[]的子类型。**
1.2. 泛型是不变的(invariant)，**对于任意两种不同的类型Type1，Type2，List<Type1>既不是List<Type>的子类型也不是父类型。**
```java
//运行时会抛出ArrayStoreException
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in";
//无法编译
List<Object> ol = new ArrayList<Long>();
```
<!-- more -->
2.1. 数组被**具体化了（reified）**，数组在运行时知道并强制执行它们的元素类型。如上所示，若将String放入Long数组，会抛出ArrayStoreException
2.2. 泛型通过**擦除（erasure）**来实现[JLS，4.6]。 这意味着它们只在编译时执行类型约束，并在运行时丢弃（或擦除）它们的元素类型信息。 擦除是允许泛型类型与不使用泛型的遗留代码自由互操作，从而确保在 Java 5 中平滑过渡到泛型。
3.1. 由于这些基本差异数组和泛型不能很好地在一起混合使用，new List<E>[] ， new
List<String>[] ， new E[] 都是不合法的。
3.2. 当将数组和列表混合使用，如将列表强制转换为数组类型时，得到泛型数组创建错误，或是未经检查的强制转换警告时，**使用集合会获得更好的类型安全性和互操作性**。
```java
public class Chooser<T> {
    private final T[] choiceArray;
    public Chooser(Collection<T>  choices) {
        choiceArray = (T[])choices.toArray();  //警告Unchecked cast:'java.lang.Object[]' to 'E[]'
    }
}

public class Chooser<T> {
    private final List<T> choiceArray;
    public Chooser(Collection<T>  choices) {
        choiceArray = new Arraylist<>(choices);
    }
}
```

### 优先使用泛型方法
+ 对于简单的泛型方法来说，**此方法编译不会生成警告，并提供类型安全性和易用性**。
```java
//无泛型
public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1);//编译警告
    result.addAll(s2);//编译警告
    return result;
}

public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```
+ 有时，需要创建一个不可改变但适用于许多不同类型的对象。因为泛型是通过擦除来实现的，所以可以使用单个对象进行所有必需的类型参数化，但是需要编写一个静态工厂方法来重复地为每个请求的类型参数化分配对象。 这种称为泛型单例工厂（generic singleton factory）的模式用于方法对象（ function objects），例如Collections.emptySet
```java
    @SuppressWarnings("rawtypes")
    public static final Set EMPTY_SET = new EmptySet<>();
    @SuppressWarnings("unchecked")
    public static final <T> Set<T> emptySet() {
        return (Set<T>) EMPTY_SET;
    }
```

### 使用限定通配符来增加API的灵活性
+ 为了获得最大的灵活性，对代表生产者或消费者的输入参数使用通配符类型。即PECS： producer-extends，consumer-super
```java
public class Stack<E> {
    public void push(E e);
    public E pop();
    public void pushAll(Iterable<? extends E> p){...};
    public void popAll(Collection<? super E> c){...};
}
```
+ Comparable和Comparator实例总是消费者，通常使用Comparable<? super T>，Comparator<? super T> 优于 Comparable<T>，Comparator<T>
```java
public static <T extends Comparable<? super T>> T max(List<T> list){...}
```
+ 如果类型参数在方法声明中只出现一次，将其替换为通配符。如果它是一个无限制的类型参数，请将其替换为无限制的通配符; 如果它是一个限定类型参数，则用限定通配符替换它。
swap 方法的客户端不需要面对更复杂的声明。辅助方法（swapHelper）具有我们认为对公共方法来说过于复杂的签名
```java
public static <E> void swap(List<E> list, int i, int j){...};

public static void swap(List<?> list, int i, int j) {
    //list.set(i, list.set(j, list.get(i))); 无法编译，因为无法确定类型
    swapHelper(list, i, j);
} 
private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

### 优先考虑类型安全的异构容器
+ 泛型 API 的通常用法（以集合 API 为例）限制了每个容器的固定数量的类型参数，可以通过**将类型参数放在键上而不是容器上来解决此限制**，可以使用 Class 对象作为此类型安全异构容器的键。 以这种方式使用的Class 对象称为**类型令牌**。 也可以使用自定义键类型。
```java
   public class Favorites {
        private Map<Class<?>, Object> favorites = new HashMap<>();
        public <T> void putFavorite(Class<T> type, T instance) {
            favorites.put(Objects.requireNonNull(type), instance);
        }
        public<T> T getFavorite(Class<T> type) {
            return type.cast(favorites.get(type));
        }
    }
```
+ Favorites 实例是类型安全的：当请求一个字符串时它永远不会返回一个整数。 它也是异构的：与普通Map 不同，所有的键都是不同的类型。 因此，我们将 Favorites 称为类型安全异构容器（typesafe heterogeneous container）
