---
layout: article
index_img: https://cdn.jsdelivr.net/gh/zhpanvip/images/blog/img/dynamic-proxy.png
title: 浅析 Java 中的动态代理
date: 2021-08-22 11:54:41
categories:
- Java进阶
tags: [设计模式,动态代理]
---


在之前的一篇文章[《静态代理这么用？聊一聊ViewPagerIndicator重构的一些经验》](https://juejin.cn/post/6844904003524886536)中详细的介绍了 java 中的静态代理，并且使用静态代理对[IndicatorView](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fzhpanvip%2Fviewpagerindicator)进行了重构。静态代理的优点不必多说，它可以让代码具有扩展性，也可以让代码解耦。但在现实开发中，静态代理有时候也存在很多弊端，列举如下：

-   当接口需要增加、删除、修改方法时，被代理类与代理类都需要修改，不易维护。
-   由于代理类要实现与被代理类一致的接口，当有多个类需要被代理时，会存在以下问题：
    - 如果让代理类实现所有被代理类的接口，这样会使代理类过于庞大；
    - 如果使用多个代理类，每个代理类只代理一个类，这样会需要构造多个代理类。


对于以上问题都可以使用动态代理来解决。**动态代理会在运行时根据被代理接口生成对应的代理类的字节码，然后加载到JVM中，并创建这个接口的实例的过程。**

本篇文章就来学习以下动态代理的使用，以及深入的理解动态代理的本质。

开始之前先给大家推荐一下[AndroidNote](https://github.com/zhpanvip/AndroidNote)这个GitHub仓库，这里是我的学习笔记，同时也是我文章初稿的出处。这个仓库中汇总了大量的java进阶和Android进阶知识。是一个比较系统且全面的Android知识库。对于准备面试的同学也是一份不可多得的面试宝典，欢迎大家到GitHub的仓库主页关注。

## 一、从 Retrofit 源码看动态代理

Retrofit 是一个封装了 OKHttp 的网络请求框架，它的使用非常简单。 只需要定义一个接口，然后创建一个Retrofit实例，通过Retrofit实例便可以生成接口的代理对象用来进行网络请求了。

简单的看下Retrofit的使用步骤：

### 1.Retrofit 使用

#### 1）定义接口

首先，定义一个名为GitHub的接口，并在接口中添加一个API方法，如下：

```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Call<List<Contributor>> contributors(@Path("owner") String owner, @Path("repo") String repo);
}
```

#### 2）创建Retrofit实例

Retrofit完美的封装让我们可以通过一个简单的链式调用，就可以生成一个Retrofit实例。代码如下：

```java
Retrofit retrofit =
    new Retrofit.Builder()
        .baseUrl(API_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build();
```

#### 3）生成API接口代理

使用Retrofit实例生成GitHub接口的代理实例，代码如下：

```java
GitHub github = retrofit.create(GitHub.class);
```
#### 4）发起网络请求

使用GitHub接口的代理，调用contributors方法发起网络请求，代码如下：

```java
github.contributors("square", "retrofit").enqueue(new Callback<List<Contributor>>() {
  @Override
  public void onResponse(Call<List<Contributor>> call, Response<List<Contributor>> response) {

  }

  @Override
  public void onFailure(Call<List<Contributor>> call, Throwable t) {

  }
});
```

在上述代码中，我们仅仅是定义了一个GitHub接口，Retrofit内部究竟是怎么生成 GitHub 实例的呢？其实这里就是用到了动态代理来实现的。

### Retrofit 动态代理

Retrofit 的 create 方法接受一个 `Class<T>`类型的参数，并且会返回一个这个`Class<T>`对应的实例对象。这里就是使用动态代理来创建`Class<T>`的实例的。看下代码：


```java
public <T> T create(final Class<T> service) {
  // 校验 service 是否合法
  validateServiceInterface(service);
  return (T)
      Proxy.newProxyInstance(  // 通过 Proxy.newProxyInstance 生成 service 的实例对象
          service.getClassLoader(), // 类加载器
          new Class<?>[] {service}, // 被代理的接口
          new InvocationHandler() { // 一个匿名内部类，是被代理方法真正执行的逻辑
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {

                // ...

              return loadServiceMethod(method).invoke(args);
            }
          });
}
```
上述代码我进行了简化，抛开被代理方法的执行逻辑来着重看静态代理的实现。首先再次明确一下 create 方法返回的是一个 GitHub 接口的代理对象. 这个对象是通过 Proxy.newProxyInstance 方法生成的。newProxyInstance 方法接收三个参数，分别为类加载器、被代理的接口，以及一个InvocationHandler的匿名内部类。

到这里可能有些同学还是没明白到底是怎么回事。让我们再来看一下动态代理的定义 ：**动态代理会在运行时根据被代理接口生成对应的代理类的字节码，然后加载到JVM中，并创建这个接口的实例的过程。**

结合create的代码来说就是这个方法会在运行时通过Proxy.newProxyInstance生成一个代理类，并且会将这个代理类加载到JVM中，并生成代理的实例。 当通过代理接口调用接口中的中的方法时，实际会执行InvocationHandler 的 invoke 方法。

所以，到这里代理与被代理的关系其实应该很清晰了。即 Proxy.newProxyInstance 生成的这个类（也就是本章第1节，第3小节中的github实例）是代理类，而匿名内部类 InvocationHandler 中的 invoke 方法是被代理的方法。所以真正的执行逻辑就在这个 invoke 方法中。



## 二、实现一个动态代理

上一章我们了解了retrofit中的动态代理。本章内容，我们来自己完成一个动态代理的例子，加深对动态代理的理解。仍然以[《静态代理这么用？聊一聊ViewPagerIndicator重构的一些经验》](https://juejin.cn/post/6844904003524886536) 这篇文章中的场景为例：

> Ryan想在上海买一套房子，但是他又不懂房地产的行情，于是委托了中介（Proxy）来帮助他买房子。

### 1.实现动态代理

#### 1）定义代理接口

毫无疑问，应该首先定义一个Buy House 的接口，如下:

```java
public interface IPersonBuyHouse {
    // 特意加了一个返回值，表示是否成功购买
    boolean buyHouse(String name);
}
```
#### 2）使用动态代理

模仿retrofit中create方法的代码，来实现动态代理。

```java
InvocationHandler handler = (proxy, method, args1) -> {
    if (method.getName().equals("buyHouse")) {
        System.out.println(args1[0] + " will buy a house.");
    }
    // 返回true，表示购买成功
    return true;
};
// 通过 Proxy.newProxyInstance 生成代理对象
IPersonBuyHouse proxy = (IPersonBuyHouse)Proxy
        .newProxyInstance(IPersonBuyHouse.class.getClassLoader(), // ClassLoader
                new Class[]{IPersonBuyHouse.class}, // 传入要实现的接口
                handler); // 传入处理调用方法的InvocationHandler
```
#### 3）执行代理

生成代理后，便可以调用代理中的方法来买房子了，代码如下：

```java
boolean success = proxy.buyHouse("Ryan");
```

执行上述代码会得到如下日志：

> Ryan will buy a house.

即 InvocationHandler 的 invoke 方法中输出的日志。


### 2. 动态代理的本质

其实，在第一章中关于动态代理的本质已经解释的很清楚了。Proxy.newProxyInstance 会生成一个IPersonBuyHouse的代理对象。执行代理对象的方法就是执行了InvocationHandler的invoke方法。也就是InvocationHandler其实是被代理的对象。

如果将上一节中的动态代理代码改成静态代理，其实就是这样的：

```java
public class Proxy implements IPersonBuyHouse {
    // 这里可以理解为被代理类 Ryan
    InvocationHandler handler = new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) {
            if (method.getName().equals("buyHouse")) {
                System.out.println(args[0] + " will buy a house.");
            }
            return true;
        }
    };

    // 代理方法
    @Override
    public boolean buyHouse(String name) {
        try {
            // 反射获取buyHouse这个Method
            Method buyHouseMethod = IPersonBuyHouse.class.getDeclaredMethod("buyHouse", String.class);
            // 将buyHouse的参数封装成一个Object数组
            Object[] params = new Object[1];
            params[0] = name;
            // 实际调用了InvocationHandler的invoke方法
            return (boolean) handler.invoke(this, buyHouseMethod, params);
        } catch (Throwable e) {
            e.printStackTrace();
        }
        return false;
    }
}
```
测试代码直接实例化Proxy并调用buyHouse即可

```java
new Proxy().buyHouse("Ryan");
```
打印结果与使用动态代理打印结果一致：

> Ryan will buy a house.

## 总结

本篇文章详细且深入的讲解了动态代理的使用以及动态代理的本质。动态代理使用Proxy.newProxyInstance来生成代理类，解决了静态代理中不能同时代理多个类的弊端。被代理类在动态代理中也没有被显示声明，而是以InvocationHandler的形式来实现。调用代理类的方法实则调用了InvocationHandler的invoke方法。真正的执行逻辑就是在这个invoke方法中。

最后贴下本篇文章的初稿：[AndroidNote#动态代理](https://github.com/zhpanvip/AndroidNote/wiki/%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86),同时，欢迎关注[AndroidNote](https://github.com/zhpanvip/AndroidNote),你会有意想不到的收获。



