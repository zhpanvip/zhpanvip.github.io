---
layout: article
index_img: https://gitee.com/zhpanvip/images/raw/master/blog/img/binder_aidl.png
title: 不得不说的Android Binder机制与AIDL
date: 2021-08-06 00:48:46
categories:
- Android进阶
tags: [Binder]
---


说起Android的进程间通信，想必大家都会不约而同的想起Android中的Binder机制。而提起Binder，想必也有不少同学会想起初学Android时被Binder和AIDL支配的恐惧感。但是作为一个Android开发者，Binder是我们必须掌握的知识。因为它是构架整个Android大厦的钢筋和混凝土，连接了Android各个系统服务和上层应用。只有了解了Binder机制才能更加深入的理解Android开发和Android Framework。这也是为什么无论是《Android开发艺术探索》还是《深入理解Android内核涉及思想》这些进阶类书籍把进程间通信和Binder机制放到靠前章节的原因，它太重要了，重要到整个Android Framework都离不开Binder的身影。

本篇文章我们暂且不去探讨Binder的底层实现，因为就目前而言，笔者掌握的程度也不足以去输出Binder实现原理的的内容。因此，为了不误导大家，这里就来写一写Binder的基础用法以及AIDL。虽然关于Binder和AIDL的基础用法网上的内容比比皆是。但是浅显易懂的内容并不多见。也就导致了很多人看完后仍不知所云。本篇文章，我将通过我自己的学习思路来带大家认识Binder机制和AIDL。


## 一、进程间通信

在操作系统中，每个进程都有一块独立的内存空间。为了保证进程的安全性，操作系统都会有一套严格的安全机制来禁止进程间的非法访问。毕竟，如果你的APP能访问到别的APP的运行空间，或者别的APP可以轻而易举的访问到你APP的运行空间，想象一下你是不是崩溃的心都有了。所以，操作系统层面会对应用进程进行内存隔离，以保证APP的运行安全。但是，很多情况下进程间也是需要相互通信的，例如剪贴板的功能，可以从一个程序中复制信息到另一个程序。这就是进程间通信诞生的背景。

> 广义的讲，进程间通信（Inter-process communication,简称IPC）是指运行在不同进程中的若干线程间的数据交换。

操作系统中常见的进程间通信方式有共享内存、管道、UDS以及Binder等。关于这些进程间的通信方式本篇文章我们不做深究，了解即可。

- **共享内存（Shared Memory)** 共享内存方式实现进程间通信依靠的是申请一块内存区域，让后将这块内存映射到进程空间中，这样两个进程都可以直接访问这块内存。在进行进程间通信时，两个进程可以利用这块内存空间进行数据交换。通过这样的方式，减少了数据的赋值操作，因此共享内存实现进程间通信在速度上有明显优势。

- **管道（Pipe）** 管道也是操作系统中常见的一种进程间通信方式，Windows系统进程间的通信依赖于此中方式实现。在进行进程间通信时，会在两个进程间建立一根拥有读（read）写(write)功能的管道,一个进程写数据，另一个进程可以读取数据，从而实现进程间通信问题。

- **UDS（UNIX Domain Socket）** UDS也被称为IPC Socket，但它有别于network 的Socket。UDS的内部实现不依赖于TCP/IP协议，而是基于本机的“安全可靠操作”实现。UDS这种进程间通信方式在Android中用到的也是比较多的。

- **Binder** Binder是Android中独有的一种进程间通信方式。它底层依靠mmap,只需要一次数据拷贝，把一块物理内存同时映射到内核和目标进程的用户空间。

本篇文章，我们重点就是了解如何使用Binder实现进程间通信。Binder仅仅从名字来看就给人一种很神秘的感觉，就因为这个名字可能就会吓走不少初学者。但其实Binder本身并没有很神秘，它仅仅是Android系统提供给开发者的一种进程间通信方式。

而从上述几种进程间通信方式来看，无论是哪种进程间通信，都是需要一个进程提供数据，一个进程获取数据。因此，我们可以把提供数据的一端称为**服务端**，把获取数据的一端称为**客户端**。从这个角度来看Binder是不是就有点类似于HTTP协议了？所以，你完全可以把Binder当成是一种HTTP协议，客户端通过Binder来获取服务端的数据。认识到这一点，再看Binder的使用就会简单多了。

## 二、使用Binder实现进程间通信

使用Binder完成进程间通信其实非常简单。我们举一个查询成绩的例子，服务端提供根据学生姓名查询学生成绩的接口，客户端连接服务端通过学生姓名来查询成绩，而客户端与服务端的媒介就是Binder。

### 1.服务端的实现

服务端自然是要提供服务的，因此就需要我们开启一个Service等待客户端的连接。关于Android的Service这里就不用多说了吧，我们实现一个GradeService并继承Service，来提供成绩查询接口。代码如下:


```java
public class GradeService extends Service {
    public static final int REQUEST_CODE=1000;
    private final Binder mBinder = new Binder() {
        @Override
        protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags) throws RemoteException {
            if (code == REQUEST_CODE) {
                String name = data.readString();
                // 根据姓名查询学生成绩并将成绩写入到返回数据
                int studentGrade = getStudentGrade(name);
                if (reply != null)
                    reply.writeInt(studentGrade);
                return true;
            }
            return super.onTransact(code, data, reply, flags);
        }
        // 根据姓名查询学生成绩
        public int getStudentGrade(String name) {
            return StudentMap.getStudentGrade(name);
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```
因为我们要实现的是跨进程通信，因此，我们将这个服务端的Service设置到远程进程中，在AndroidManifest文件中如下：

```
<service
    android:name="com.zhpan.sample.binder.server.GradeService"
    android:process=":server">
    <intent-filter>
        <action android:name="android.intent.action.server.gradeservice" />
    </intent-filter>
</service>
```

就这样，一个远程的，提供成绩查询的服务就完成了。

### 2.客户端的实现

客户端自然而然的是要连接服务端进程成绩查询。因此，我们在客户端的Activity中取绑定GradeService进行成绩查询。代码如下：

```java
public class BinderActivity extends AppCompatActivity {
    // 远程服务的Binder代理
    private IBinder mRemoteBinder;

    private final ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            // 获取远程服务的Binder代理
            mRemoteBinder = iBinder;
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mRemoteBinder = null;
        }
    };



    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_binder);
        // 绑定服务
        findViewById(R.id.btn_bind_service).setOnClickListener(view -> bindGradeService());
        // 查询学生成绩
        findViewById(R.id.btn_find_grade).setOnClickListener(view -> getStudentGrade("Anna"));
    }
    // 绑定远程服务
    private void bindGradeService() {
        String action = "android.intent.action.server.gradeservice";
        Intent intent = new Intent(action);
        intent.setPackage(getPackageName());
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
    }
    // 从远程服务查询学生成绩
    private int getStudentGrade(String name) {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        int grade = 0;
        data.writeString(name);
        try {
            if (mRemoteBinder == null) {
                throw new IllegalStateException("Need Bind Remote Server...");
            }
            mRemoteBinder.transact(REQUEST_CODE, data, reply, 0);
            grade = reply.readInt();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return grade;
    }
```


就这么简单，一个用Binder实现的进程通信就完成了。是不是有点出乎你所料？但是Binder的使用就是这么简单呀。那我们之前写的AIDL呢？AIDL生成的那一堆是什么玩意儿？莫慌，我们且往下看。

## 三、代理模式优化Binder使用

首先，我们的服务端保持跟上一章不变，客户端使用代理模式进行优化，让代理类来完成Binder的任务。

接下来，需要一个接口，代码如下：

```java
public interface IGradeInterface {
    int getStudentGrade(String name);
}
```
接着，我们实现一个自定义的Binder，并实现上述接口，代码如下：

```java
public class GradeBinder extends Binder implements IGradeInterface {

    @Override
    public int getStudentGrade(String name) {
        return StudentMap.getStudentGrade(name);
    }

    @Override
    protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags) throws RemoteException {
        if (code == REQUEST_CODE) {
            String name = data.readString();
            int studentGrade = getStudentGrade(name);
            if (reply != null)
                reply.writeInt(studentGrade);
            return true;
        }
        return super.onTransact(code, data, reply, flags);
    }
}
```
这段代码比较容易理解，就不再多费口舌了。接下来来看代理类的实现，它会持有Binder，并替代Binder来完成进程间通信。代码如下：

```java
public class BinderProxy implements IGradeInterface {
    // 被代理的Binder
    private final IBinder mBinder;

    private BinderProxy(IBinder binder) {
        mBinder = binder;
    }

  	// 通过Binder查询成绩
    @Override
    public int getStudentGrade(String name) {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        int grade = 0;
        data.writeString(name);
        try {
            if (mBinder == null) {
                throw new IllegalStateException("Need Bind Remote Server...");
            }
            mBinder.transact(REQUEST_CODE, data, reply, 0);
            grade = reply.readInt();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return grade;
    }

    // 实例化Binder代理类的对象
    public static IGradeInterface asInterface(IBinder iBinder) {
        if (iBinder == null) {
            return null;
        }

        if (iBinder instanceof IGradeInterface) {
            LogUtils.e("当前进程");
            // 如果是同一个进程的请求，则直接返回Binder
            return (IGradeInterface) iBinder;
        } else {
            LogUtils.e("远程进程");
            // 如果是跨进程查询则返回Binder的代理对象
            return new BinderProxy(iBinder);
        }
    }

}
```
上述代码asInterface方法中通过判断Binder是不是IGradeInterface从而确定是不是跨进程的通信。如果不是跨进程通信，则返回当前这个Binder，否则就返回Binder的这个代理类。

这样，我们便完了客户端的代码优化，接着来看客户端的连接代码，如下：

```java
public class BinderProxyActivity extends BaseViewBindingActivity<ActivityBinderBinding> {
    // 此处可能是BinderProxy也可能是GradeBinder
    private IGradeInterface mBinderProxy;

    private final ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            // 连接服务成功，根据是否跨进程获取BinderProxy或者GradeBinder实例
            mBinderProxy = BinderProxy.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mBinderProxy = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding.btnBindService.setOnClickListener(view -> bindGradeService());
      	// 查询学生成绩点击事件，通过mBinderProxy查询成绩
        binding.btnFindGrade.setOnClickListener(view -> ToastUtils.showShort("Anna grade is " + mBinderProxy.getStudentGrade("Anna")));
    }

		// 绑定服务
    private void bindGradeService() {
        String action = "android.intent.action.server.gradeservice";
        Intent intent = new Intent(action);
        intent.setPackage(getPackageName());
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
    }

}
```
可以看到，这时候的代码整洁了不少。

## 四、AIDL

终于到了最让人头大的AIDL环节了，但是，这里我保证对于AIDL绝对让你一看就会。AIDL是Android Interface Description Languaged 简写。用于描写客户端/服务端通信接口的一种描述语言。其实说人话就是通过定义AIDL接口来生成第二章中的代理类。即AIDL的原理其实跟上一章的代理模式优化Binder的使用是一样的。目的就是为了省去我们自己编写代理的代码。AIDL的使用非常简单，大致步骤如下：

### 1.创建AIDL接口

首先，我们在要创建AIDL的目录上右键->New->AIDL->AIDl File 来创建一个AIDL文件，如下图所示：

![截屏2021-08-06 上午12.29.39.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07778cc901ce4de3a0717e4ba65809c7~tplv-k3u1fbpfcp-watermark.image)


创建一个名为IGradeService的AIDL文件，如下：

```java
// IGradeService.aidl
package com.zhpan.sample.binder.aidl;

// Declare any non-default types here with import statements

interface IGradeService {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

    int getStudentGrade(String name);
}
```
代码很简单，AIDL中有一个getStudentGrade的方法。

### 2.AIDL生成的代码

接着Rebuild项目后就会看到在build目录下com.zhpan.sample.binder.aidl包中生成了一个名为IGradeService的接口，代码如下：

```java
// 这个接口相当于上一章中的IGradeInterface接口
public interface IGradeService extends android.os.IInterface {

  ...

  // Stub是一个Binder，相当于上一章中的GradeBinder
  public static abstract class Stub extends android.os.Binder
      implements com.zhpan.sample.binder.aidl.IGradeService {
    private static final java.lang.String DESCRIPTOR = "com.zhpan.sample.binder.aidl.IGradeService";

    public Stub() {
      this.attachInterface(this, DESCRIPTOR);
    }

    public static IGradeService asInterface(android.os.IBinder obj) {

      if ((obj == null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin != null) && (iin instanceof com.zhpan.sample.binder.aidl.IGradeService))) {
        // 如果是当前进程则直接返回当前Binder对象
        return ((com.zhpan.sample.binder.aidl.IGradeService) iin);
      }
      // 跨进程则返回Binder的代理对象
      return new com.zhpan.sample.binder.aidl.IGradeService.Stub.Proxy(obj);
    }

    @Override public android.os.IBinder asBinder() {
      return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
        throws android.os.RemoteException {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code) {
        case INTERFACE_TRANSACTION: {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_basicTypes: {
          data.enforceInterface(descriptor);
          int _arg0;
          _arg0 = data.readInt();
          long _arg1;
          _arg1 = data.readLong();
          boolean _arg2;
          _arg2 = (0 != data.readInt());
          float _arg3;
          _arg3 = data.readFloat();
          double _arg4;
          _arg4 = data.readDouble();
          java.lang.String _arg5;
          _arg5 = data.readString();
          this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
          reply.writeNoException();
          return true;
        }
        case TRANSACTION_getStudentGrade: {
          data.enforceInterface(descriptor);
          java.lang.String _arg0;
          _arg0 = data.readString();
          int _result = this.getStudentGrade(_arg0);
          reply.writeNoException();
          reply.writeInt(_result);
          return true;
        }
        default: {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }
    // Binder的代理类，相当于上一章中的BinderProxy
    private static class Proxy implements com.zhpan.sample.binder.aidl.IGradeService {
      private android.os.IBinder mRemote;

      Proxy(android.os.IBinder remote) {
        mRemote = remote;
      }

      @Override public android.os.IBinder asBinder() {
        return mRemote;
      }

      public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
      }

      @Override public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
          double aDouble, java.lang.String aString) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeInt(anInt);
          _data.writeLong(aLong);
          _data.writeInt(((aBoolean) ? (1) : (0)));
          _data.writeFloat(aFloat);
          _data.writeDouble(aDouble);
          _data.writeString(aString);
          boolean _status = mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().basicTypes(anInt, aLong, aBoolean, aFloat, aDouble, aString);
            return;
          }
          _reply.readException();
        } finally {
          _reply.recycle();
          _data.recycle();
        }
      }

      @Override public int getStudentGrade(java.lang.String name)
          throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeString(name);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getStudentGrade, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getStudentGrade(name);
          }
          _reply.readException();
          _result = _reply.readInt();
        } finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }

      public static com.zhpan.sample.binder.aidl.IGradeService sDefaultImpl;
    }

    static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_getStudentGrade = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

    public static boolean setDefaultImpl(com.zhpan.sample.binder.aidl.IGradeService impl) {
      if (Stub.Proxy.sDefaultImpl == null && impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }

    public static com.zhpan.sample.binder.aidl.IGradeService getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }

  /**
   * Demonstrates some basic types that you can use as parameters
   * and return values in AIDL.
   */
  public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble,
      java.lang.String aString) throws android.os.RemoteException;

  public int getStudentGrade(java.lang.String name) throws android.os.RemoteException;
}
```

上述代码中的重要位置都加了代码注释，可以看到，通过AIDL生成的代码和上一章我们使用代理模式优化后的代码基本原理是一致的，只不过生成的代码都被放在了同一个类中。到这里，是不是突然觉得AIDL原来就这么简单啊！

好了，我们来看下客户端的使用吧。

### 3.AIDL客户端
使用AIDL的客户端实现也非常简单，几乎与第三章中的代码一致，如下：

```java
public class AidlActivity extends BaseViewBindingActivity<ActivityBinderBinding> {

    private IGradeService mBinderProxy;

    private final ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            // 连接服务后，根据是否跨进程获取Binder或者Binder的代理对象
            mBinderProxy = IGradeService.Stub.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mBinderProxy = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding.btnBindService.setOnClickListener(view -> bindGradeService());
      	// 查询学生成绩
        binding.btnFindGrade.setOnClickListener(view -> getStudentGrade("Anna"));
    }

    // 绑定服务
    private void bindGradeService() {
        String action = "android.intent.action.server.aidl.gradeservice";
        Intent intent = new Intent(action);
        intent.setPackage(getPackageName());
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
    }

    // 查询成绩
    private void getStudentGrade(String name) {
        int grade = 0;
        try {
            grade = mBinderProxy.getStudentGrade(name);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        ToastUtils.showShort("Anna grade is " + grade);
    }
}
```

## 五、总结

本篇文章主要带大家认识了进程间通信和Binder与AIDL的使用。通过本篇文章的学习可以发现Binder与AIDL其实是非常简单的。了解了Binder之后，我们就可以去更加深入的学习Android Framework层的知识了。






