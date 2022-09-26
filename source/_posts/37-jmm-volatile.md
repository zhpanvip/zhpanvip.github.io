---
layout: article
index_img: https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e2beb686a1244b69911427b8677548e~tplv-k3u1fbpfcp-zoom-crop-mark:3024:3024:3024:1702.awebp
title: 这一次，彻底搞懂Java内存模型与volatile关键字
date: 2021-05-30 00:07:05
categories:
- Java进阶
tags: [多线程,JMM]

---


多线程并发是Java语言中非常重要的一块内容，同时，也是Java基础的一个难点。说它重要是因为多线程是日常开发中频繁用到的知识，说它难是因为多线程并发涉及到的知识点非常之多，想要完全掌握Java的并发相关知识并非易事。也正因此，Java并发成了Java面试中最高频的知识点之一。本系列文章将从Java内存模型、volatile关键字、synchronized关键字、ReetrantLock、Atomic并发类以及线程池等方面来系统的认识Java的并发知识。通过本系列文章的学习你将深入理解volatile关键字的作用，了解到synchronized实现原理、AQS和CLH队列锁，清晰的认识自旋锁、偏向锁、乐观锁、悲观锁...等等一系列让人眼花缭乱的并发知识。

多线程并发系列文章：

[这一次，彻底搞懂Java内存模型与volatile关键字](https://zhpanvip.gitee.io/2021/05/30/37-jmm-volatile/)

[这一次，彻底搞懂Java中的synchronized关键字](https://zhpanvip.gitee.io/2021/06/14/39-synchronized/)

[这一次，彻底搞懂Java中的ReentranLock实现原理](https://zhpanvip.gitee.io/2021/06/19/40-reentranlock/)

[这一次，彻底搞懂Java并发包中的Atomic原子类](https://zhpanvip.gitee.io/2021/06/26/41-atomic-cas/)

[深入理解Java线程的等待与唤醒机制（一）](https://zhpanvip.gitee.io/2021/07/02/42-wait-notify1/)

[深入理解Java线程的等待与唤醒机制（二）](https://zhpanvip.gitee.io/2021/07/03/43-wait-notify2/)

[Java并发系列终结篇：彻底搞懂Java线程池的工作原理](https://zhpanvip.gitee.io/2021/07/10/44-thread-pool/)

[Java并发系列番外篇：ThreadLocal原理其实很简单](https://zhpanvip.gitee.io/2021/07/19/45-ThreadLocal/)

本文是Java并发系列的第一篇文章，将详细的讲解Java内存模型与volatile关键字的作用。

## 一、Java内存模型

提到Java内存模型，很多同学首先想到的是Java的内存区域划分。在这里首先声明本节内容并非讲解Java内存区域。但是，了解Java的内存区域，对于理解Java的内存模型会有一定的帮助。如果你想了解Java内存区域，可以参考我之前写的一篇文章[深入JVM--Java运行时内存区域详解](https://juejin.cn/post/6868340872698658830)

Java内存模型英文为**Java Memory Model**，简称为JMM。JMM本身是一个抽象概念，并非真实存在于Java虚拟机中。它的目的仅仅是定义了程序中各种变量（指实例变量、静态字段和构成数组对象的元素，但不包括局部变量和方法参数，因为局部变量和方法参数是线程私有的，不会被线程共享）的访问规范，即关注在虚拟机中把变量存储到内存和从内存中取出变量值这样的底层细节。而在虚拟机中的运算单元就是线程，因此**可以理解为JMM定义的就是线程访问共享变量的方式。** 当然，这么解释JMM可能仍然很抽象和难以理解。关于JMM我们不妨先放一放，先来了解一下计算机的缓存一致性问题，了解缓存一致性问题将更有利于我们认识Java内存模型。


### 1.缓存一致性
我们知道，一个简单的计算机可以抽象为**CPU**、**内存**以及I/O设备，其中，CPU负责数据的处理与运算，内存则可以理解为存储CPU运算后的数据。CPU在运行时，会首先从内存中取出运算指令，然后解码并确定其类型和操作数，最后执行该指令。在指令执行完毕后，CPU会将计算所得数据写入内存。

然而，在计算机系统中存在一个CPU的运算速度与内存读写速度不匹配的问题，即CPU的运算速度远比内存的读写速度快。由于读写速度缓慢，严重拖累了计算机的运行效率。为了解决这一问题，现代计算机系统在CPU与内存之间加入了一层或多层**高速缓存**，而高速缓存的读写速度与CPU的运算速度几乎相当。在加入高速缓存后，CPU在执行指令前，需要先将要运算的数据从内存读取（即复制）到高速缓存中，接着CPU对数据进行处理，然后再将运算后的数据写入到高速缓存，最后再从缓存同步回内存中。

可以看到，基于高速缓存的存储交互很好的解决了单核CPU与内存读写速度之间的矛盾。但在多核心CPU的计算机中却引来了新的问题。由于人们对计算机性能的追求，单核CPU已经很难维持“摩尔定律”。目前市面上绝大部分都是多核CPU的计算机。在多核CPU的系统中，每个处理器都有自己的高速缓存，而它们又共享同一个主内存，如下图。当多个处理器的运算任务都涉及到同一块主内存区域时，将可能导致各自缓存数据不一致的问题。例如，处理器1与处理器2都从主内存读取了同一个数据分别存储到自己的高速缓存区域，然后，两个处理器都对这一数据进行了修改。那么再同步回主内存的时候应该以哪条数据为准呢？这一问题就是**缓存一致性问题**

为了解决缓存一致性问题，设计者们为CPU制定了一个读写协议，并要求各个CPU在读写缓存时都要遵循这一协议。这类协议有MSI、MESI、MOSI等，被称为**缓存一致性协议**。只要CPU的读写遵循了缓存一致性协议就能很好的解决缓存一致性问题了。关于协议的具体实现，不是本篇文章的内容，这里不再赘述。

![0AC13792-79A8-420E-AD8D-F447E9AEC514.png](https://img-blog.csdnimg.cn/img_convert/9d051f7b3cf711b9bb397341c71fcb9c.png)

### 2.Java内存模型（JMM）

在了解了缓存一致性问题后，我们继续回到jMM。在本章开篇，我们为JMM下了一个比较抽象的定义。并且提到JMM可以简单的理解为线程访问共享变量的方式。可见JMM是Java并发编程的底层基础，想要深入了解并发编程，就需要先理解JMM。那么本节内容，我们就来具体的谈一谈JMM。

**JMM规定所有变量都存储在主内存中，每条线程还有自己的工作内存。线程的工作内存中保存了被线程使用的变量的主内存副本，线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的数据。不同线程之间也无法直接访问对方的工作内存中的变量，线程间变量值的传递需要通过主内存来完成。** 也就是说Java线程之间的通信采用的是共享内存。

看到这里是不是觉得似曾相识？没错，这里其实跟上一节**缓存一致性**中讲到的多核CPU共享主内存是类似的。只不过在虚拟机中不是CPU，而是线程。每条线程都有自己的工作空间，而共享变量存储在共享内存中。线程在运行时会首先将共享内存中的数据读取到自己的工作内存，即在线程的工作内存中复制了一个共享变量的副本，然后对其进行计算，计算完成后线程会将自己工作内存中的这个共享变量副本同步回主内存。线程、工作内存、与主内存的关系如下图所示：
![FFC84BBD-0C24-42FE-A38C-99A40F967F72.png](https://img-blog.csdnimg.cn/img_convert/5e6586e16baa9dee5ff6105689781042.png)

到这里，大家对于Java的内存模型应该都有了一个深入的了解。但是，很多同学还是会疑惑Java内存模型和Java的内存区域到底有什么关系？在[深入JVM--Java运行时内存区域详解](https://juejin.cn/post/6868340872698658830)这一篇文章中有讲到虚拟机的内存被分为了程序计数器、Java堆、方法区以及虚拟机栈等几个内存区域，而在这几个内存区域中，虚拟机栈是线程所独有的。到这里，我想有些读者心里应该已经有了答案。其实，Java内存模型与Java内存区域并不是同一个层次对内存的划分，可以说两者并没有什么关系。但是它们之间也存在着比较明显的对应关系，即主内存对应Java堆中的实例数据部分，而工作内存则对应虚拟机栈中的一些区域。

到这里我们已经理清楚了Java内存模型的概念，但是似乎还少了点东西。我们知道多核CPU的计算机的存在缓存一致性问题。那么对于Java内存模型来说，多个线程之间是不是也会存在一样的类似问题呢？那对于这一问题，Java又是怎么解决的呢？

## 二、volatile关键字
上一章中我们认识了Java内存模型，并且提出了Java内存模型中也会存在缓存一致性问题。而解决Java内存模型的缓存一致性问题靠的就是本章的主角--volatitle关键字。volatitle关键字是面试中的常客，虽然它的使用却很简单，但真正理解volatile关键字的人并不多。因为要理解volatile关键字，首先要搞懂Java的内存模型。本章内容，就在上章内容的基础上来认识volatile关键字。

在认识volatile之前，我们先来了解一下Java并发编程的三大性质即：是原子性、可见性以及有序性。

- 原子性 原子在化学中反应上是不可在分割的粒子。因此原子性指的是一个不可以被分割的操作，即这个操作在执行过程中不能被中断，要么全部不执行，要么全部执行。且一旦开始执行，不会被其他线程打断。
- 可见性 指的是一个线程修改了共享变量后，另外线程能立即感知这个变量被修改。
- 有序性 指程序按照代码的先后顺序执行。有时候为了优化性能，编译器会对字节码指令进行重排序。但是能保证重排序后的执行结果与重排序之前是一致的。

volatitle经常被用到并发编程的场景中。它的作用有两个，即：
- 保证可见性；
- 保证有序性。

但是，要注意volatile关键字并不能保证原子性。接下来我们对volatile的两个作用进行详细分析。



### 1.volatile保证可见性

在第一章中我们已经知道，由于每个线程都有自己的工作空间，导致多线程的场景下会出现缓存不一致性的问题。即，当两个线程共用一个共享变量时，如果其中一个线程修改了这个共享变量的值。但是由于另外一个线程在自己的工作内存中已经保留了一份该共享变量的副本，因此它无法感知该变量的值已经被修改。

看下面的一个例子：

```java

public class VolatileDemo {

    private static boolean ready;

    public static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("MyThread is running...");
            while (!ready) ; // 如果ready为false，则死循环
            System.out.println("MyThread is end");
        }
    }


    public static void main(String[] args) throws InterruptedException {
        new MyThread().start();
        Thread.sleep(1000);
        ready = true;
        System.out.println("ready = " + ready);
        Thread.sleep(5000);
        System.out.println("main thread is end.");
    }

}
```
代码中定义了一个boolean类型的成员变量ready,其默认值为false。在MyThread线程中判断如果ready为false时则进行死循环。接下来在main方法中开启MyThread线程，并在睡眠1s后将ready修改为true。正常情况下ready修改为true后MyThread线程中的死循环则会停止，并打印“MyThread is end"。但是来看下运行效果跟我们猜想是否一致，打印日志如下：

```
MyThread is running...
ready = true
main thread is end.
```
可以看见当ready被修改为true后，MyThread线程依然未结束。通过这一例子也证实了MyThread线程中的ready副本并没有得到及时的更新。

那么接下来我们将成员变量ready使用volatile关键字修饰后，再运行看打印日志：


```
MyThread is running...
MyThread is end
ready = true
main thread is end.
```

可见，当在主线程中修改了ready为true后，MyThread线程立即感知了ready的变化，并结束了死循环。从这个例子中也可以看见volatile确实能有效的保证多个线程共享变量的可见性。
### 2.volatile保证有序性
我们知道，编译器为了优化程序性能，可能会在编译时对字节码指令进行重排序。重排序后的指令在单线程中运行时没有问题的，但是如果在多线程中，重排序后的代码则可能会出现问题。因此，一般在多线程并发情况下我们都应该禁止指令重排序的优化。而volatile关键字就可以禁止编译器对字节码进行重排序。volatile保证有序性在我们平时开发中有一个很常见的例子，即双重锁校验的单利模式下需要使用volatile关键字来禁止指令重排序。我们来看下代码：


```Java
public class DoubleCheckLock {

    private volatile static DoubleCheckLock instance;

    private DoubleCheckLock(){}

    public static DoubleCheckLock getInstance(){

        //第一次检测
        if (instance==null){
            //同步
            synchronized (DoubleCheckLock.class){
                if (instance == null){
                    //多线程环境下可能会出现问题的地方
                    instance = new DoubleCheckLock();
                }
            }
        }
        return instance;
    }
}

```
如果上述代码中没有给instance加上volatile关键字会怎么呢？我们不妨来分析一下，首先我们应该清楚instance = new DoubleCheckLock();这一操作并不是一个原子操作，实例化对象的字节指令可以分为三步，如下：
- 1.分配对象内存：memory = allocate();
- 2.初始化对象：instance(memory);
- 3.instance指向刚分配的内存地址：instance = memory;

而由于编译器的指令重排序，以上指令可能会出现以下顺序：

- 1.分配对象内存：memory = allocate();
- 2.instance指向刚分配的内存地址：instance = memory;
- 3.初始化对象：instance(memory);

以优化后的字节码指令来看双重锁校验的代码是否有问题呢？不难发现，如果线程1第一次调用单利方法，在该线程的时间片轮转结束后执行到了优化后的第二个指令，即instance被赋值，但是还未被分配初始化对象。此时，线程2抢到了CPU时间片，同时调用了getInstance方法，第一次校验就发现instance不为null，遂将其返回。在得到这个单利后调用单利的方法，此时必定出现空指针异常。

因此，可见指令重排序在多线程并发的情况下是会出现问题的。此时，我们便可以通过volatile关键字来禁止编译器的优化，从而避免空指针的出现。

### 3.volatile不能保证原子性
对于原子操作，volatile关键字是无能为力的。如果需要保证原子操作，则需要使用synchronized关键字、Lock锁
或者Autom相关类来确保操作的原子性。关于这些内容本篇文章不再赘述，将会在后续文章中详细分析。


## 参考&推荐阅读

《深入理解Java虚拟机》第3版，作者周志明

《现代操作系统》作者Andrews.Tanenbaum/Herber Bos

[全面理解Java内存模型(JMM)及volatile关键字](https://blog.csdn.net/javazejian/article/details/72772461)

[Java内存模型](https://blog.csdn.net/eff666/article/details/66479636)


