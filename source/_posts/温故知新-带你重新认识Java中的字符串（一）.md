---
title: 温故知新--带你重新认识Java中的字符串（一）
date: 2020-09-08 02:08:20
categories:
- Java
tags:
- String
- StringBuilder
---



初学Java时我们已经知道Java中可以分为两大数据类型，分别为基本数据类型和引用数据类型。而在这两大数据类型中有一个特殊的数据类型String，String属于引用数据类型，但又有区别于其它的引用数据类型。可以说它是数据类型中的一朵奇葩。那么，本篇文章我们就来深入的认识一下Java中的String字符串。

## 一、从String字符串的内存分配说起

上一篇文章[《温故知新--你不知道的JVM内存分配》](https://blog.csdn.net/qq_20521573/article/details/108394326)详细的分析了JVM的内存模型。在常量池部分我们了解了三种常量池，分别为：字符串常量池、Class文件常量池以及运行时常量池。而字符串的内存分配则和字符串常量池有着莫大的关系。

我们知道，实例化一个字符串可以通过两种方法来实现，第一种最常用的是通过字面量赋值的方式，另一种是通过构造方法传参的方式。代码如下：

```java
	String str1="abc";
	String str2=new String("abc");
```
这两种方式在内存分配上有什么不同呢? 相信大家在初学Java的时候老师都有给我们讲解过：

> 1.通过字面量赋值的方式创建String，只会在字符串常量池中生成一个String对象。
> 2.通过构造方法传入String参数的方式会在堆内存和字符串常量池中各生成一个String对象，并将堆内存上String的引用放入栈。

这样的回答正确吗？至少在现在看来并不完全正确，因为它完全取决于使用的Java版本。上一篇文章[《温故知新--你不知道的JVM内存分配》](https://blog.csdn.net/qq_20521573/article/details/108394326)谈到HotSpot虚拟机在不同的JDK上对于字符串常量池的实现是不同的，摘录如下：

>在JDK7以前，字符串常量池在方法区（永久代）中，此时常量池中存放的是字符串对象。而在JDK7中，字符串常量池从方法区迁移到了堆内存，同时将字符串对象存到了Java堆，字符串常量池中只是存入了字符串对象的引用。


这句话应该怎么理解呢？我们以String str1=new String("abc")为例来分析：
### 1.JDK6中的内存分配
先来分析一下JDK6的内存分配情况，如下图所示：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9jOTc5MDFhMGFmZDI0Nzk2YmFjYTJhNTkwOGIwZDI2ZX50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
当调用new String("abc")后，会在Java堆与常量池中各生成一个“abc”对象。同时，将str1指向堆中的“abc”对象。
### 2.JDK7中的内存分配
而在JDK7及以后版本中，由于字符串常量池被移到了堆内存，所以内存分配方式也有所不同，如下图所示：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9lZWM2NDYyYTNkNzE0NjllYjEzMmZlZDAwYjAxNmVlOH50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
当调用了new String("abc")后，会在堆内存中创建两个“abc"对象，str1指向其中一个”abc"对象，而常量池中则会生成一个“abc"对象的引用，并指向另一个”abc"对象。

至于Java中为什么要这么设计，我们在上篇文章中也已经解释了：** 因为String是Java中使用最频繁的一种数据类型，为了节省程序内存提高程序性能，Java的设计者们开辟了一块字符串常量池区域，这块区域是是所有类共享的，每个虚拟机只有一个字符串常量池。因此，在使用字面量方式赋值的时候，如果字符串常量池中已经有了该字符串，则不会在堆内存中重新创建对象，而是直接将其指向了字符串常量池中的对象。**


## 二、String的intern()方法
在了解了String的内存分配之后，我们需要再来认识一下String中一个很重要的方法：String.intern()。

很多读者可能对于这一方法并不是太了解，但并不代表他不重要。我们先来看一下intern()方法的源码：

```java
/**
     * Returns a canonical representation for the string object.
     * <p>
     * A pool of strings, initially empty, is maintained privately by the
     * class {@code String}.
     * <p>
     * When the intern method is invoked, if the pool already contains a
     * string equal to this {@code String} object as determined by
     * the {@link #equals(Object)} method, then the string from the pool is
     * returned. Otherwise, this {@code String} object is added to the
     * pool and a reference to this {@code String} object is returned.
     * <p>
     * It follows that for any two strings {@code s} and {@code t},
     * {@code s.intern() == t.intern()} is {@code true}
     * if and only if {@code s.equals(t)} is {@code true}.
     * <p>
     * All literal strings and string-valued constant expressions are
     * interned. String literals are defined in section 3.10.5 of the
     * <cite>The Java&trade; Language Specification</cite>.
     *
     * @return  a string that has the same contents as this string, but is
     *          guaranteed to be from a pool of unique strings.
     */
    public native String intern();
```
emmmmm....居然是一个native方法，不过没关系，即使看不到源码我们也能从其注释中得到一些信息：**当调用intern方法的时候，如果字符串常量池中已经包含了一个等于该String对象的字符串，则直接返回字符串常量池中该字符串的引用。否则，会将该字符串对象包含的字符串添加到常量池，并返回此对象的引用。**

### 1.一个关于intern()的简单例子
了解了intern方法的用途之后，来看一个简单的列子：

```java
public class Test {
	public static void main(String[] args) {
		String str1 = "hello world";
		String str2 = new String("hello world");
		String str3=str2.intern();
		System.out.println("str1 == str2:"+(str1 == str2));
		System.out.println("str1 == str3:"+(str1 == str3));
	}
}
```
上面的一段代码会输出什么？编译运行之后如下：
![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9iYmJhOTZkOGYzOWE0ZGU3YjA5ODg0ZTVlNjFiZWI3N350cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)

如果理解了intern方法就很容易解释这个结果了，从上面截图中可以看到，我们的运行环境是JDK8。

***String str1 = "hello world";*** 这行代码会首先在Java堆中创建一个对象，并将该对象的引用放入字符串常量池中，str1指向常量池中的引用。

***String str2 = new String("hello world");***这行代码会通过new来实例化一个String对象，并将该对象的引用赋值给str2，然后检测字符串常量池中是否已经有了与“hello world”相等的对象，如果没有，则会在堆内存中再生成一个值为"hello world"的对象，并将其引用放入到字符串常量池中，否则，不会再去创建。这里，第一行代码其实已经在字符串常量池中保存了“hello world”字符串对象的引用，因此，第二行代码就不会再次向常量池中添加“hello world"的引用。

***String str3=str2.intern();*** 这行代码会首先去检测字符串常量池中是否已经包含了”hello world"的String对象，如果有则直接返回其引用。而在这里，str2.intern()其实刚好返回了第一行代码中生成的“hello world"对象。

因此【System.out.println("str1 == str3:"+(str1 == str3));】这行代码会输出true.

如果切到JDK6，其打印结果与上一致，至于原因读者可以自行分析。
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC85MmM1YmU3Y2ZhNjQ0OGM0YTNkMTEzNTRiOGYzNDBlM350cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
### 2.改造例子，再看intern

上一节中我们通过一个例子认识了intern()方法的作用，接下来，我们对上述例子做一些修改：

```
public class Test {
	public static void main(String[] args) {
		String str1=new String("he")+new String("llo");
		String str2=str1.intern();
		String str3="hello";
		System.out.println("str1 == str2:"+(str1 == str2));
		System.out.println("str1 == str3:"+(str2 == str3));
	}
}
```
先别急着看下方答案，思考一下在JDK7（或JDK7之后）及JDK6上会输出什么结果？
#### 1).JDK8的运行结果分析
我们先来看下我们先来看下JDK8的运行结果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200908100638930.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)

通过运行程序发现输出的两个结果都是true，这是为什么呢？我们通过一个图来分析：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wOS1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC84M2M3Njg1MTZjYzY0NDEzOTRmNTY2ZmViZDBkYzdhN350cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
***String str1=new String("he")+new String("llo");*** 这行代码中new String("he")和new String("llo")会在堆上生成四个对象，因为与本例无关，所以图上没有画出，new String("he")+new String("llo")通过”+“号拼接后最终会生成一个"hello"对象并赋值给str1。

***String str2=str1.intern();*** 这行代码会首先检测字符串常量池，发现此时还没有存在与”hello"相等的字符串对象的引用，而在检测堆内存时发现堆中已经有了“hello"对象，遂将堆中的”hello"对象的应用放入字符串常量池中。

***String str3="hello";*** 这行代码发现字符串常量池中已经存在了“hello"对象的引用，因此将str3指向了字符串常量池中的引用。

此时，我们发现str1、str2、str3指向了堆中的同一个”hello"对象，因此，就有了上边两个均为true的输出结果。

#### 2).JDK6的运行结果分析
我们将运行环境切换到JDK6，来看下其输出结果：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9kMzU4MDk1Y2YzOTY0ODJiODU5M2Y2N2I1OTUzYjQxN350cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
有点意思！相同的代码在不同的JDK版本上输出结果竟然不相等。这是怎么回事呢？我们还通过一张图来分析：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMS1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC85YjVmZjAxMzMwNWE0YThlOGRkMmYyNDQzNTg0MjM0OH50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
***String str1=new String("he")+new String("llo");*** 这行代码会通过new String("he")和new String("llo")会分别在Java堆与字符串常量池中各生成两个String对象，由于与本例无关，所以并没有在图中画出。而new String("he")+new String("llo")通过“+”号拼接后最终会在Java堆上生成一个"hello"对象，并将其赋值给了str1。

***String str2=str1.intern();*** 这行代码检测到字符串常量池中还没有“hello"对象，因此将堆中的”hello“对象复制到了字符串常量池，并将其赋值给str2。

***String str3="hello";*** 这行代码检测到字符串常量池中已经有了”hello“对象，因此直接将str3指向了字符串常量池中的”hello“对象。
此时str1指向的是Java堆中的”hello“对象，而str2和str3均指向了字符串常量池中的对象。因此，有了上面的输出结果。

通过这两个例子，相信大家因该对String的intern()方法有了较深的认识。那么intern()方法具体在开发中有什么用呢？推荐大家可以看下美团技术团队的一篇文章[《深入解析String#intern》](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)中举的两个例子。限于篇幅，本文不再举例分析。


## 三、String类的结构及特性分析
前两节我们认识了String的内存分配以及它的intern()方法，这两节内容其实都是对String内存的分析。到目前为止，我们还并未认识String类的结构以及它的一些特性。那么本节内容我们就此来分析。先通过一段代码来大致了解一下String类的结构（代码取自jdk8）：

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
		/** The value is used for character storage. */
    	private final char value[];
    	/** Cache the hash code for the string */
   		 private int hash; // Default to 0
		// 	...
}
```
可以看到String类实现了Serializable接口、Comparable接口以及CharSequence接口，意味着它可以被序列化，同时方便我们排序。另外，String类还被声明为了final类型，这意味着String类是不能被继承的。而在其内部维护了一个char数组，说明String是通过char数组来实现的，同时我们注意到这个char数组也被声明为了final，这也是我们常说的String是不可变的原因。

### 1.不同JDK版本之间String的差异

Java的设计团队一直在对String类进行优化，这就导致了不同jdk版本上String类的实现有些许差异，只是我们使用上并无感知。下图列出了jdk6-jdk9中String源码的一些变化。
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wOS1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9lYTY3ZmI5ZGNmNWM0NjMwODZkNzQ5OTU3ZDgzZGZhNn50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
可以看到在Java6之前String中维护了一个char 数组、一个偏移量 offset、一个字符数量 count以及一个哈希值 hash。
String对象是通过 offset 和 count 两个属性来定位 char[]  数组，获取字符串。这么做可以高效、快速地共享数组对象，同时节省内存空间，但这种方式很有可能会导致内存泄漏。

在Java7和Java8的版本中移除了 offset 和 count 两个变量了。这样的好处是String对象占用的内存稍微少了些，同时 String.substring 方法也不再共享 char[]，从而解决了使用该方法可能导致的内存泄漏问题。

从Java9开始，String中的char数组被byte[]数组所替代。我们知道一个char类型占用两个字节，而byte占用一个字节。因此在存储单字节的String时，使用char数组会比byte数组少一个字节，但本质上并无任何差别。
另外，注意到在Java9的版本中多了一个coder，它是编码格式的标识，在计算字符串长度或者调用 indexOf() 函数时，需要根据这个字段，判断如何计算字符串长度。coder 属性默认有 0 和 1 两个值， 0 代表Latin-1（单字节编码），1 代表 UTF-16 编码。如果 String判断字符串只包含了 Latin-1，则 coder 属性值为 0 ，反之则为 1。

### 2.String字符串的裁剪、拼接等操作分析
在本节内容的开头我们已经知道了字符串的不可变性。那么为什么我们还可以使用String的substring方法进行裁剪，甚至可以直接使用”+“连接符进行字符串的拼接呢？
#### (1)String的substring实现
关于substring的实现，其实我们直接深入String的源码查看即可，源码如下：


```java
	public String substring(int beginIndex) {
	        if (beginIndex < 0) {
	            throw new StringIndexOutOfBoundsException(beginIndex);
	        }
	        int subLen = value.length - beginIndex;
	        if (subLen < 0) {
	            throw new StringIndexOutOfBoundsException(subLen);
	        }
	        return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
	    }
```
从这段代码中可以看出，其实字符串的裁剪是通过实例化了一个新的String对象来实现的。所以，如果在项目中存在大量的字符串裁剪的代码应尽量避免使用String，而是使用性能更好的StringBuilder或StringBuffer来处理。

#### (2)String的字符串拼接实现
String的字符串拼接通常用到的有两种方法：

1).通过concat方法进行拼接。

2).通过”+“连接符进行拼接。

这两种方法有什么区别呢？哪一种方法的性能会更好一些呢？我们接着往下看。

**1) concat方法的实现原理**

同样，我们直接查看concat方法的源码，如下：

```java
public String concat(String str) {
        int otherLen = str.length();
        if (otherLen == 0) {
            return this;
        }
        int len = value.length;
        char buf[] = Arrays.copyOf(value, len + otherLen);
        str.getChars(buf, len);
        return new String(buf, true);
    }
```
可以看到，在concat方法中使用Arrays的copyOf将原char数组进行了拷贝并赋值给了buf[]，接下来又通过getChars方法再次进行了数组拷贝，最后通过new实例化了String对象并返回，可见通过concat方法的一次操作，在String内部生成了至少三个对象！如果项目中存在大量的使用String的concat方法进行字符串拼接的代码的话，对程序的性能影响可见非同一般。所以在实际开发中我们应该避免使用这一方法。

**2)通过”+“连接符进行字符串拼接**

通过”+“连接符拼接字符串是我们在项目中会经常用到的，这种拼接方法是如何实现的呢？相比 concat方法有什么优越呢？想弄清楚这一点还是有些麻烦的，因为它不像substring和concat方法我们可以从源码中看到实现代码。不过，我们可以通过Java中给我们提供的一个javap工具进行反汇编，通过查看汇编指令来一探究竟！先写一个字符串拼接的代码，如下：

```java
public static void main(String[] args) {
		String str="ab"+"cd";
	}
```
然后执行javap命令，得到如下汇编指令：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC83ODk0NDI4MTgxMjk0NDFmODhjZDVjODIyMWI2ZDg1YX50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
注意一下main方法中”0：“的那行指令，可以看到编译器在编译时已经将“ab"+"cd"优化成了”abcd"。因此，如果是两个常量通过”+“进行拼接是不会有任何的性能损耗的。
但是，到这里有一个问题，如果”+“拼接的不是常量，而是变量会怎样呢？我们将上边代码进行修改后如下：

```java
public static void main(String[] args) {
		String str1="cd";
		String str="ab"+str1;
	}
```
再次进行javap命令，此时可以看到与上边指令有所不同：
![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wMy1qdWVqaW4uYnl0ZWltZy5jb20vdG9zLWNuLWktazN1MWZicGZjcC9iYjA5MDE4MzI3MjU0MWE5ODkxZmI2MWEwODA0OWNhZH50cGx2LWszdTFmYnBmY3Atem9vbS0xLmltYWdl?x-oss-process=image/format,png)
显然，这次的指令比上次要复杂一些，在main方法的第三行指令中可以看到实例化了一个StringBuilder。并在第12行调用了StringBuiler的append方法。到这里一切都清晰明了了，原来编译器帮我们做了优化：”+“连接符其实是在虚拟机内部通过StringBuilder来实现的。
总之，在Java中使用”+“连接符不管是拼接常量还是拼接变量，其性能明显都要优于String的concat方法。好了，到这里不知道你是否疑惑，为什么编译器会使用StringBuilder进行优化，StringBuilder相比String又有何优势？这里先留个疑问吧！
除了字符串裁剪与拼接，String还提供了很多的操作字符串的方法，涉及到修改字符串的方法一般都是通过实例化一个新的对象来实现的。源码很容易读懂，所以在这里就不再分析。

## 四、总结
本篇文章我们深入分析了String字符串的内存分配、intern()方法，以及String类的结构及特性。关于这块知识，网上的文章鱼龙混杂，甚至众说纷纭。笔者也是参考了大量的文章并结合自己的理解来做的分析。但是，避免不了的可能会出现理解偏差的问题，如果有，希望大家多多讨论给予指正。
同时，文章中多次提到StringBuilder,但限于文章篇幅，没能给出关于其详细分析。不过不用担心，我会在下一篇文章中再做探讨。
不管怎样，相信大家看完这篇文章后一定
对String有了更加深入的认识，尤其是了解String类的一些裁剪及拼接中可能造成的性能问题，在今后的开发中应该尽量避免。

## 参考&推荐阅读
[深入解析String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)

[Java String 对象，你真的了解了吗？](https://juejin.im/post/6844903951351939086#comment)
