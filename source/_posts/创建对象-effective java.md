---
title: 创建对象-effective java
tags:
  - effective java 3rd
  - 读书笔记
declare: true
abbrlink: bff37142
date: 2020-02-23 14:00:53
---
### 使用静态工厂方法替代构造
``` java
public static Boolean valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;
}
```
+ 静态工厂方法是有名字的，在类中需要具有**相同签名(方法的名称和参数类型)**的多个构造方法时，用静态工厂方法替换构造，并仔细选择名字来突出它们差异。
+ 静态工厂方法不需要每次调用时都创建一个对象，这是享元模式（Flyweight）的基础，如果经常请求等价对象，那么将极大的提高性能。
+ 静态工厂方法可以返回其返回类型的子类型对象。
<!-- more -->
### 当构造方法参数过多时使用builder模式
```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;
    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;
        // Optional parameters - initialized to default values
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        } 
        public Builder calories(int val) {
            calories = val;
            return this;
        } 
        public Builder fat(int val) {
            fat = val;
            return this;
        } 
        public Builder sodium(int val) {
            sodium = val;
            return this;
        } 
        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        } 
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    } 
    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

### 使用私有构造方法执行非实例化
+ 实用类（utility classes）（例如Collections）不是设计用来被实例化的：一个实例是没有意义的。
+ 在没有显式构造方法的情况下，编译器提供了一个公共的、无参的默认构造方法，可以通过包含一个私有构造方法来实现类的非实例化
```java
public class UtilityClass {
    private UtilityClass() {
        throw new AssertionError();
    } 
}
```

### 使用依赖注入取代硬链接资源
+ 许多类依赖一个或多个底层资源。例如有一个拼写检查类
```
错误的,每种语言都有自己的字典，特殊的字典被用于特殊的词汇表
public class SpellChecker {
    private static final Lexicon dictionary = ...;
    private SpellChecker() {} // Noninstantiable
    public static boolean isValid(String word) { ... }
    public static List<String> suggestions(String typo) { ... }
}

实现这一需求的简单模式是在创建新实例时将资源传递到构造方法中，这是
依赖项注入（dependency injection）的一种形式：字典是拼写检查器的一个依赖项，当它创建时被注入到拼写检查器
中
public class SpellChecker {
    private final Lexicon dictionary;
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    } 
    public boolean isValid(String word) { ... }
    public List<String> suggestions(String typo) { ... }
}
```

### 避免创建不必要对象
+ 在需要时重用一个对象而不是创建一个新的相同功能对象通常是恰当的。
```java
String.matches每次都会创建一个Pattren对象,创建Pattern实例是昂贵的，因为它需要将正则表达式编译成有限状态机
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
    * "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
将正则表达式显式编译为一个 Pattern 实例（不可变），缓存它
private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
static boolean isRomanNumeral(String s) {
    return ROMAN.matcher(s).matches();
}
```
```java
变量 sum 被声明成了Long 而不是 long ，这意味着程序构造了大量不必要的 Long 实例
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;
    return sum;
}
```
+ 除非池中的对象**非常重量级**，否则通过维护自己的对象池来避免对象创建是一个坏主意,现代JVM 实现具有高度优化的垃圾收集器，它们在轻量级对象上轻松胜过此类对象池。