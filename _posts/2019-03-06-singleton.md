---
layout: post
category: "设计模式"
title:  "看似简单的单例模式"
tags: [Java, 设计模式, 多线程]
---


* content
{:toc}

![](https://picsum.photos/800/300/?image=727)
单例模式作为较为常用的设计模式，实现思路看似简单，但要写出一个在多线程环境中能正确运行的单例模式却并不容易。





## 最简单的实现思路
一说起单例模式，我们脑中自然会思路：**用一个成员变量存储这个单例，只在第一次用到时进行实例化。**于是大手一挥，写下下面的代码：
```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {} // 将构造方法设置为 private，禁止私自创建对象

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

这种方式看似满足了要求，但其实只在单线程环境中能满足条件，放在多线程环境中，就变成了一个**典型的 check-then-act 问题，没有任何线程同步机制下会引发竞态**。上面的实现如果两个线程同时执行`if (instance == null)`代码，则会同时判断`instance`变量为`null`，最终创建两个对象实例，这样就不满足单例模式的初衷了。

## 简单加锁
那既然没有线程同步机制，加上就是了，我们最容易想到的线程同步机制就是加锁，于是有了第二版实现：
```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}

    public static Singleton getInstance() {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton();
            }
        }

        return instance;
    }
}
```
我们用`synchronized`内部锁对可能产生竞态的代码块进行了加锁，确保临界区内的代码每次只有一个线程在执行。这样确实做到了线程安全，但在每次调用`getInstance()`方法时都需要进行加锁和释放操作，由于我们的竞态只会在实例化对象时发生，之后的加锁和释放操作毫无意义，可能会产生不必要的上下文切换开销，也就是锁的粒度太大，还需进行优化。（这也是 HashTable 和 Vector 这类集合被弃用的原因）

之后的实现就有些套路的意思了。

## 双重检查锁定
在上一版的基础上出现了双重检查锁定 Double-checked Locking（DCL），具体方式是**只当`instance`变量为`null`的前提下再进行第二版里的加锁检查操作。**
```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {                     // ①
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();     // ②
                }
            }
        }

        return instance;
    }
}
```
这样我们只会在实例化对象时有加速动作，成功实例化对象之后的`getInstance()`调用不会再出现锁的争用和上下文切换开销。

但这里还有一个问题，`instance = new Singleton()`并不是一个原子操作，在 JVM 中它实际上是分成三步进行的：
1. 为 Singleton 对象分配内存
2. 调用构造函数对变量进行必要的初始化
3. 将 instance 引用指向 Signleton 对象

JVM 在执行语句时可能会发生重排序，上面的顺序可能是 1-2-3，也可能是 1-3-2，当线程 A 在执行 语句 ② 时，如果发生了 1-3-2 的重排，另一线程 B 在执行语句 ① 的时候会直接判断`instance`不为`null`，最终导致返回的`instance` 是一个没有经过初始化的对象。

这里的解决方式也很简单，只需要将`instance`变量加上`volatile`关键字即可，它会子语句 3 前后加上内存屏障，防止对`instance`的写操作和之前的任何内存操作进行重排，防止语句 2 3 进行重排，并且确保 3 的执行对其他线程可见。
> volatile 的内存语义参见之前的博文[JMM 小结](https://cui1994.github.io/2019/01/24/jmm/)

```java
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }

        return instance;
    }
}
```

## 在静态变量中创建对象
到目前为止，我么都是自己来控制对象只被创建一次，可以换一个思路，可以**将创建对象的过程写在静态变量里，这样就可以在类初始化的时候创建对象，而 JVM 可以保证类只会被初始化一次，在多线程环境中也会被正确的加载和同步。**
> Java 类加载机制见[JVM基础（二） 类加载过程](https://cui1994.github.io/2019/01/15/classload/)

```java
public class Singleton {
    private final static Singleton instance = new Singleton();
    private Singleton() {}

    public static Singleton getInstance() {
        return instance;
    }
}
```
但这样也有一个问题，对象是在`Singleton`类被初始化时创建的，而不是第一次调用`getInstance()`时被创建的，提前创建对象会造成一定的内存消耗，因此，我们要采用**延时加载策略**，将对象的创建完全控制在`getInstance()`方法内。解决方式是构建一个内部静态类，在`getInstance()`方法内引用这个类的静态变量触发其初始化。
```java
public class Singleton {
    private Singleton() {}

    private static class InstanceHandler {
        final static Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return InstanceHandler.INSTANCE;
    }
}
```

## 枚举单例
最后算是 Java 的一点私货，**只有一个实例的枚举类本身就是一个单例实现**，而且只有当它的实例对象被访问时才会触发对象的创建，其他如`Singleton.class.getName()`的方法不会创建实例（这类反射方法会触发类的初始化）。
```java
public enum Singleton {
    INSTANCE;
    public void otherMethod(){}
}
```


(End)

