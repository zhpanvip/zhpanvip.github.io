---
layout: article
index_img: https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c9a5f1afaea45f48f3bfa1e97163fc6~tplv-k3u1fbpfcp-zoom-crop-mark:3024:3024:3024:1702.awebp?
title: 反思 Android 消息机制设计与实现
date: 2022-06-17 20:18:46
categories:
- Framework
tags: [Handler]
---


[上篇文章](https://juejin.cn/post/6994057245113729038)介绍了 Android 中的 Binder 机制。Binder 在 Android 系统中占有着举足轻重的地位，它是 Android 系统中跨进程通信最重要的方式。而另外一个重要的且能与Binder相提并论的角色便是本文要分析的 Handler。Binder 支撑起了 Android 系统进程间的通信，而 Handler 支撑起的则是进程内线程间的通信。同时，Android 应用程序的运行皆依靠 Handler 的消息机制驱动，这其中就包括触摸事件的分发、View的绘制流程、屏幕的刷新机制以及Activity的生命周期等等。

关于 Handler 其实早在几年前笔者就写过一篇[《追根溯源—— 探究Handler的实现原理》](https://blog.csdn.net/qq_20521573/article/details/77919141?spm=1001.2014.3001.5502)的文章。 但是鉴于当时对于 Handler 的理解并不那么深刻，所以这篇文章的内容与网上大多数写 Handler 的文章一样仅仅是源码分析，对 Android 消息机制没有一个深刻的理解和认识。当然，并不是说这样的文章不好，对于初学者来说更适合阅读这样的文章的。所以，如果你对于 Handler 还没有太熟悉的话，不妨先读一读。

如今，作为一个已有多年 Android 开发经验的从业者，在阅读了大量的 framework 源码之后，对于 Android 的消息机制有了一些更加深刻的认识，这是要写这篇文章的原因。


## 一、从“生产者-消费者”模型说起

关注笔者比较久的同学可能看过我之前写过的一篇文章 [《深入理解Java线程的等待与唤醒机制》](https://juejin.cn/post/6980002998361522190)。在这篇文章中为了分析 synchronized 锁的等待与唤醒机制，举了一个 “生产者-消费者” 问题的例子。

> “生产者-消费者” 问题又称有限缓冲问题（Bounded-buffer problem），是一个多线程同步问题的经典案例。该问题描述了共享固定大小缓冲区的两个线程——即所谓的“生产者”和“消费者”——在实际运行时会发生的问题。生产者的主要作用是生成一定量的数据放到缓冲区中，然后重复此过程。与此同时，消费者会在缓冲区消耗这些数据。该问题的关键就是要保证生产者不会在缓冲区满时加入数据，消费者也不会在缓冲区中空时消耗数据。
”生产者-消费者“模型图如下：

![生产者-消费者](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08aa1fce00cb43fb830907d484e6911e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

可以看得出来，图中的 "生产者"和"消费者"处于两个不同的线程，但是他们共用了同一个队列。生产者在完成数据的生产后会通过 notify 方法唤醒消费者线程，当队列满的时候，生产者线程会调用 wait 方法来阻塞自身。同时，消费者线程在被唤醒后则会从队列中取出数据，并通过 notify 方法唤醒生产者线程继续生产数据。当队列中的数据被取空的时候，消费者线程同样会调用 wait 方法阻塞自身。

我们仍然拿生产面包的例子来看。首先需要有一个存放面包的容器（缓冲队列），可以将其命名为BreadContainer，提供放入面包和取出面包的方法。代码如下：

```java
public class BreadContainer {

    LinkedList<Bread> list = new LinkedList<>();
    // 容器容量
    private final static int CAPACITY = 10;

    /**
     * 放入面包
     */
    public synchronized void put(Bread bread) {
        while (list.size() == CAPACITY) {
            try {
                // 如果容器已满，则阻塞生产者线程
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        list.add(bread);
        // 面包生产成功后通知消费者线程
        notify();
        System.out.println(Thread.currentThread().getName() + " product a bread" + bread.toString() + " size = " + list.size());
    }

    /**
     * 取出面包
     */
    public synchronized Bread take() {
        while (list.isEmpty()) {
            try {
                // 如果容器为空，则阻塞消费者线程
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        Bread bread = list.removeFirst();
        // 消费后通知生产者生产面包
        notify();
        System.out.println("Consumer " + Thread.currentThread().getName() + " consume a bread" + bread.toString() + " size = " + list.size());
        return bread;
    }
}
```

接下来实现生产者，让生产者来生产面包，并提供面包容器。

来生产面包，并提供面包容器。

```
// 生产者
public class Producer {

    private final BreadContainer container;

    public Producer() {
        container = new BreadContainer();
    }

    public BreadContainer getContainer() {
        return container;
    }

    public void makeBread() {
        // 生产者生产面包
        container.put(new Bread());
    }
}
```

实现消费者则负责从面包容器中取出面包进行消费

```
// 消费者
public class Consumer {
    BreadContainer container;

    public Consumer(BreadContainer container){
        this.container = container;
    }

    public void takeBread() {
        for (; ; ) {
            Bread bread = container.take();
            bread.eat();
        }
    }
}
```

完成生产者与消费者两个角色后，我们可以一个测试代码，如下:

```
public static void main(String[] args) {

    // 实例化生产者
    Producer producer = new Producer();
    // 实例化消费者
    Consumer consumer = new Consumer(producer.getContainer());
    // 开启生产者线程
    new Thread(() -> {
        for (int i = 0; i < 100000; i++) {
            producer.makeBread();
        }
    }).start();

    // 消费者在主线程消费   
    consumer.takeBread();
}
```
看到上边这段代码是否有种似曾相识的感觉？如果没有那么不妨去看看Android源码中ActivityThread 的main方法的实现，你一定会恍然大悟！

”生产者-消费者“模型的案例在平时的开发中是很常见的。例如 Rxjava 的流控制就是典型的”生产者-消费者“模型，除此之外还有线程池的内部实现，以及 Android 系统中输入事件的采集与派发都是基于”生产者-消费者“模型设计的。

上文提到”生产者-消费者“模型解决的是多个线程共享内存的有限缓冲问题。但其实它还解决了另外一个重要问题，即实现了线程间的通信。在生产与消费的过程中由于共用了同一个缓冲队列，”生产者“产生的数据从生产者线程传递给了消费者线程，对缓冲队列内的数据而言就实现了线程的切换。

了解了“生产者-消费者”模型之后对于 Android 理解消息机制的设计思想会有很大的帮助。



## 二、设计消息机制的框架

现在让我们回到 Android 来想一下 Android 中的场景。在文章开头已经提到 Android 应用中触摸事件的分发、View的绘制、屏幕的刷新以及 Activity 的生命周期等都是基于消息实现的。这意味着在 Android 应用中随时都在产生大量的 Message，同时也有大量的 Message 被消费掉。

另外我们都知道在 Android 系统中，UI更新只能在主线程中进行。因此，为了更流畅的页面渲染，所有的耗时操作包括网络请求、文件读写、资源文件的解析等都应该放到子线程中。在这一场景下，线程间的通信就显得尤为重要。因为我们需要在网络请求、文件读写或者资源解析后将得到的数据交给主线程去进行页面渲染。

那在这样的背景下如果让我们作为 Android 系统的设计者，会如何设计并实现 Android 的消息机制，让其即满足具有缓冲功能又能实现线程切换的能力呢？


有了第一章的知识后我想你一定会很自然的想到使用”生产者-消费者“模型来实现。因为这一模型既能解决数据缓冲问题，又实现了数据在线程间的切换。

没错，Android系统的设计者也是这么想的，于是便诞生 Handler 这一杰作！接下来让我们跟随消息机制设计者们的思维来看一下如何实现这一功能。


### 1.设计消息缓冲区--MessageQueue

由于系统中无时无刻都在产生消息，因此我们首先需要有一个消息缓冲区，用来存放各个生产者线程所产生的消息，我们将这个缓冲区命名为 MessageQueue。作为一个消息队列，其内部需要有一个存放消息的容器。同时需要对外提供插入消息和取出消息的接口，将这两个接口方法分别命名为 `enqueueMessage(Message msg)` 与 `next()`。MessageQueue 伪代码实现如下：


```java
public class MessageQueue {
    // 消息容器，暂且使用 LinkedList。
    private LinkedList<Message> list = new LinkedList<>();

    public synchronized void enqueueMessage(Message msg) {
        list.add(msg);
    }

    public synchronized Message next() {
    
         while (list.isEmpty()) {
            try {
                // 如果容器为空，则阻塞消费者线程
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
       
        return list.removeFirst()  
     }
  }
}

```

在 MessageQueue 中维护了一个集合，当插入消息时将消息存入这个集合，取出消息时，将消息从集合中移除并返回。

有了消息队列之后，还需要有”生产者“与”消费者“这两个角色，生产者负责向 MessageQueue 中插入消息，而消费者负责从 MessageQueue 中取出消息并进行消费。


### 2.设计消息机制的”生产者“--Handler

系统各处产生的消息需要存入到消息队列中，等待消费者取出消息并将其消费。因此我们可以设计一个包装类来实现消息插入到消息队列，将这个包装类命名为 Handler。既然作为消息队列的包装， Handler 肯定要持有 MessageQueue，以便于向消息队列中插入消息。同时还需要对外提供插入消息的API，可以将这个插入消息的API命名为 `sendMessage(Message msg)`,在这个方法内实现把 Message 插入到 MessageQueue 的逻辑。

另外，作为Handler的使用方，在通过 Handler 将消息插入到 MessageQueue 后，肯定迫切的需要知道消息何时被消费者处理。因此，还需要在 Handler 中添加一个处理消息的回调方法，以便于使用方重写该方法，并完成需要的逻辑。我们将这个方法命名为`dispatchMessage`。至此，”生产者“的设计就完成了。Handler 的伪代码实现如下：


```java
public class Handler {

   private MessageQueue mQueue;
   
   // 这里的 MessageQueue 应当与消费者的 MessageQueue 是同一个
   public Handler(MessageQueue queue){
       mQueue = queue;
   }
   
   // 处理消息的回调
   public void dispatchMessage(Message msg){
       
   }
   
   // 向消息队列中插入消息
   public void sendMessage(Message msg) {
       // Message 中需要持有Handler，以便回调 dispatchMessage 方法。
       msg.target = this;
       mQueue.enqueueMessage(msg);
   }
   
   // ... 另外还可以实现多个 sendMessage 的重载方法，以适用不同的需求。
}
```

### 3.设计消息机制的信使-- Message

Message 应该充当的是信使的作用，即 Message 需要携带调用方赋予的数据。而这一数据类型并不确定，因此我们可以将它声明为 Object 类型。另外，在消息被消费者处理的时候需要通知调用方消息被处理了。因此可以让 Message 持有一个 Handler,以便在消息被处理后回调给Handler。我们将这个 Handler 命名为 target。于是 Message 的伪代码就可以有如下实现：

```java
public class Message{
    // 携带的消息
    Object obj;
    // 持有 Handler
    Handler target;
    // ...
}
```

### 4.设计消息机制的”消费者“--Looper

在完成了消息队列、生产者、以及消息信使的设计之后，我们还需要实现消费者这一角色。作为消费者，它的职责就是从 MessageQueue 中取出 Message ，并将其消费。值得注意的是这个 MessageQueue 必须是与生产者所共用的。这里我们将”消费者“这一角色命名为 Looper。Looper作为”消费者“，其职责就是需要不断的从 MessageQueue 中取出消息并进行消费。那么我们就将这个取消息的方法命名为 loop(),因为需要不断的从 MessageQueue 中取出消息，所以这个方法应该被设计成一个死循环，没有消息的时候就阻塞执行。因此 Looper 的伪代码实现如下:


```java
public class Looper {

    // 在 Looper 中实例化消息队列，并提供给Handler，实现生产者与消费者共享
    MessageQueue mQueu = new MessageQueue();
    
    // 从消息队列中取出消息并消费
    public static void loop(){
        for(;;) {
            // 静态方法，需要获取Looper的实例
            Message msg = myLooper().mQueue.next();
            if(msg == null) {
                return;
            }
            // 回调到 Handler
            msg.target.dispatchMessage(mgs);
        }
    }
    
    // 这里假设通过myLooper方法拿到looper的实例
    public Looper myLooper(){
        return new Looper();
    }
}
```

通过”生产者-消费者“模型，我们可以很轻松的写出 Android 消息机制的大体框架.而接下来我们要思考的是在这个实现过程中会面临什么样的问题，以及该如何去解决。

## 三、完善消息机制的实现逻辑

上一章中，我们搭建起了消息机制的大体框架，下一步就是要实现具体的逻辑了。仔细想一想会发现我们面临着不少的问题，我列举出了以下几个，不妨来思考思考。

- 线程切换是消息机制的一个重要功能，应该如何实现？
- 一个线程可以有多个 Looper 实例吗？如果不可以，那应该如何保证线程级别的 Looper 单例？
- APP 在运行时会产生大量的 Message，每次都通过 "new" 关键字实例化 Message 可行吗 ？
- 系统发送的某些消息具有较高的优先级，如何才能保证其优先执行？

### 1.实现线程切换

消息机制一个很重要的需求，即在子线程中获取到的数据需要发送给主线程进行页面渲染。这个让数据从子线程切换到主线程的功能该如何实现呢？

这其实是一个很简单的问题，只需要在主线程中实例化 Looper，同时在实例化 Handler 的时候在 Handler 构造方法中传入 Looper 持有的 MessageQueue 即可。这样 Looper 和 Handler 共享了同一个 MessageQueue。不管 Handler 在哪个线程发送消息，最终 Looper 都会在主线程中取出消息并执行。看一下代码的实现：

```java
public class MyActivityThread {
    public void main(String[] args){
    
        // 实例化一个 Looper
        Looper looper = Looper.myLooper();
        
        // 实例化 Handler,并传入 Looper 持有的 MessageQueue
        Handler handler = new Handler(looper.mQueue) {
            @Override
            public void dispatchMessage(Message msg){
                // 在主线程中得到了 Message
            }
        };
        
        // 开启一个子线程
        new Thread() {
            @Override
            public void run(){
                // 在子线程中通过 Handler 发出一个消息
                handler.sendMessage(new Message());
            }
        
        }
        
        // 调用loop,开启循环不断的从MessageQueue中取出消息，没有消息就会被阻塞。
        // 通过这样的方式，导致 main 方法不会被执行完而退出程序，Android 系统源码也是
        // 这样实现的，这也是为什么 Android 中的 APP 不会像java程序一样，执行完逻辑就结束掉。
        looper.loop();
    }
}
```

当然，这里举的是子线程与主线程的例子，对于子线程与子线程的切换是与之类似的。

### 2.实现线程级别的 Looper 单例

先来看这个问题，一个线程可以有多个 Looper 实例吗？一个线程中有多个 Looper 站在逻辑的角度来看显然是没什么问题的。但是如果站在 Android 系统的角度来考虑，一个线程有多个Looper实例显然有很大的问题！因为 Looper 的 loop() 方法是一个阻塞方法。如果在一个线程中实例化了多个 Looper，并且都调用了它的 loop 方法，那么一定只有第一个调用 loop 方法的 Looper 实例会运行，其他的 Looper 会被阻塞永远也执行不了。

因此，作为消息机制的设计者，我们应该保证单个线程只能实例化一个 Looper。而不能寄托于使用者，要求他们在使用 Looper 的时候只实例化一次。

那么这个时候再来回顾一下上一章我们对 Looper 的设计，似乎是有很大缺陷的。因为此时的Looper可以在主线程中通过 myLooper 方法实例化出任意多个 Looper 对象。显然这是不符合我们的需求的。有的同学说可以将 Looper 设计成单例，这样就不会被实例化出多个 Looper 了。但这样显然也是不符合需求的，我们需要的是在同一个线程里边只能有一个 Looper 实例，但多个线程可以有多个 Looper 实例。也就是说这个 Looper 应该是一个线程级别的单例。那应该怎么实现呢？

说到这里，java基础掌握比较好的同学应该已经想到了，可以使用 ThreadLocal来实现！

ThreadLocal提供了线程级别的数据存储能力。即在A线程中使用 ThreadLocal 存储了一个数据，那么这个数据只对A线程可见，只有在A线程中才能取出，其他线程无法取到。关于 ThreadLocal 了解这么多就足够了，这里不再赘述 ThreadLocal 的实现原理，如果你对 ThreadLocal 比较感兴趣可以参考我之前写的一篇文章[《Java并发系列番外篇：ThreadLocal原理其实很简单》](https://juejin.cn/post/6986301941269659656)

有了以上理论的支持，我们就可以重构 Looper 的实现逻辑了。修改后的 Looper 代码如下：

```java
public class Looper {

    // 使用 ThreadLocal 存储Looper
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
    MessageQueue mQueu;
    
    // 私有化构造方法，避免单个线程中被多次实例化
    private Looper() {
        // 实例化 MessageQueue
        mQueue = new MessageQueue();
    }
    
    // 添加一个 prepare 方法来实例化 Looper,并将其存储到 TheadLocal
    private static void prepare() {
        // 需要保证Looper是线程唯一的
        if (sThreadLocal.get() != null) {
            // 走到这里说明该线程已经实例化过 Looper 了，抛出异常终止程序执行
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // 将 Looper 实例存入到 ThreadLocal
        sThreadLocal.set(new Looper());
    }
    
    // 从 ThreadLocal 中取出 Looper 实例
    public static Looper myLooper() {
        return sThreadLocal.get();
    }

    // 从消息队列中取出消息并消费
    public static void loop(){
        for(;;) {
            Message msg = myLooper().mQueue.next();
            if(msg == null) {
                return;
            }
            msg.target.dispatchMessage(mgs);
        }
    }
}
```

通过以上代码，我们实现了一个线程级别的单例，保证了每个线程只能创建一个 Looper，多次创建就会导致程序崩溃。

### 3.Message 对象池

Android APP 在运行的时候会有大量的 Message 由系统插入到 MessageQueue 中，前面已经提到过的 View 的绘制过程、事件分发过程、Activity 启动过程等等都会向 MessageQueue 插入消息。这就意味着会有大量的 Message 被实例化。如果每次用到 Message 的时候都通过 "new" 关键字来实例化实现 Message 对象，那么肯定会导致严重的内存抖动问题。

因此，为了避免 Message 的频繁实例化，我们可以对获取 Message 对象的过程做一些优化。通常避免频繁的创建对象的解决方案都是使用对象池。也就是维护一个 Message 对象池，在用完之将后将 Message 的数据进行重置，并将其放入到对象池中，等待下次复用。这样就避免了频繁实例化 Message 可能导致的内存抖动问题。主要是了解一下池化思想，这里就直接 copy 系统的源码了，系统源码中 Message 是一个拥有链表结构的类。

```java
public final class Message implements Parcelable {
    
    public Object obj;
    
    Message next;
    
    // 对象池中有空闲对象直接使用，没有则实例化Message
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    // 回收 Message，并加入到对象池
    public void recycle() {
        if (isInUse()) {
            if (gCheckRecycle) {
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }

    public Message() {
    }
   
}
```

可以看到 Message 实现对象池的两个核心方法就是 obtain() 与 recycle()，obtain负责从对象池中取出 Message，如果对象池没有空闲的 Message，则直接实例化 Message，而 recycle() 则是回收用完的Message，并将其插入到复用链表中。这里所谓的对象池其实就是一个空闲的 Message 链表。

### 4. 无界队列 MessageQueue

在上一小节中我们已经知道 Message 其实是一个拥有链表结构的类。因此 MessageQueue 中的容器其实并非像第二章第1小节中写的那样是一个 LinkedList，而是一个 Message 链表。

通常来说”生产者-消费者“模型中的缓冲队列是有特定的容量的，在缓冲队列填满的时候就会阻塞生产者继续添加数据,即出现所谓的**背压**问题。因此，一个标准的”生产者-消费者“模型必然要考虑**背压策略**，就比如大家所熟知的 RxJava 由于内部使用的是有界队列，因此当队列的容量不足时就会抛出 `MissingBackpressureException`。而 Rxjava 也给出了多个背压策略，例如丢弃事件、扩容、或者直接抛出异常。与之类似的是线程池的实现，区别是线程池内不叫背压策略，而是叫**拒绝策略**。

但是作为接收系统消息的 MessageQueue 如果被设计成有界队列合适吗？显然是不合适的，因为系统发送的消息多是一些中要的消息，任何事件的丢失都可能会导致严重的系统 bug。所以作为消息机制设计者一定会把 MessageQueue 设计成一个无界队列。这样插入消息永远不会被阻塞，也不用考虑所谓的背压策略了。这是消息机制与标准的 ”生产者-消费者“ 模型的区别之一。

关于 MessageQueue 插入消息与取出消息的实现，前面我们只是简单写了伪代码，而且是使用集合实现的。由于我们已经知道系统源码中的 Message 是一个链表结构的类。因此，我们可以使用链表的结构来实现插消息和取消息。

因为这两个方法的实现还是比较复杂的，因此现在我们跳出设计者的身份，跟随 Android 系统源码来解读这两个方法的实现。

#### （1）插入消息的实现

MessageQueue 插入消息的逻辑是在 enqueueMessage 方法中实现的， 简化后的源码如下：

```java
boolean enqueueMessage(Message msg, long when) {

    synchronized (this) {

        msg.when = when;
        
        Message p = mMessages;
        boolean needWake;
       
        if (p == null || when == 0 || when < p.when) {
            // 如果队列为空，或者该消息不是延迟消息，或者是延迟消息
            // 但执行的时间比头消息早，则将消息插入到队列的头部
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 这种情况下说明要插入的消息时延迟消息，遍历链表找到合适的插入位置
        
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                // 如果已经遍历到队列尾部了或者在队列中找到了比要插入的消
                // 息延迟时间更长的消息则终止循环，即找到了合适的插入位置
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            // 将消息插入到这个合适的位置
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            // 唤醒线程
            nativeWake(mPtr);
        }
    }
    return true;
}
```
enqueueMessage 的逻辑其实并不难理解，就是把 Message 插入到链表中去，同时插入链表的位置是根据消息 delay 的时间决定的，delay 的时间越长，插入队列时就越靠后，即越晚执行。最后根据是否要唤醒线程来调用 nativeWake， 这个方法是在 native 层实现的。可以把他理解为"生产者-消费者"模型中 通过 notify 唤醒线程的操作。

#### （2）取出消息的实现

取出消息是通过 MessageQueue 的 next 方法实现的，简化后的 next 方法的源码如下：

```java
Message next() {

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {

        // 阻塞线程
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
        
            // ... 省略异步消息的处理
            
            if (msg != null) {
              
                if (now < msg.when) { // 根据delay的时间判断该Message是否可以执行
                    // 未到执行时间则走到下一次循环调用nativePollOnce阻塞该方法
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 从链表取出消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    msg.markInUse();
                    // 将消息返回
                    return msg;
                }
            } else {
                // 消息队列为空，阻塞时间设置于为-1，表示一直阻塞
                nextPollTimeoutMillis = -1;
            }
        }
        
        // 省略 IdelHandler 的处理逻辑
    }
}
```
这里暂时忽略 next 方法的异步消息处理逻辑以及 IdleHandler 的处理逻辑。简化后的代码并也容易理解。首先是一个用 for 实现的死循环，循环中先调用 nativePollOnce 对线程进行阻塞，这个方法也是一个 native 方法，可以将它理解为”生产者-消费者“模型中阻塞线程的 wait 方法。这个方法的第二个参数表示阻塞的时间，如果是正数，则表示阻塞这个值的毫秒时长，如果是0表示不阻塞，如果小于0，则会一直阻塞。

接下来的逻辑则是判断消息是不是到了该执行的时间了，如果没到，则继续 for 循环执行 nativePollOnce 来阻塞方法，如果消息到了执行的时间就将消息从链表中取出并返回给 Looper，交给 Looper 对消息进行消费。

### 5.Message 的优先级

在标准的”生产者-消费者“模型中消息是没有优先级之分的。即按照标准的队列执行先进先出的逻辑。但是在 Android 消息机制中是需要对消息进行优先级划分的，普通消息应该将优先执行的权利让给那些会影响程序性能的消息，比如 View 绘制的消息、屏幕刷新的消息以及Activity启动的消息等。这是消息机制与标准的”生产者-消费者“模型的又一重要区别。

就我的理解而言，消息机制中的消息一共被划分了四个优先级。优先级由高到低分别是**异步消息**、**普通消息**、**IDleHandler** 以及**延迟消息**。

#### （1）异步消息

在 Message 的源码中为开发者提供了一个 setAsynchronous 的方法，这个方法是对外开放的。通过这个方法会为消息设置一个异步标记。使用代码如下：

```java
Message message = Message.obtain();
// 设置异步消息
message.setAsynchronous(true);
```
异步消息拥有最高的执行优先级，但是仅仅将其设置为异步消息并没有什么作用。它需要配合同步屏障消息来执行。什么是同步屏障消息呢？其实就是一个 Message.target 为 null 的消息。它的实现逻辑在 MessageQueue 的 postSyncBarrier 方法中。源码如下：

```java
// MessageQueue

@UnsupportedAppUsage // 不支持APP调用
@TestApi
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```
可以看到 postSyncBarrier 方法被 `@UnsupportedAppUsage` 注解所修饰，意味着这个方法对开发者是不可见的。而 postSyncBarrier 中的逻辑其实就是向链表的头部插入了一条 Message。而这个 Message 与普通消息不同的是，它的 target 并没有被赋值。在上一节中分析 MessageQueue 的 next 方法时我们忽略了异步消息的处理逻辑。现在来具体看一下这段代码的实现：


```java
Message next() {

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {

        // 阻塞线程
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // 消息队列头
            Message msg = mMessages;
            // 如果 msg 不为 null,并且 msg.target 为 null，则执行if中的逻辑
            if (msg != null && msg.target == null) {
                // 走到这里说明读取到了同步屏障消息
                // 通过 do...while 循环遍历message链表，找到异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
        
            // ... 省略取消息逻辑
        }
        
        // 省略 IdelHandler 的处理逻辑

    }
}
```
可以看到这里首先判断如果 msg 不为 null,并且 msg.target 为 null，则执行if中的逻辑，而if中则通过 do...while 循环来遍历 message 链表，直到找到异步消息才会终止do...while。然后取出异步消息执行下边的消息处理逻辑。不难看出，当遇到同步屏障消息之后就会阻塞普通消息的执行。然后遍历 Message 链表找到被标记为异步的消息优先执行。

但是有个问题，同步屏障消息是在什么时候被插件 MessageQueue 的呢？答案是在ViewRootImpl 中当计划开始遍历 View 树的时候。看下 ViewRootImpl 的 requestLayout 方法，其源码如下：


```java
// ViewRootImpl.java

@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        // 检查是否是在主线程中，如果不是主线程则直接抛出异常
        checkThread();
        // mLayoutRequested标记设置为true，在同一个Vsync周期内，执行多次requestLayout的流程
        mLayoutRequested = true;
        // 计划遍历View树
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {
        // 保证一次 requestLayout 只执行一次View树的遍历
        mTraversalScheduled = true;
        // 通过Handler发送同步屏障阻塞同步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        // 通过Choreographer发出一个mTraversalRunnable，会在这里执行
        mChoreographer.postCallback(
        Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        // ...
    }
}
```
可以看到在调用 ViewRootImpl 的 requestLayout 方法后，会执行scheduleTraversals方法，在这个方法中通过 `mHandler.getLooper().getQueue().postSyncBarrier()`调用了 MessageQueue 的同步屏障方法，从而插入了一个同步屏障消息。

如果你对 Android 的屏幕绘制流程有一定了解的话，应该知道 Vsync 信号与 Choreographer。 Choreographer 会向系统底层订阅 Vsync 信号,系统底层会间隔大约16ms（60hz刷新率的屏幕）发送一次 Vsync信号。等到接收到 Vsync 信号后，Choreographer 会回调 ViewRootImpl 的 doTraversal 方法开始 View 树真正的遍历与绘制。doTraversal 源码如下：


```java
// ViewRootImpl

final class TraversalRunnable implements Runnable {
   @Override
   public void run() {
      doTraversal();
   }
}

   void doTraversal() {
      if (mTraversalScheduled) {
         mTraversalScheduled = false;
         // 移除同步屏障
         mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

         if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
         }
         //  通过该方法开启View的绘制流程，会调用performMeasure方法、performLayout方法和performDraw方法。
         performTraversals();

         if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
         }
      }
   }
```
可以看到在doTraversal方法中会将同步屏障消息移除掉，之后普通消息又会得到执行的机会。其实到这里也很容易理解为什么 postSyncBarrier 方法不允许开发者调用了。因为一旦开发者执行这个方法，且没有及时移除同步屏障就会导致普通消息再也没有被执行的机会。

#### （2）普通消息和延迟消息

关于普通消息和延迟消息其实没有什么可说的，普通消息的优先级比异步消息低毋庸置疑。而延迟消息由于在插入消息队列时会根据延迟时间确定插入到队列中的位置，即延迟越久的消息在队列中的位置越靠后。因此延迟消息的优先级是最低的。

#### （3）IdleHandler

除了以上消息的优先级外，还有一种叫 IdelHandler 的消息(这里冒昧的称它为消息)，IdelHandler 从本质上来说并不是一个 Message，而是一个接口，其源码如下：

```java
public static interface IdleHandler {
    /**
     * Called when the message queue has run out of messages and will now
     * wait for more.  Return true to keep your idle handler active, false
     * to have it removed.  This may be called if there are still messages
     * pending in the queue, but they are all scheduled to be dispatched
     * after the current time.
     */
    boolean queueIdle();
}
```
通过注释可以看得出来 queueIdle 方法是在 MessageQueue 中的消息执行完后或者有延迟消息在等待执行时才会被调用。因此可以看得出 IdleHandler 执行的优先级是比异步消息和普通消息低的，但要比延迟消息优先级高。接下来我们看一下IdleHandler的具体源码实现。

```java
public final class MessageQueue {
    
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    
    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }
    
    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }
    
}
```
可以看到在 MessageQueue 中有一个泛型为 IdleHandler 的 ArrayList 的成员变量，并且提供了addIdleHandler 方法可以向 mIdleHandlers 中添加 IdelHandler,同时也提供了 removeIdleHandler 来移除 IdleHandler。

那接下来继续看 MessageQueue 的 next 方法中对于 IdleHandler 的处理逻辑。


```java
Message next() {
    // pendingIdleHandlerCount 默认是-1，小于0
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        // ...
        
        int pendingIdleHandlerCount = -1;

        synchronized (this) {
         

            // ... 省略异步消息与普通消息的处理逻辑

            // 如果第一次调用next方法，pendingIdleHandlerCount 一定小于0
            // mMessage == null 说明消息队列中没有消息
            // now < mMessages.when 说明有延迟消息，但是还没有到执行的时间
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                // 获取IdleHandler的个数    
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            // 说明没有 IdleHandler
            if (pendingIdleHandlerCount <= 0) {
                mBlocked = true;
                跳过下面的逻辑，继续执行for循环
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            // 将mIdleHandlers中的数据复制到mPendingIdleHandlers数组中
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // 遍历 mPendingIdleHandlers 数组
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            // 取出遍历到的IdleHandler
            final IdleHandler idler = mPendingIdleHandlers[i];
            // 将以取出的位置设置为null
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                // 执行Idler.queueIdle方法
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            // 这里可以看出 queueIdle 如果返回false,则会将这个IdleHandler从集合中移除，下次就不会再执行了。
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
    }
}
```

代码中的注释已经写得非常详细了，可以看得出来 IdleHandler 只有在 MessageQueue 没有消息时或者延迟消息没有到执行时间时才会执行 IdleHandler 的 queueIdle 方法。并且 queueIdle 方法的返回值确定了是否会将这个 IdleHandler 从集合中移除。

消息机制的设计者设计 IdelHandler 的目的就是为了执行一些不那么紧急的任务，在异步消息和普通消息执行完后，处于空闲时间时才会开始执行 IdleHandler。

而 IdleHandler 在 Framework 的源码中也是被频繁用到的。典型的用法是 Activity 的 onDestroy 生命周期的调用，就是通过向 MessageQueue 中添加 IdleHandler 来实现的。也就是说当 Activity 执行了 finish 方法后并不会立即执行 onDestory 方法，而是要等到消息队列空闲时 onDestory 才会被调用。

如果是这样的话，在Activity调用 finish 时，不断的向 MessageQueue 中插入消息，是不是会导致 Activity 的 onDestory 一直不会被调用呢？理论上是这样的，但是在系统的源码中做了一个兜底，即如果finish之后过了十秒 Activity 依然没有被销毁则会主动调用 Activity 的 onDestory 来执行销毁逻辑。

关于 onDestory 部分的源码分析可以参考路遥的一篇文章[《面试官：为什么 Activity.finish() 之后 10s 才 onDestroy ？》](https://juejin.cn/post/6898588053451833351)，这里就不做过多解读了。

## 四、Handler 的设计思想在开发中的应用


### 1.Handler 与卡顿监控


导致卡顿的原因一般是因为主线程里有耗时操作。由于主线程中只有一个Looper,且主线程是被 Looper.loop()阻塞着的。所以可以通过监控主线程的 Message 执行时间来找出耗时的地方。


在Looper 的 loop 方法中有这样一段代码：


```
public static void loop() {
    ...

    for (;;) {
        ...

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        ...
    }
}
```

如果想要监控 Message 的执行耗时，只需要自定义一个Printer即可：

```
class LooperMonitor implements Printer {  
    
    private boolean mPrintingStarted = false;  
  
    @Override
    public void println(String x) {
        if (!mStartedPrinting) {
            mStartTimeMillis = System.currentTimeMillis();
            mStartThreadTimeMillis = SystemClock.currentThreadTimeMillis();
            mStartedPrinting = true;
        } else {
            final long endTime = System.currentTimeMillis();
            mStartedPrinting = false;
            if (isBlock(endTime)) {
                notifyBlockEvent(endTime);
            }
        }
    }
    
    private boolean isBlock(long endTime) {
        return endTime - mStartTimeMillis > mBlockThresholdMillis;
    }
}    
    
```

然后将这个自定义的 Printer 设置到主线程的 Looper 中，就可以监控到所有主线程消息的耗时了。

```
Looper.getMainLooper().setMessageLogging(mainLooperPrinter);
```

### 2.捕获异常，让 APP不再Crash.

为什么Android程序发生空指针等异常时，会导致应用会崩溃？

```
publicclass RuntimeInit {
    finalstatic String TAG = "AndroidRuntime";
  
    ....
      
    privatestaticclass LoggingHandler implements Thread.UncaughtExceptionHandler {
        publicvolatileboolean mTriggered = false;

        @Override
        public void uncaughtException(Thread t, Throwable e) {
            mTriggered = true;

            if (mCrashing) return;
                //打印异常日志
            if (mApplicationObject == null && (Process.SYSTEM_UID == Process.myUid())) {
                Clog_e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
            } else {
                StringBuilder message = new StringBuilder();
                message.append("FATAL EXCEPTION: ").append(t.getName()).append("\n");
                final String processName = ActivityThread.currentProcessName();
                if (processName != null) {
                    message.append("Process: ").append(processName).append(", ");
                }
                message.append("PID: ").append(Process.myPid());
                Clog_e(TAG, message.toString(), e);
            }
        }
    }

    privatestaticclass KillApplicationHandler implements Thread.UncaughtExceptionHandler {
        privatefinal LoggingHandler mLoggingHandler;
        public KillApplicationHandler(LoggingHandler loggingHandler) {
            this.mLoggingHandler = Objects.requireNonNull(loggingHandler);
        }

        @Override
        public void uncaughtException(Thread t, Throwable e) {
            try {
                ensureLogging(t, e);

                if (mCrashing) return;
                mCrashing = true;
                if (ActivityThread.currentActivityThread() != null) {
                    ActivityThread.currentActivityThread().stopProfiling();
                }

                ActivityManager.getService().handleApplicationCrash(
                        mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e));
            } catch (Throwable t2) {
                if (t2 instanceof DeadObjectException) {
                } else {
                    try {
                        Clog_e(TAG, "Error reporting crash", t2);
                    } catch (Throwable t3) {
                    }
                }
            } finally {
                //杀死进程
                Process.killProcess(Process.myPid());
                System.exit(10);
            }
        }
        private void ensureLogging(Thread t, Throwable e) {
            if (!mLoggingHandler.mTriggered) {
                try {
                    mLoggingHandler.uncaughtException(t, e);
                } catch (Throwable loggingThrowable) {
                }
            }
        }
    
      ....
    }
  
  
    protected static final void commonInit() {
                //设置异常处理回调
        LoggingHandler loggingHandler = new LoggingHandler();
        Thread.setUncaughtExceptionPreHandler(loggingHandler);
        Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));
      
                ....
    }
```

`RuntimeInit`有两个的内部类，`LoggingHandler`和`KillApplicationHandler`。`LoggingHandler`的作用是打印异常日志，而`KillApplicationHandler`就是App发生`Crash`的真正原因，其内部调用了`Process.killProcess(Process.myPid())`来杀死发生`Uncaught`异常的进程。




这两个内部类都实现了`Thread.UncaughtExceptionHandler`接口。分别通过`Thread.setUncaughtExceptionPreHandler`和`Thread.setDefaultUncaughtExceptionHandler`方法进行注册




-   Thread.setUncaughtExceptionPreHandler，覆盖所有线程，会在回调DefaultUncaughtExceptionHandler之前调用，只能在Android Framework内部调用该方法
-   Thread.setDefaultUncaughtExceptionHandler，如果在任意线程中调用即可覆盖所有线程的异常，可以在应用层调用，每次调用传入的Thread.UncaughtExceptionHandler都会覆盖上一次的，即我们可以手动覆盖系统实现的KillApplicationHandler
-   new Thread().setUncaughtExceptionHandler()，只可以覆盖当前线程的异常，如果某个Thread有定义UncaughtExceptionHandler，则忽略全局DefaultUncaughtExceptionHandler

Uncaught异常发生时会终止线程，此时，系统便会通知`UncaughtExceptionHandler`，告诉它被终止的线程以及对应的异常， 然后便会调用`uncaughtException`函数。如果该UncaughtExceptionHandler没有被显式设置，则会调用对应线程组的默认UncaughtExceptionHandlerr。如果我们要捕获该异常，必须实现我们自己的handler




应用层调用`Thread.setDefaultUncaughtExceptionHandler`来实现所有线程的`Uncaught`异常的监听，并且会覆盖系统的默认实现的`KillApplicationHandler`，这样就可以做到让线程发生Uncaught异常的时候只是当前杀死线程，而不会杀死整个进程。这适用于子线程发生Uncaught异常。如果主线程发生Uncaught异常呢？主线程都被销毁了，这和Crash似乎就没什么区别的。那么有办法让主线程发生Uncaught异常也不会发生Crash吗？




由于整个系统都是基于消息机制，主线程的异常一定会经过Looper.loop()，所以其实只要try catch Looper.loop()即可捕获主线程异常。

```
publicclass CrashCatch {

    private CrashHandler mCrashHandler;

    privatestatic CrashCatch mInstance;

    private CrashCatch(){

    }

    private static CrashCatch getInstance(){
        if(mInstance == null){
            synchronized (CrashCatch.class){
                if(mInstance == null){
                    mInstance = new CrashCatch();
                }
            }
        }

        return mInstance;
    }

    public static void init(CrashHandler crashHandler){
        getInstance().setCrashHandler(crashHandler);
    }

    private void setCrashHandler(CrashHandler crashHandler){

        mCrashHandler = crashHandler;
        //主线程异常拦截
        new Handler(Looper.getMainLooper()).post(new Runnable() {
            @Override
            public void run() {
                for (;;) {
                    try {
                        Looper.loop();
                    } catch (Throwable e) {
                        if (mCrashHandler != null) {
                          //处理异常
                    mCrashHandler.handlerException(Looper.getMainLooper().getThread(), e);
                        }
                    }
                }
            }
        });
      
         //所有线程异常拦截，由于主线程的异常都被我们catch住了，所以下面的代码拦截到的都是子线程的异常
        Thread.setDefaultUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            @Override
            public void uncaughtException(Thread t, Throwable e) {
                if(mCrashHandler!=null){
                  //处理异常
                   mCrashHandler.handlerException(t,e);
                }
            }
        });
    }

    publicinterface CrashHandler{
        void handlerException(Thread t,Throwable e);
    }
}
```

## 五、总结

本篇文章的内容比较长，文章从 ”生产者-消费者“模型来对比Android消息机制的实现，并尝试站在设计者的角度分析应该怎样设计系统的消息机制，还尝试分析了在实现过程中碰到的问题及解决方案。如果你能细心的看完这篇文章，一定会有所收获，并且会对 Android 的消息机制有一个全新的理解。

## 推荐阅读

[《追根溯源—— 探究Handler的实现原理》](https://blog.csdn.net/qq_20521573/article/details/77919141?spm=1001.2014.3001.5502)

[《Java并发系列番外篇：ThreadLocal原理其实很简单》](https://juejin.cn/post/6986301941269659656)

[《深入理解Java线程的等待与唤醒机制》](https://juejin.cn/post/6980002998361522190)

[《面试官：为什么 Activity.finish() 之后 10s 才 onDestroy ？》](https://juejin.cn/post/6898588053451833351)

[你知道 Android 为什么会 Crash 吗](https://mp.weixin.qq.com/s/cHXrB582Op1lu3ZlrLKYcg)