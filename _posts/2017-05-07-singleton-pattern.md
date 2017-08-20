---
layout: post
title: 设计模式-单例模式
date: 2017-05-07
categories: blog
tags: [java,设计模式]
description: 区别各种单例模式实现的优点和缺点

---

### 单例模式的使用场景

当你需要使用某个类只有一个实例的时候

ps：对单个方法而言静态方法同样也可以达到这个目的，例如xxUtils里的方法。

### 单例模式和静态方法的区别

静态方法是基于对象的，而单例模式是面向对象的。虽然不面相对象也能解决问题，面相对象的代码提供一个更好的编程思想。

如果一个方法和他所在类的实例对象无关，那么它就应该是静态的（例如xxUtils），反之他就应该是非静态的。
如果我们确实应该使用非静态的方法，但是在创建类时又确实只需要维护一份实例时，就需要用单例模式了（例如xxManager，xxHelper）。

这里通常会有一些理解上的误区：静态方法常驻内存，非静态方法只有使用的时候才分配内存？

这里其实是错误的，静态方法和非静态方法，他们都是在第一次加载后就常驻内存，所以方法本身在内存里，没有什么区别

[具体原理请原文](http://www.cnblogs.com/seesea125/archive/2012/04/05/2433463.html)

### 单例模式常见的几种实现方式和特点

**1.最常用，懒汉式**

如其名，典型的懒加载，但是特点是在多线程中可能会不能正常工作（多个线程几乎同时去调用getInstance（）
都被判断为==null，于是各自new了一个实例，这与单例模式的初衷有冲突了）
```java
public class SingletonClass {

    private static SingletonClass sInstance;
    private SingletonClass () {
      //TODO u can do something u want
    }

    public static SingletonClass getInstance() {
      if (sInstance == null) {
          sInstance = new SingletonClass();
      }
    return sInstance;
    }
}
```

**2.饿汉式**

 通过classloder机制避免了多线程的同步问题，sInstance在类装载时就实例化，
 虽然在单例模式中大多数都是调用getInstance方法， 但是也不能确定有其他的方式（或者其他的静态方法）导致类装载
 从而造成不需要使用sInstance时初始化了sInstance（达不到lazy loading的效果）

 下面是常见导致类加载的原因：
1. 使用new关键字实例化对象

2. 读取一个类的静态字段（被final修饰、已在编译期把结果放在常量池的静态字段除外）

3. 设置一个类的静态字段（被final修饰、已在编译期把结果放在常量池的静态字段除外）

4. 调用一个类的静态方法

```java
public class SingletonClass {

    private static SingletonClass sInstance = new SingletonClass();
    private SingletonClass () {
        //TODO u can do something u want
    }

    public static synchronized SingletonClass getInstance() {
        return sInstance;
    }
}
```

**3. 线程安全的懒汉式**

解决了线程不安全的问题，但是每次使用的时候都会同步，导致效率低，然而真正需要使用同步锁的机会并不多
```java
public class SingletonClass {

    private static SingletonClass sInstance;
    private SingletonClass () {
        //TODO u can do something u want
    }

    public static synchronized SingletonClass getInstance() {
        if (sInstance == null) {
            sInstance = new SingletonClass();
        }
        return sInstance;
    }
}
```

**4.静态内部类保证安全和效率**

这种方式同样利用了classloder的机制来保证初始化instance时只有一个线程,
但是这种方式保证了在没有调用getInstance方法但是导致SingletonClass被装载时不会初始化sInstance，
既保证了线程安全又真正达到了懒加载的效果

ps：加载一个类时，其内部类不会同时被加载。一个类被加载，当且仅当其某个静态成员（静态域、构造器、静态方法等）被调用时发生。
```java
public class SingletonClass {

    private static class SingletonClassHolder {
        private static SingletonClass sInstance = new SingletonClass();
    }

    private SingletonClass () {
        //TODO u can do something u want
    }

    public static SingletonClass getInstance() {
        return SingletonClassHolder.sInstance;
    }
}
```

**5.双重锁模式**

这种方式是懒汉式和饿汉式结合的升级版，初始化sInstance时会有同步锁保护，
而且也不会每次调用getInstance都会有同步锁保护导致效率低下。
```java
public class SingletonClass {

    private volatile static SingletonClass sInstance;
    private SingletonClass () {
        //TODO u can do something u want
    }

    public static SingletonClass getInstance() {
        if (sInstance == null) {
            synchronized (SingletonClass.class) {
                if (sInstance == null) {
                    sInstance = new SingletonClass();
                }
            }
        }
        return sInstance;
    }
}
```
注意：sInstance前面还有一个volatile修饰导致了必须在JDK1.5版本后才能使用。理论上说这个volatile是不能去掉的

因为，sInstance = new SingletonClass()这句，这并非是一个原子操作，事实上在 JVM 中这句话大概做了下面 3 件事情。
1. 给 sInstance 分配内存
1. 调用 SingletonClass 的构造函数来初始化成员变量
1. 将sInstance对象指向分配的内存空间（执行完这步 sInstance 就为非 null 了）

但是在 JVM 的即时编译器中存在指令重排序的优化。
也就是说上面的第二步和第三步的顺序是不能保证的，最终的执行顺序可能是 1-2-3
 也可能是 1-3-2。如果是后者，则在 3 执行完毕、2 未执行之前，被线程二抢占了，
 这时 sInstance 已经是非 null 了（但却没有初始化），
 所以线程二会直接返回 sInstance，然后使用，然后顺理成章地报错。

 有些人认为使用 volatile 的原因是可见性，也就是可以保证线程在本地不会存有 instance 的副本，
 每次都是去主内存中读取。但其实是不对的。
 使用 volatile 的主要原因是其另一个特性：禁止指令重排序优化。
 也就是说，在 volatile 变量的赋值操作后面会有一个内存屏障（生成的汇编代码上），
 读操作不会被重排序到内存屏障之前。比如上面的例子，取操作必须在执行完 1-2-3 之后或者 1-3-2 之后，
 不存在执行到 1-3 然后取到值的情况。
 从「先行发生原则」的角度理解的话，
 就是对于一个 volatile 变量的写操作都先行发生于后面对这个变量的读操作（这里的“后面”是时间上的先后顺序）。

[关于双重锁分析的原文](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)

[volatile的具体作用分析](http://www.cnblogs.com/aigongsi/archive/2012/04/01/2429166.html)

**6.通过枚举实现单例**

用枚举写单例实在太简单了！这也是它最大的优点。
```java
public enum SingletonClass {

    instance;

    public volatile int count = 0;
    SingletonClass () {
        //TODO u can do something u want
    }
}
```
直接通过SingletonClass.instance获取对象。创建枚举默认就是线程安全的，而且还能防止反序列化导致重新创建新的对象。
但是还是很少看到有人这样写，可能是因为不太熟悉吧。
