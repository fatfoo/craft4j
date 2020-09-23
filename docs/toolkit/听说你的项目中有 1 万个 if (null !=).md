# 听说你的项目中有 1 万个 if (null !=)

![](http://img.uprogrammer.cn/static/20200923201125.png)

## 判空灾难

![](http://img.uprogrammer.cn/static/20200923194031.png)

作为搬砖党的一族们，我们对判空一定再熟悉不过了，不要跟我说你很少进行判空，除非你喜欢 NullPointerException。

不过 NullPointerException 对于很多猿们来说，也是 Exception 家族中最亲近的一员了。

![](http://img.uprogrammer.cn/static/20200923194113.png)

为了避免 NullPointerException 来找我们，我们经常会进行如下操作。

```java
if (data != null) {
    do sth.
}
```

如果一个类中多次使用某个对象，那你可能要一顿操作，so:

![](http://img.uprogrammer.cn/static/20200923194147.png)

“世界第九大奇迹”就这样诞生了。Maybe 你会想，项目中肯定不止你一个人会这样一顿操作，然后按下 Command+Shift+F，真相就在眼前：

![](http://img.uprogrammer.cn/static/20200923194219.png)

What，我们有接近一万行的代码都是在判空？

![](http://img.uprogrammer.cn/static/20200923194254.png)

好了，接下来，要进入正题了。

## NullObject 模式

对于项目中无数次的判空，对代码质量整洁度产生了十分之恶劣的影响，对于这种现象，我们称之为“判空灾难”。

那么，这种现象如何治理呢，你可能听说过 NullObject 模式，不过这不是我们今天的武器，但是还是需要介绍一下 NullObject 模式。

什么是 NullObject 模式呢？

> In object-oriented computer programming, a null object is an object with no referenced value or with defined neutral ("null") behavior. The null object design pattern describes the uses of such objects and their behavior (or lack thereof).

以上解析来自 Wikipedia。

NullObject 模式首次发表在“ 程序设计模式语言 ”系列丛书中。一般的，在面向对象语言中，对对象的调用前需要使用判空检查，来判断这些对象是否为空，因为在空引用上无法调用所需方法。

空对象模式的一种典型实现方式如下图所示(图片来自网络)：

![](http://img.uprogrammer.cn/static/20200923194403.png)

示例代码如下（命名来自网络，哈哈到底是有多懒）：

Nullable 是空对象的相关操作接口，用于确定对象是否为空，因为在空对象模式中，对象为空会被包装成一个 Object，成为 Null Object，该对象会对原有对象的所有方法进行空实现。

```java
public interface Nullable {

    boolean isNull();

}
```

这个接口定义了业务对象的行为。

```java
public interface DependencyBase extends Nullable {

    void Operation();

}
```

这是该对象的真实类，实现了业务行为接口 DependencyBase 与空对象操作接口 Nullable。

```java
public class Dependency implements DependencyBase, Nullable {

    @Override
    public void Operation() {
        System.out.print("Test!");
    }

    @Override
    public boolean isNull() {
        return false;
    }

}
```

这是空对象，对原有对象的行为进行了空实现。

```java
public class NullObject implements DependencyBase{

    @Override
    public void Operation() {
        // do nothing
    }

    @Override
    public boolean isNull() {
        return true;
    }

}
```

在使用时，可以通过工厂调用方式来进行空对象的调用，也可以通过其他如反射的方式对对象进行调用（一般多耗时几毫秒）在此不进行详细叙述。

```java
public class Factory {

    public static DependencyBase get(Nullable dependencyBase){
        if (dependencyBase == null){
            return new NullObject();
        }
        return new Dependency();
    }

}
```

这是一个使用范例，通过这种模式，我们不再需要进行对象的判空操作，而是可以直接使用对象，也不必担心 NPE（NullPointerException）的问题。

```java
public class Client {

    public void test(DependencyBase dependencyBase){
        Factory.get(dependencyBase).Operation();
    }

}
```

关于空对象模式，更具体的内容大家也可以多找一找资料，上述只是对 NullObject 的简单介绍，但是，今天我要推荐的是一款协助判空的插件 NR Null Object，让我们来优雅地进行判空，不再进行一顿操作来定义繁琐的空对象接口与空独享实现类。

## NR Null Object

NR Null Object 是一款适用于 Android Studio、IntelliJ IDEA、PhpStorm、WebStorm、PyCharm、RubyMine、AppCode、CLion、GoLand、DataGrip 等 IDEA 的 Intellij 插件。其可以根据现有对象，便捷快速生成其空对象模式需要的组成成分，其包含功能如下：

- 分析所选类可声明为接口的方法；
- 抽象出公有接口；
- 创建空对象，自动实现公有接口；
- 对部分函数进行可为空声明；
- 可追加函数进行再次生成；
- 自动的函数命名规范。

让我们来看一个使用范例：

![](http://img.uprogrammer.cn/static/20200923194744.png)

怎么样，看起来是不是非常快速便捷，只需要在原有需要进行多次判空的对象中，邮件弹出菜单，选择 Generate，并选择 NR Null Object 即可自动生成相应的空对象组件。

那么如何来获得这款插件呢？

## 安装方式

可以直接通过 IDEA 的 Preferences 中的 Plugins 仓库进行安装。

选择 Preferences → Plugins → Browse repositories

![](http://img.uprogrammer.cn/static/20200923194831.png)

搜索“NR Null Oject”或者“Null Oject”进行模糊查询，点击右侧的 Install，restart IDEA 即可。

![](http://img.uprogrammer.cn/static/20200923194853.png)

## Optional

还有一种方式是使用 Java8 特性中的 Optional 来进行优雅地判空，Optional 来自官方的介绍如下：

> A container object which may or may not contain a non-null value. If a value is present, `isPresent()` will return `true` and `get()` will return the value.

一个可能包含也可能不包含非 null 值的容器对象。 如果存在值，`isPresent()` 将返回 true，`get()` 将返回该值。

话不多说，举个例子。

![](http://img.uprogrammer.cn/static/20200923194957.png)

有如下代码，需要获得 Test2 中的 Info 信息，但是参数为 Test4，我们要一层层的申请，每一层都获得的对象都可能是空，最后的代码看起来就像这样。

```java
public String testSimple(Test4 test) {
    if (test == null) {
        return "";
    }
    if (test.getTest3() == null) {
        return "";
    }
    if (test.getTest3().getTest2() == null) {
        return "";
    }
    if (test.getTest3().getTest2().getInfo() == null) {
        return "";
    }
    return test.getTest3().getTest2().getInfo();
}
```

但是使用 Optional 后，整个就都不一样了。

```java
public String testOptional(Test test) {
    return Optional.ofNullable(test).flatMap(Test::getTest3)
            .flatMap(Test3::getTest2)
            .map(Test2::getInfo)
            .orElse("");
}
```

1. Optional.ofNullable(test)，如果 test 为空，则返回一个单例空 Optional 对象，如果非空则返回一个 Optional 包装对象，Optional 将 test 包装；

```java
public static <T> Optional<T> ofNullable(T value) {
    return value == null ? empty() : of(value);
}
```

2. flatMap(Test::getTest3) 判断 test 是否为空，如果为空，继续返回第一步中的单例 Optional 对象，否则调用 Test 的 getTest3 方法；

```java
public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Objects.requireNonNull(mapper.apply(value));
    }
}
```

3. flatMap(Test3::getTest2) 同上调用 Test3 的 getTest2 方法；

4. map(Test2::getInfo) 同 flatMap 类似，但是 flatMap 要求 Test3::getTest2 返回值为 Optional 类型，而 map 不需要，flatMap 不会多层包装，map 返回会再次包装 Optional；

```java
public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Optional.ofNullable(mapper.apply(value));
    }
}
```

5. orElse("") 获得 map 中的 value，不为空则直接返回 value，为空则返回传入的参数作为默认值。

```java
public T orElse(T other) {
    return value != null ? value : other;
}
```

怎么样，使用 Optional 后我们的代码是不是瞬间变得非常整洁，或许看到这段代码你会有很多疑问，针对复杂的一长串判空，Optional 有它的优势，但是对于简单的判空使用 Optional 也会增加代码的阅读成本、编码量以及团队新成员的学习成本。

如果直接使用 Java8 中的 Optional，需要保证安卓 API 级别在 24 及以上。

![](http://img.uprogrammer.cn/static/20200923195554.png)

你也可以直接引入 Google 的 Guava。（啥是 Guava？来自官方的提示）

> Guava is a set of core libraries that includes new collection types (such as multimap and multiset), immutable collections, a graph library, functional types, an in-memory cache, and APIs/utilities for concurrency, I/O, hashing, primitives, reflection, string processing, and much more!

引用方式，就像这样：

```groovy
dependencies {
  compile 'com.google.guava:guava:27.0-jre'
  // or, for Android:
  api 'com.google.guava:guava:27.0-android'
}
```

不过 IDEA 默认会显示黄色，提示让你将 Guava 表达式迁移到 Java Api 上。

![](http://img.uprogrammer.cn/static/20200923195643.png)

当然，你也可以通过在 Preferences 搜索 "Guava" 来 Kill 掉这个 Yellow 的提示。

![](http://img.uprogrammer.cn/static/20200923195720.png)

关于 Optional 使用还有很多技巧，感兴趣可以查阅 Guava 和 Java8 相关书籍和文档。

使用 Optional 具有如下优点：

1.  将防御式编程代码完美包装
2.  链式调用
3.  有效避免程序代码中的空指针

但是也同样具有一些缺点：

1.  流行性不是非常理想，团队新成员需要学习成本
2.  安卓中需要引入 Guava，需要团队每个人处理 IDEA 默认提示，或者忍受黄色提示
3.  有时候代码阅读看起来可能会如下图所示：

![](http://img.uprogrammer.cn/static/20200923195811.png)

---

作者：李良逸

原文：http://blog.imuxuan.com/archives/86

---

如果你看完本文有收获，欢迎关注微信公众号：精进Java(ID: **craft4j**)，更多 Java 后端与架构的干货等你一起学习与交流。