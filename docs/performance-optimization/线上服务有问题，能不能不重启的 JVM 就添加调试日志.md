# 线上服务有问题，能不能不重启的 JVM 就添加调试日志？

> 作者：高扬
>
> 链接：https://tech.meituan.com/2019/02/28/java-dynamic-trace.html

在遥远的希艾斯星球爪哇国塞沃城中，两名年轻的程序员正在为一件事情苦恼，程序出问题了，一时看不出问题出在哪里，于是有了以下对话：

“Debug 一下吧。”

“线上机器，没开 Debug 端口。”

“看日志，看看请求值和返回值分别是什么？”

“那段代码没打印日志。”

“改代码，加日志，重新发布一次。”

“怀疑是线程池的问题，重启会破坏现场。”

长达几十秒的沉默之后：“据说，排查问题的最高境界，就是只通过 Review 代码来发现问题。”

比几十秒长几十倍的沉默之后：“我轮询了那段代码一十七遍之后，终于得出一个结论。”

“结论是？”

“我还没到达只通过 Review 代码就能发现问题的至高境界。”

## 从 JSP 说起

对于大多数 Java 程序员来说，早期的时候，都会接触到一个叫做 JSP（Java Server Pages）的技术。虽然这种技术，在前后端代码分离、前后端逻辑分离、前后端组织架构分离的今天来看，已经过时了，但是其中还是有一些有意思的东西，值得拿出来说一说。

当时刚刚处于 Java 入门时期的我们，大多数精力似乎都放在了 JSP 的页面展示效果上了：

“这个表格显示的行数不对”

“原来是 for 循环写的有问题，改一下，刷新页面再试一遍”

“嗯，好了，表格显示没问题了，但是，登录人的姓名没取到啊，是不是 Sesstion 获取有问题？”

“有可能，我再改一下，一会儿再刷新试试”

……

在一遍一遍修改代码刷新浏览器页面重试的时候，我们自己也许并没有注意到一件很酷的事情：我们修改完代码，居然只是简单地刷新一遍浏览器页面，修改就生效了，整个过程并没有重启 JVM。按照我们的常识，Java 程序一般都是在启动时加载类文件，如果都像 JSP 这样修改完代码，不用重启就生效的话，那文章开头的问题就可以解决了啊：Java 文件中加一段日志打印的代码，不重启就生效，既不破坏现场，又可以定位问题。忍不住试一试：修改、编译、替换 class 文件。额，不行，新改的代码并没有生效。那为什么偏偏 JSP 可以呢？让我们先来看看 JSP 的运行原理。

当我们打开浏览器，请求访问一个 JSP 文件的时候，整个过程是这样的:

![JSP文件处理过程](http://img.uprogrammer.cn/static/20200823202220.png)

JSP 文件修改过后，之所以能及时生效，是因为 Web 容器（Tomcat）会检查请求的 JSP 文件是否被更改过。如果发生过更改，那么就将 JSP 文件重新解析翻译成一个新的 Sevlet 类，并加载到 JVM 中。之后的请求，都会由这个新的 Servet 来处理。这里有个问题，根据 Java 的类加载机制，在同一个 ClassLoader 中，类是不允许重复的。为了绕开这个限制，Web 容器每次都会创建一个新的 ClassLoader 实例，来加载新编译的 Servlet 类。之后的请求都会由这个新的 Servlet 来处理，这样就实现了新旧 JSP 的切换。

HTTP 服务是无状态的，所以 JSP 的场景基本上都是一次性消费，这种通过创建新的 ClassLoader 来“替换” class  的做法行得通，但是对于其他应用，比如 Spring 框架，即便这样做了，对象多数是单例，对于内存中已经创建好的对象，我们无法通过这种创建新的 ClassLoader 实例的方法来修改对象行为。

我就是想不重启应用加个日志打印，就这么难吗？

## Java 对象行为

既然 JSP 的办法行不通，那我们来看看还有没有其他的办法。仔细想想，我们会发现，文章开头的问题本质上是动态改变内存中已存在对象的行为的问题。所以，我们得先弄清楚 JVM 中和对象行为有关的地方在哪里，有没有更改的可能性。

我们都知道，对象使用两种东西来描述事物：行为和属性。举个例子：

```java
public class Person {

    private int age;

    private String name;

    public void speak(String str) {
        System.out.println(str);
    }

    public Person(int age, String name) {
        this.age = age;
        this.name = name;
    }
}
```

上面 Person 类中 age 和 name 是属性，speak 是行为。对象是类的事例，每个对象的属性都属于对象本身，但是每个对象的行为却是公共的。举个例子，比如我们现在基于 Person 类创建了两个对象， personA 和  personB：

```java
Person personA = new Person(43, "lixunhuan");

personA.speak("我是李寻欢");

Person personB = new Person(23, "afei");

personB.speak("我是阿飞");
```

personA 和 personB 有各自的姓名和年龄，但是有共同的行为：speak。想象一下，如果我们是 Java 语言的设计者，我们会怎么存储对象的行为和属性呢？

“很简单，属性跟着对象走，每个对象都存一份。行为是公共的东西，抽离出来，单独放到一个地方。”

“咦？抽离出公共的部分，跟代码复用好像啊。”

“大道至简，很多东西本来都是殊途同归。”

也就是说，第一步我们首先得找到存储对象行为的这个公共的地方。一番搜索之后，我们发现这样一段描述：

> Method area is created on virtual machine startup, shared among all Java virtual machine threads and it is logically part of heap area. It stores per-class structures such as the run-time constant pool, field and method data, and the code for methods and constructors.

Java 的对象行为（方法、函数）是存储在方法区的。

“方法区中的数据从哪来？”

“方法区中的数据是类加载时从 class 文件中提取出来的。”

“class 文件从哪来？”

“从 Java 或者其他符合 JVM 规范的源代码中编译而来。”

“源代码从哪来？”

“废话，当然是手写！”

“倒着推，手写没问题，编译没问题，至于加载……有没有办法加载一个已经加载过的类呢？如果有的话，我们就能修改字节码中目标方法所在的区域，然后重新加载这个类，这样方法区中的对象行为（方法）就被改变了，而且不改变对象的属性，也不影响已经存在对象的状态，那么就可以搞定这个问题了。可是，这岂不是违背了 JVM 的类加载原理？毕竟我们不想改变 ClassLoader。”

“少年，可以去看看 `java.lang.instrument.Instrumentation`。”

## java.lang.instrument.Instrumentation

看完文档之后，我们发现这么两个接口： `redefineClasses` 和 `retransformClasses`。一个是重新定义 class，一个是修改 class。这两个大同小异，看 `redefineClasses` 的说明：

> This method is used to replace the definition of a class without reference to the existing class file bytes, as one might do when recompiling from source for fix-and-continue debugging. Where the existing class file bytes are to be transformed \(for example in bytecode instrumentation\) retransformClasses should be used.

都是替换已经存在的 class 文件，`redefineClasses` 是自己提供字节码文件替换掉已存在的 class 文件，`retransformClasses` 是在已存在的字节码文件上修改后再替换之。

当然，运行时直接替换类很不安全。比如新的 class 文件引用了一个不存在的类，或者把某个类的一个 field 给删除了等等，这些情况都会引发异常。所以如文档中所言，instrument 存在诸多的限制：

> The redefinition may change method bodies, the constant pool and attributes. The redefinition must not add, remove or rename fields or methods, change the signatures of methods, or change inheritance. These restrictions maybe be lifted in future versions. The class file bytes are not checked, verified and installed until after the transformations have been applied, if the resultant bytes are in error this method will throw an exception.

我们能做的基本上也就是简单修改方法内的一些行为，这对于我们开头的问题，打印一段日志来说，已经足够了。当然，我们除了通过 `retransform` 来打印日志，还能做很多其他非常有用的事情，这个下文会进行介绍。

那怎么得到我们需要的 class 文件呢？一个最简单的方法，是把修改后的 Java 文件重新编译一遍得到 class 文件，然后调用 `redefineClasses` 替换。但是对于没有（或者拿不到，或者不方便修改）源码的文件我们应该怎么办呢？其实对于 JVM 来说，不管是 Java 也好，Scala 也好，任何一种符合 JVM 规范的语言的源代码，都可以编译成 class 文件。JVM 的操作对象是 class 文件，而不是源码。所以，从这种意义上来讲，我们可以说 “JVM 跟语言无关”。既然如此，不管有没有源码，其实我们只需要修改 class 文件就行了。

## 直接操作字节码

Java 是软件开发人员能读懂的语言，class 字节码是 JVM 能读懂的语言，class 字节码最终会被 JVM 解释成机器能读懂的语言。无论哪种语言，都是人创造的。所以，理论上（实际上也确实如此）人能读懂上述任何一种语言，既然能读懂，自然能修改。只要我们愿意，我们完全可以跳过 Java 编译器，直接写字节码文件，只不过这并不符合时代的发展罢了，毕竟高级语言设计之始就是为我们人类所服务，其开发效率也比机器语言高很多。

对于人类来说，字节码文件的可读性远远没有 Java 代码高。尽管如此，还是有一些杰出的程序员们创造出了可以用来直接编辑字节码的框架，提供接口可以让我们方便地操作字节码文件，进行注入修改类的方法，动态创造一个新的类等等操作。其中最著名的框架应该就是 ASM 了，cglib、Spring 等框架中对于字节码的操作就建立在 ASM 之上。

我们都知道，Spring 的 AOP 是基于动态代理实现的，Spring 会在运行时动态创建代理类，代理类中引用被代理类，在被代理的方法执行前后进行一些神秘的操作。那么，Spring 是怎么在运行时创建代理类的呢？动态代理的美妙之处，就在于我们不必手动为每个需要被代理的类写代理类代码，Spring 在运行时会根据需要动态地创造出一个类，这里创造的过程并非通过字符串写 Java 文件，然后编译成 class 文件，然后加载。Spring 会直接“创造”一个 class 文件，然后加载，创造 class 文件的工具，就是 ASM 了。

到这里，我们知道了用 ASM 框架直接操作 class 文件，在类中加一段打印日志的代码，然后调用 `retransformClasses` 就可以了。

## BTrace

截止到目前，我们都是停留在理论描述的层面。那么如何进行实现呢？先来看几个问题：

1.  在我们的工程中，谁来做这个寻找字节码，修改字节码，然后 retransform 的动作呢？我们并非先知，不可能知道未来有没有可能遇到文章开头的这种问题。考虑到性价比，我们也不可能在每个工程中都开发一段专门做这些修改字节码、重新加载字节码的代码。
2.  如果 JVM 不在本地，在远程呢？
3.  如果连 ASM 都不会用呢？能不能更通用一些，更“傻瓜”一些。

幸运的是，因为有 BTrace 的存在，我们不必自己写一套这样的工具了。什么是 BTrace 呢？[BTrace](https://github.com/btraceio/btrace "BTrace")已经开源，项目描述极其简短：

> A safe, dynamic tracing tool for the Java platform.

BTrace 是基于 Java 语言的一个安全的、可提供动态追踪服务的工具。BTrace 基于 ASM、Java Attach Api、Instruments 开发，为用户提供了很多注解。依靠这些注解，我们可以编写 BTrace 脚本（简单的 Java 代码）达到我们想要的效果，而不必深陷于 ASM 对字节码的操作中不可自拔。

看 BTrace 官方提供的一个简单例子：拦截所有 java.io 包中所有类中以 read 开头的方法，打印类名、方法名和参数名。当程序 IO 负载比较高的时候，就可以从输出的信息中看到是哪些类所引起，是不是很方便？

```java
package com.sun.btrace.samples;

import com.sun.btrace.annotations.*;
import com.sun.btrace.AnyType;
import static com.sun.btrace.BTraceUtils.*;

/**
 * This sample demonstrates regular expression
 * probe matching and getting input arguments
 * as an array - so that any overload variant
 * can be traced in "one place". This example
 * traces any "readXX" method on any class in
 * java.io package. Probed class, method and arg
 * array is printed in the action.
 */
@BTrace public class ArgArray {
    @OnMethod(
        clazz="/java\\.io\\..*/",
        method="/read.*/"
    )
    public static void anyRead(@ProbeClassName String pcn, @ProbeMethodName String pmn, AnyType[] args) {
        println(pcn);
        println(pmn);
        printArray(args);
    }
}
```

再来看另一个例子：每隔 2 秒打印截止到当前创建过的线程数。

```java
package com.sun.btrace.samples;

import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;
import com.sun.btrace.annotations.Export;

/**
 * This sample creates a jvmstat counter and
 * increments it everytime Thread.start() is
 * called. This thread count may be accessed
 * from outside the process. The @Export annotated
 * fields are mapped to jvmstat counters. The counter
 * name is "btrace." + <className> + "." + <fieldName>
 */
@BTrace public class ThreadCounter {

    // create a jvmstat counter using @Export
    @Export private static long count;

    @OnMethod(
        clazz="java.lang.Thread",
        method="start"
    )
    public static void onnewThread(@Self Thread t) {
        // updating counter is easy. Just assign to
        // the static field!
        count++;
    }

    @OnTimer(2000)
    public static void ontimer() {
        // we can access counter as "count" as well
        // as from jvmstat counter directly.
        println(count);
        // or equivalently ...
        println(Counters.perfLong("btrace.com.sun.btrace.samples.ThreadCounter.count"));
    }
}
```

看了上面的用法是不是有所启发？忍不住冒出来许多想法。比如查看 HashMap 什么时候会触发 rehash，以及此时容器中有多少元素等等。

有了 BTrace，文章开头的问题可以得到完美的解决。至于 BTrace 具体有哪些功能，脚本怎么写，这些 Git 上 BTrace 工程中有大量的说明和举例，网上介绍 BTrace 用法的文章更是恒河沙数，这里就不再赘述了。

我们明白了原理，又有好用的工具支持，剩下的就是发挥我们的创造力了，只需在合适的场景下合理地进行使用即可。

既然 BTrace 能解决上面我们提到的所有问题，那么 BTrace 的架构是怎样的呢？

BTrace 主要有下面几个模块：

1.  BTrace 脚本：利用 BTrace 定义的注解，我们可以很方便地根据需要进行脚本的开发。
2.  Compiler：将 BTrace 脚本编译成 BTrace class 文件。
3.  Client：将 class 文件发送到 Agent。
4.  Agent：基于 Java 的 Attach Api，Agent 可以动态附着到一个运行的 JVM 上，然后开启一个 BTrace Server，接收 client 发过来的 BTrace 脚本；解析脚本，然后根据脚本中的规则找到要修改的类；修改字节码后，调用 Java Instrument 的 retransform 接口，完成对对象行为的修改并使之生效。

整个 BTrace 的架构大致如下：

![BTrace工作流程](http://img.uprogrammer.cn/static/20200823204511.png)

BTrace 最终借 Instruments 实现 class 的替换。如上文所说，出于安全考虑，Instruments 在使用上存在诸多的限制，BTrace 也不例外。BTrace 对 JVM 来说是“只读的”，因此 BTrace 脚本的限制如下：

1.  不允许创建对象
2.  不允许创建数组
3.  不允许抛异常
4.  不允许 catch 异常
5.  不允许随意调用其他对象或者类的方法，只允许调用 `com.sun.btrace.BTraceUtils` 中提供的静态方法（一些数据处理和信息输出工具）
6.  不允许改变类的属性
7.  不允许有成员变量和方法，只允许存在 **static public void** 方法
8.  不允许有内部类、嵌套类
9.  不允许有同步方法和同步块
10. 不允许有循环
11. 不允许随意继承其他类（当然，`java.lang.Object` 除外）
12. 不允许实现接口
13. 不允许使用 assert
14. 不允许使用 `Class` 对象

如此多的限制，其实可以理解。BTrace 要做的是，虽然修改了字节码，但是除了输出需要的信息外，对整个程序的正常运行并没有影响。

## Arthas

BTrace 脚本在使用上有一定的学习成本，如果能把一些常用的功能封装起来，对外直接提供简单的命令即可操作的话，那就再好不过了。阿里的工程师们早已想到这一点，就在去年（2018 年 9 月份），阿里巴巴开源了自己的 Java 诊断工具——[Arthas](https://github.com/alibaba/arthas "Arthas")。Arthas 提供简单的命令行操作，功能强大。究其背后的技术原理，和本文中提到的大致无二。Arthas 的文档很全面，想详细了解的话可以戳[这里](https://alibaba.github.io/arthas/ "Arthas")。

本文旨在说明 Java 动态追踪技术的来龙去脉，掌握技术背后的原理之后，只要愿意，各位读者也可以开发出自己的“冰封王座”出来。

## 尾声：三生万物

现在，让我们试着站在更高的地方“俯瞰”这些问题。

Java 的 Instruments 给运行时的动态追踪留下了希望，Attach API 则给运行时动态追踪提供了“出入口”，ASM 则大大方便了“人类”操作 Java 字节码的操作。

基于 Instruments 和 Attach API 前辈们创造出了诸如 JProfiler、Jvisualvm、BTrace、Arthas 这样的工具。以 ASM 为基础发展出了 cglib、动态代理，继而是应用广泛的 Spring AOP。

Java 是静态语言，运行时不允许改变数据结构。然而，Java 5 引入 Instruments，Java 6 引入 Attach API 之后，事情开始变得不一样了。虽然存在诸多限制，然而，在前辈们的努力下，仅仅是利用预留的近似于“只读”的这一点点狭小的空间，仍然创造出了各种大放异彩的技术，极大地提高了软件开发人员定位问题的效率。

计算机应该是人类有史以来最伟大的发明之一，从电磁感应磁生电，到高低电压模拟 0 和 1 的比特，再到二进制表示出几种基本类型，再到基本类型表示出无穷的对象，最后无穷的对象组合交互模拟现实生活乃至整个宇宙。

两千五百年前，《道德经》有言：“道生一，一生二，二生三，三生万物。”

两千五百年后，计算机的发展过程也大抵如此吧。

# 更多好文

[微服务网关 Zuul 的知识点，一篇文章就讲全了](https://mp.weixin.qq.com/s/rop33BKERIcV0JtXwrZXmQ)

[学会MyBatis动态SQL，写SQL也能爽到飞起](https://mp.weixin.qq.com/s/RIPt-2LhMACHkle7P9JCtg)

[一文搞懂 Spring Boot 启动原理](https://mp.weixin.qq.com/s/YT9k8PpEKTWzXcBXA8fftw)

[线上服务器 CPU 100%, 如何一键定位](https://mp.weixin.qq.com/s/xmYpkaeq3lFjxx-2jOhEyA)

[冒着挂科的风险也要给你们看的 Spring Cloud 入门总结](https://mp.weixin.qq.com/s/aTuPgiyr2pJGs5iRuvq_uQ)