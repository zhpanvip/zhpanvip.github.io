---
layout: article
title: Java基础面试题
date: 2018-09-07 14:27:03
tags:
- 面试
---


## 一、面向对象与Java基础知识
###  1.Java中“==” 和 equals 有什么
”==“ 是 Java 中一种操作符，它有两种比较方式

 - 1.对于基本数据类型来说，" ==" 判断的是两边的值是否相等
 - 2.对于引用类型来说，  "=="判断的是两边的引用是否相等，也就是判断两个对象是否指向了同一块内存区域。

而equals方法 是 Object 类定义的一个方法。源码如下：

```
    public boolean equals(Object obj) {
        return (this == obj);
    }
```
可以看到，在Object中其实equals方法与使用”==“是等价的。但是Java的开发人员赋予equals方法的意义是比较两个对象的值是否相等。因此就需要我们在有需要的时候去根据自身需要来重写equals方法。重写equals方法的同时，需要一并重写hashCode方法。

### 2.为什么重写 equals 方法必须重写 hashcode 方法
个人认为重写equals方法并不一定就要重写hashcode。需要根据情况来讨论。
如果这个对象是用到Hash表实现的数据结构中的话，那么一定要重写hashcode方法。比如这个对象可能会被存储到HashMap、HashSet这样的数据结构中。
但是如果重写equals方法仅仅是为了比较两个对象是的值否相等或者该数据仅仅是存储在List这样的数据结构中，则是否重写hashcode方法其实并无影响。
但通常情况下，一个类的使用场景较多，如果开始这个类没有存储到Hash表的数据结构中的需求，但是并不能确定以后是否会出现这样的情况，所以规定重写equals方法必须重写hashCode方法。

那equals方法与hashcode有什么关系呢？为什么规定重写equals方法就必须重写hashCode呢？这就需要从Hash表实现的数据结构说起。以HashMap为例，HashMap内部是通过数组和链表来对数据进行存储的（使用链地址法来处理Hash冲突）。在HashMap中会通过hashcode来计算Key的hash值，并根据计算的Hash值对该元素进行散列存储。如果碰到hash值重复的情况，则将该元素插入到链表的头部。有相同hash值的元素会在同一个链表中。如下图：

![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/1.jpeg)
而从HashMap中取数据的时候，首会先根据Key的hash值确定元素桶的位置，接着通过Key的equals方法对比获取到查找的元素。那么，如果一个作为HashMap Key的对象只重写了equals方法，但没有重写hashcode，就会造成两个equals相等的Key，有不同的hashcode，进而造成了HashMap的同一个Key出现映射了两个Value的情况，显然这违背了HashMap的初衷，导致数据的错误。因此，规定重写equals方法需要一并重写hashcode方法。


###  3.下面的代码在JVM中生成了几个String对象？JVM是如何对其进行内存分配的？
```
String s1 = new String("abc")
```
对于这一问题应该分情况讨论，因为虚拟机在JDK7之前与之后对于字符串常量池的实现是不一样的。

**（1）JDK7之前**
对于String类型的数据，JVM维护了一个字符串常量池来存放字符串对象。JDK7之前，字符串常量池位于方法区（永久代）。上述代码中如果常量池中还没有”abc"对象，则会首先在字符常量池中生成一个”abc"对象，如果“abc"已经存在于字符串常量池，则不会再次生成。注意这个字符串对象是位于永久代而非堆内存。接着，在执行new的时候在堆内存中再生成一个“abc"对象，并将这个对象的引用赋值给了s1。因此上述代码可能生成两个String对象，也可能生成一个。如下图：
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/string/jdk6-01.png)


而在JDK7之后，由于常量池被移到了堆内存，并且常量池中不再存放字符对象，而是存放字符串的引用。
那么上述代码中则首先会在堆内存中生成一个“abc"对象，并将这个对象的引用会放入常量池。接着，在执行new时，会再次在堆中生成一个”abc"对象，并将这个对象引用赋值给s1。如下图：

![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/string/jdk7-01.png)

因此，无论是在JDK7之前还是之后，这一代码都会生成一个或者两个字符串对象，只是内存的分配会有所区别。

可参考:[Java进阶--深入理解Java中的字符串（一）](https://blog.csdn.net/qq_20521573/article/details/108425352)

### 4.了解String的intern()方法吗？它有什么作用？
当调用intern方法的时候，如果字符串常量池中已经包含了一个等于该String对象的字符串，则直接返回字符串常量池中该字符串的引用。否则，会将该字符串对象包含的字符串添加到常量池，并返回此对象的引用。

详情可参考:[Java进阶--深入理解Java中的字符串（一）](https://blog.csdn.net/qq_20521573/article/details/108425352)


###  5.String、StringBuffer与StringBuilder有区别？
（1）String 类被final修饰，同时内部维护的char[] 数组也是final修饰的，意味着String类是无法被继承和改变的。它内部不存在扩容机制。字符串拼接，截取等操作都会生成一个新的字符串对象。因此，频繁操作String对象效率会比较低。

（2）StringBuilder继承自AbstractStringBuilder， 类内部维护了一个可变长度char[] (即没有final修饰)， 初始化数组容量为16，当char[]的空间不足时会启用存在扩容机制。在操作StringBuilder的字符串时会调用System的native方法进行数组的拷贝。因此，不会像String一样重新生成新的字符串对象。
但在调用 toString方法时不会共享StringBuilder对象内部的char[]，而是进行一次char[]的copy操作，使用新生成的char[]数组重新生成的String对象。toString代码如下：

```java
public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
```

另外，StringBuilder是非线程安全的。

（3）StringBuffer同样也继承自AbstractStringBuilder，由于大部分逻辑都是在AbstractStringBuilder中实现的，所以StringBuilder与StringBuilder差别不大。只是StringBuffer重写了相关方法，并加了synchronize关键字，保证了线程安全。另外一点是StringBuffer有一个String类型的toStringCach的缓存，在进行字符串操作的时候都会更新toStringCache的值。在调用toString方法时，返回的String共享了toStringCach。如下代码：

```java
@Override
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        return new String(toStringCache, true);
    }
```

详情参考：[Java进阶--深入理解Java中的字符串（二）](https://blog.csdn.net/qq_20521573/article/details/108521881)

###  6.访问修饰符public,private,protected,以及不写（默认）时的区别？
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/4.png)
类的成员不写访问修饰时默认为default。默认对于同一个包中的其他类相当于公开（public），对于不是同一个包中的其他类相当于私有（private）。受保护（protected）对子类相当于公开，对不是同一包中的没有父子关系的类相当于私有。Java中，外部类的修饰符只能是public或默认，类的成员（包括内部类）的修饰符可以是以上四种。

### 7.final有哪几种用法？每种用法是什么含义？
1）当你在类中定义变量时，在其前面加上final关键字，那便是说，这个变量一旦被初始化便不可改变。
2）将方法声明为final，那就说明你已经知道这个方法提供的功能已经满足你要求，不需要进行扩展，并且也不允许任何从此类继承的类来覆写这个方法，但是继承仍然可以继承这个方法，也就是说可以直接使用。
3）当你将final用于类身上时，你就需要仔细考虑，因为一个final类是无法被任何人继承的，那也就意味着此类在一个继承树中是一个叶子类，并且此类的设计已被认为很完美而不需要进行修改或扩展。

### 8.static 关键的作用
static 是 Java 中非常重要的关键字，static 表示的概念是 静态的，在 Java 中，static 主要用来：

- 修饰变量，static 修饰的变量称为静态变量、也称为类变量，类变量属于类所有，对于不同的类来说，static 变量只有一份，static 修饰的变量位于方法区中；static 修饰的变量能够直接通过 类名.变量名 来进行访问，不用通过实例化类再进行使用。
- 修饰方法，static 修饰的方法被称为静态方法，静态方法能够直接通过 类名.方法名 来使用，在静态方法内部不能使用非静态属性和方法
- static 可以修饰代码块，主要分为两种，一种直接定义在类中，使用 static{}，这种被称为静态代码块，一种是在类中定义静态内部类，使用 static class xxx 来进行定义。
- static 可以用于静态导包，通过使用 import static xxx  来实现，这种方式一般不推荐使用
- static 可以和单例模式一起使用，通过双重检查锁来实现线程安全的单例模式。
-
详情请参考 [static 还能难得住我？](https://mp.weixin.qq.com/s?__biz=MzI0ODk2NDIyMQ==&mid=2247484455&idx=1&sn=582d5d2722dab28a36b6c7bc3f39d3fb&chksm=e999f135deee7823226d4da1e8367168a3d0ec6e66c9a589843233b7e801c416d2e535b383be&token=1154740235&lang=zh_CN#rd)


### 9.内部类可以引用外部类的成员吗？有没有什么限制？

内部类如果不是static修饰的，那么它可以通过"this."访问创建它的外部类对象的所有属性。这是因为非静态内部类默认持有外部类的引用。怎么理解呢？其实是因为编译器在编译非静态内部类的时候给非静态内部类的构造方法加了一个外部类的参数，所以非静态内部类可以通过这个参数访问到外部类程成员和方法。可以通过javap -p来反汇编字节或者通过反射调用非静态内部类的构造方法来验证。
另外，非静态内部类无法直接实例化，而是需要先实例化外部类后再实例化非静态内部类，如果是通过反射调用非静态内部类的构造方法，则必须传入外部类的对象才可成功实例化非静态内部类。


内部类如果是sattic修饰的，则这个内部类和普通的类其实并没有差别，它只可以访问外部类对象的所有static属性。
一般普通类只有public或package的访问修饰，而内部类可以实现static，protected，private等访问修饰。
当从外部类继承的时候，内部类是不会被覆盖的，它们是完全独立的实体，每个都在自己的命名空间内，如果从内部类中明确地继承，就可以覆盖原来内部类的方法。

### 10.int和Integer有什么区别？
Java是一个近乎纯洁的面向对象编程语言，但是为了编程的方便还是引入了基本数据类型，但是为了能够将这些基本数据类型当成对象操作，Java为每一个基本数据类型都引入了对应的包装类型（wrapper class），int的包装类就是Integer，从Java 5开始引入了自动装箱/拆箱机制，使得二者可以相互转换。
Java 为每个原始类型提供了包装类型：
- 原始类型: boolean，char，byte，short，int，long，float，double
- 包装类型：Boolean，Character，Byte，Short，Integer，Long，Float，Double

```
class AutoUnboxingTest {

    public static void main(String[] args) {
        Integer a = new Integer(3);
        Integer b = 3;                  // 将3自动装箱成Integer类型
        int c = 3;
        System.out.println(a == b);     // false 两个引用没有引用同一对象
        System.out.println(a == c);     // true a自动拆箱成int类型再和c比较
    }
}
```
最近还遇到一个面试题，也是和自动装箱和拆箱有点关系的，代码如下所示：

```
public class Test03 {

    public static void main(String[] args) {
        Integer f1 = 100, f2 = 100, f3 = 150, f4 = 150;

        System.out.println(f1 == f2);
        System.out.println(f3 == f4);
    }
}
```
如果不明就里很容易认为两个输出要么都是true要么都是false。首先需要注意的是f1、f2、f3、f4四个变量都是Integer对象引用，所以下面的==运算比较的不是值而是引用。装箱的本质是什么呢？当我们给一个Integer对象赋一个int值的时候，会调用Integer类的静态方法valueOf，如果看看valueOf的源代码就知道发生了什么。

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
IntegerCache是Integer的内部类，其代码如下所示：

```java
/**
     * Cache to support the object identity semantics of autoboxing for values between
     * -128 and 127 (inclusive) as required by JLS.
     *
     * The cache is initialized on first usage.  The size of the cache
     * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
     * During VM initialization, java.lang.Integer.IntegerCache.high property
     * may be set and saved in the private system properties in the
     * sun.misc.VM class.
     */

    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```
简单的说，如果整型字面量的值在-128到127之间，那么不会new新的Integer对象，而是直接引用常量池中的Integer对象，所以上面的面试题中f1= =f2的结果是true，而f3==f4的结果是false。

### 11.Java 面向对象的特征有哪些方面？
- 抽象：抽象是将一类对象的共同特征总结出来构造类的过程，包括数据抽象和行为抽象两方面。抽象只关注对象有哪些属性和行为，并不关注这些行为的细节是什么。
- 继承：继承是从已有类得到继承信息创建新类的过程。提供继承信息的类被称为父类（超类、基类）；得到继承信息的类被称为子类（派生类）。继承让变化中的软件系统有了一定的延续性，同时继承也是封装程序中可变因素的重要手段（如果不能理解请阅读阎宏博士的《Java与模式》或《设计模式精解》中关于桥梁模式的部分）。
- 封装：通常认为封装是把数据和操作数据的方法绑定起来，对数据的访问只能通过已定义的接口。面向对象的本质就是将现实世界描绘成一系列完全自治、封闭的对象。我们在类中编写的方法就是对实现细节的一种封装；我们编写一个类就是对数据和数据操作的封装。可以说，封装就是隐藏一切可隐藏的东西，只向外界提供最简单的编程接口（可以想想普通洗衣机和全自动洗衣机的差别，明显全自动洗衣机封装更好因此操作起来更简单；我们现在使用的智能手机也是封装得足够好的，因为几个按键就搞定了所有的事情）。
- 多态性：多态性是指允许不同子类型的对象对同一消息作出不同的响应。简单的说就是用同样的对象引用调用同样的方法但是做了不同的事情。多态性分为编译时的多态性和运行时的多态性。如果将对象的方法视为对象向外界提供的服务，那么运行时的多态性可以解释为：当A系统访问B系统提供的服务时，B系统有多种提供服务的方式，但一切对A系统来说都是透明的（就像电动剃须刀是A系统，它的供电系统是B系统，B系统可以使用电池供电或者用交流电，甚至还有可能是太阳能，A系统只会通过B类对象调用供电的方法，但并不知道供电系统的底层实现是什么，究竟通过何种方式获得了动力）。方法重载（overload）实现的是编译时的多态性（也称为前绑定），而方法重写（override）实现的是运行时的多态性（也称为后绑定）。运行时的多态是面向对象最精髓的东西，要实现多态需要做两件事：1). 方法重写（子类继承父类并重写父类中已有的或抽象的方法）；2). 对象造型（用父类型引用引用子类型对象，这样同样的引用调用同样的方法就会根据子类对象的不同而表现出不同的行为）。



### 12.简述Java反射机制，反射的作用和应用？
在Java中，所有已经被虚拟机的类加载器加载过的类（称为T）都会在虚拟机中生成一个唯一的与T类所对应的Class<T>对象。在程序运行时，通过这个Class<T>对象，我们可以实例化出来一个T对象；可以通过Class<T>对象访问T对象中的任意成员变量，调用T对象中的任意方法，甚至可以对T对象中的成员变量进行修改。我们将这一系列操作称为Java的反射机制。

到这里我们发现，其实Java的反射也没有那么神秘了。说白了就是通过Class对象来操控我们的对象罢了。因此，接下来我们想要弄懂反射只需要来详细的认识一下Class这个类给我们提供的API即可。

详情请参考：[Java进阶--深入理解Java的反射机制
](https://blog.csdn.net/qq_20521573/article/details/111751156)

### 13.Java泛型是什么？泛型的类型擦除是怎么回事？
详情参考:[Java进阶--Java中的泛型详解](https://blog.csdn.net/qq_20521573/article/details/112712619)

### 14.什么是序列化以及用途，什么时候使用序列化？
序列化是指把对象转换为字节序列的过程称为对象的序列化；而反序列化是指把字节序列恢复为对象的过程称为对象的反序列化。
对象的序列化主要有两种用途：
1）把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；
2）在网络上传送对象的字节序列。
什么时候使用序列化：
1）对象序列化可以实现分布式对象。主要应用例如：RMI要利用对象序列化运行远程主机上的服务，就像在本地机上运行对象时一样。
2）java对象序列化不仅保留一个对象的数据，而且递归保存对象引用的每个对象的数据。可以将整个对象层次写入字节流中，可以保存在文件中或在网络连接上传递。利用对象序列化可以进行对象的"深复制"，即复制对象本身及引用的对象本身。序列化一个对象可能得到整个对象序列。

### 15.如何实现对象克隆？
有两种方式：
1). 实现Cloneable接口并重写Object类中的clone()方法；
2). 实现Serializable接口，通过对象的序列化和反序列化实现克隆，可以实现真正的深度克隆，代码如下。

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

public class MyUtil {

    private MyUtil() {
        throw new AssertionError();
    }

    public static <T> T clone(T obj) throws Exception {
        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bout);
        oos.writeObject(obj);

        ByteArrayInputStream bin = new ByteArrayInputStream(bout.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bin);
        return (T) ois.readObject();

        // 说明：调用ByteArrayInputStream或ByteArrayOutputStream对象的close方法没有任何意义
        // 这两个基于内存的流只要垃圾回收器清理对象就能够释放资源，这一点不同于对外部资源（如文件流）的释放
    }
}
```
下面是测试代码：

```java
import java.io.Serializable;

/**
 * 人类
 * @author 骆昊
 *
 */
class Person implements Serializable {
    private static final long serialVersionUID = -9102017020286042305L;

    private String name;    // 姓名
    private int age;        // 年龄
    private Car car;        // 座驾

    public Person(String name, int age, Car car) {
        this.name = name;
        this.age = age;
        this.car = car;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public Car getCar() {
        return car;
    }

    public void setCar(Car car) {
        this.car = car;
    }

    @Override
    public String toString() {
        return "Person [name=" + name + ", age=" + age + ", car=" + car + "]";
    }

}
```

```java
/**
 * 小汽车类
 * @author 骆昊
 *
 */
class Car implements Serializable {
    private static final long serialVersionUID = -5713945027627603702L;

    private String brand;       // 品牌
    private int maxSpeed;       // 最高时速

    public Car(String brand, int maxSpeed) {
        this.brand = brand;
        this.maxSpeed = maxSpeed;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }

    public int getMaxSpeed() {
        return maxSpeed;
    }

    public void setMaxSpeed(int maxSpeed) {
        this.maxSpeed = maxSpeed;
    }

    @Override
    public String toString() {
        return "Car [brand=" + brand + ", maxSpeed=" + maxSpeed + "]";
    }

}
```

```java
class CloneTest {

    public static void main(String[] args) {
        try {
            Person p1 = new Person("Hao LUO", 33, new Car("Benz", 300));
            Person p2 = MyUtil.clone(p1);   // 深度克隆
            p2.getCar().setBrand("BYD");
            // 修改克隆的Person对象p2关联的汽车对象的品牌属性
            // 原来的Person对象p1关联的汽车不会受到任何影响
            // 因为在克隆Person对象时其关联的汽车对象也被克隆了
            System.out.println(p1);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
注意：基于序列化和反序列化实现的克隆不仅仅是深度克隆，更重要的是通过泛型限定，可以检查出要克隆的对象是否支持序列化，这项检查是编译器完成的，不是在运行时抛出异常，这种是方案明显优于使用Object类的clone方法克隆对象。让问题在编译的时候暴露出来总是优于把问题留到运行时。

**7.Comparator接口简单描述一下？**

java.util.Comparator是比较器接口，如果我们需要控制某个类的次序并且该类本身不支持排序，那么就可以建立一个类比较器来进行排序，实现方式很简单只需要实现java.util.Comparator接口。
java.util.Comparator接口只包括两个函数，它的源码如下：

```
package java.util;

public interface Comparator<T> {

    int compare(T o1, T o2);

    boolean equals(Object obj);

}
```

1）若一个类要实现java.util.Comparator接口：它一定要实现int compare(T o1, T o2) 函数，而另一个可以不实现boolean equals(Object obj) 函数
2）int compare(T o1, T o2) 是比较o1和o2的大小，如果返回值为负数意味着o1比o2小，否则返回为零意味着o1等于o2，返回为正数意味着o1大于o2
### 16.Comparable 接口简单描述一下？
Comparable 是排序接口。若一个类实现了Comparable接口，就意味着“该类支持排序”。  即然实现Comparable接口的类支持排序，假设现在存在“实现Comparable接口的类的对象的List列表(或数组)”，则该List列表(或数组)可以通过 Collections.sort（或 Arrays.sort）进行排序。

此外，“实现Comparable接口的类的对象”可以用作“有序映射(如TreeMap)”中的键或“有序集合(TreeSet)”中的元素，而不需要指定比较器。
Comparable 接口仅仅只包括一个函数，它的定义如下：

```
package java.lang;
import java.util.*;

public interface Comparable<T> {
    public int compareTo(T o);
}
```
假设我们通过 x.compareTo(y) 来“比较x和y的大小”。若返回“负数”，意味着“x比y小”；返回“零”，意味着“x等于y”；返回“正数”，意味着“x大于y”。
### 17.comparable 和 comparator的区别
　Comparable是排序接口，若一个类实现了Comparable接口，就意味着“该类支持排序”。而Comparator是比较器，我们若需要控制某个类的次序，可以建立一个“该类的比较器”来进行排序。

　　Comparable相当于“内部比较器”，而Comparator相当于“外部比较器”。

　　两种方法各有优劣， 用Comparable 简单， 只要实现Comparable 接口的对象直接就成为一个可以比较的对象，但是需要修改源代码。 用Comparator 的好处是不需要修改源代码， 而是另外实现一个比较器， 当某个自定义的对象需要作比较的时候，把比较器和对象一起传递过去就可以比大小了， 并且在Comparator 里面用户可以自己实现复杂的可以通用的逻辑，使其可以匹配一些比较简单的对象，那样就可以节省很多重复劳动了。

二、Java集合框架
----
![这里写图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/5.png)


![这里写图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/6.png)

### 1.HashMap的工作原理
HashMap是基于哈希表实现的，用于存储key-value的键值对，并允许使用null值和null键。由于是基于Hash表实现的，因此HashMap具有较高的查询效率，理想情况下HashMap的查找时间复杂度可达到O(1)。

**(1)HashMap的存储结构**
HashMap实际上是一个“链表散列”的数据结构，即数组和链表的结合体（链地址法处理冲突）。HashMap内部封装了一个包含key和value的Entry类，用于存储键值对。在put操作中会根据key的hashcode计算在哈希表中的存储位置，并将Entry存入该位置。由于存在Hash冲突的情况，HashMap采用了链地址法来处理Hash冲突。即使用链表的形式将相同哈希值的元素连起来。如下图所示：
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/1.jpeg)
取元素时在HashMap的get方法中计算key的哈希值，并定位到元素所在桶的位置，接着使用equals方法查找到目标元素。

**（2) HashMap的扩容与ReHash**
由于HashMap存的长度是确定的，可以初始化时候指定长度或者默认长度16。随着HashMap中插入的元素越来越多，发生哈希冲突的概率会越来越大，相应的查找的效率就会越来越低。这意味着影响哈希表性能的因素除了哈希函数与处理冲突的方法之外，还与哈希表的装填因子大小有关。

> 我们将哈希表中元素数与哈希表长度的比值称为**装填因子**。装填因子 **α= $\frac{哈希表中元素数}{哈希表长度}$**

在HashMap中，装填因子的阈值为0.75，当装填因子大于0.75时则会出发HashMap的扩容机制。这里我们应该知道，扩容并不是在原数组基础上扩大容量，而是需要申请一个长度为原来2倍的新数组。因此，扩容之后就需要将原来的数据从旧数组中重新散列存放到扩容后的新数组。这个过程我们称之为Rehash。

Rehash的操作将会重新散列扩容前已经存储的数据，这一操作涉及大量的元素移动，是一个非常消耗性能的操作。因此，在开发中我们应该尽量避免Rehash的出现。比如，可以预估元素的个数，事先指定哈希表的长度，这样可以有效减少Rehash。

**（3）JDK1.8中对HashMap的优化**
JDK1.7 中，HashMap 采用位桶 + 链表的实现，即使用链表来处理冲突，同一 hash 值的链表都存储在一个数组中。但是当位于一个桶中的元素较多，即 hash 值相等的元素较多时，通过 key 值依次查找的效率较低。如下图所示的元素查找效率直接降低为了链表的时间复杂度o(n)

![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/hashtable/8.png)
为了优化这一问题，JDK 1.8 在底层结构方面做了一些改变，当每个桶中元素大于 8 的时候，会转变为红黑树，而红黑树的查找效率为o(logn)，高于链表的查找效率。
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/hashtable/10.png)
详情参考：
[面试官：哈希表都不知道，你是怎么看懂HashMap的？](https://blog.csdn.net/qq_20521573/article/details/108701471)
[HashMap原理深入理解](https://blog.csdn.net/visant/article/details/80045154)
[HashMap的工作原理](https://blog.csdn.net/ty564457881/article/details/78206049)

### 2.为什么HashMap在多线程并发存在死循环的问题，JDK1.8中做了哪些优化？
详情参考
[《我们一起进大厂》系列-HashMap](https://juejin.cn/post/6844904017269637128)
 [老生常谈，HashMap的死循环](https://juejin.cn/post/6844903554264596487)
[HashMap为何从头插入改为尾插入](https://juejin.cn/post/6844903682664824845)

### 3.Hashtable与HashMap有什么区别？
**1.安全性**
Hashtable是线程安全，HashMap是非线程安全。HashMap的性能会高于Hashtable，我们平时使用时若无特殊需求建议使用HashMap，在多线程环境下若使用HashMap需要使用Collections.synchronizedMap()方法来获取一个线程安全的集合（Collections.synchronizedMap()实现原理是Collections定义了一个SynchronizedMap的内部类，这个类实现了Map接口，在调用方法时使用synchronized来保证线程同步
**2.是否可以使用null作为key**
HashMap可以使用null作为key，不过建议还是尽量避免这样使用。HashMap以null作为key时，总是存储在table数组的第一个节点上。而Hashtable则不允许null作为key
**3.继承了什么，实现了什么**
HashMap继承了AbstractMap，HashTable继承Dictionary抽象类，两者均实现Map接口
**4.默认容量及如何扩容**
HashMap的初始容量为16，Hashtable初始容量为11，两者的填充因子默认都是0.75。HashMap扩容时是当前容量翻倍即:capacity  2，Hashtable扩容时是容量翻倍+1即:capacity  (2+1)
**5.底层实现**
HashMap和Hashtable的底层实现都是数组+链表结构实现
**6.计算hash的方法不同**
Hashtable计算hash是直接使用key的hashcode对table数组的长度直接进行取模，HashMap计算hash对key的hashcode进行了二次hash，以获得更好的散列值，然后对table数组长度取模

### 4.了解ConcurrentHashMap吗？它是怎么实现的?
在多线程环境下，使用HashMap进行put操作时存在丢失数据的情况，为了避免这种bug的隐患，强烈建议使用ConcurrentHashMap代替HashMap。

HashTable是一个线程安全的类，它使用synchronized来锁住整张Hash表来实现线程安全，即每次锁住整张表让线程独占，相当于所有线程进行读写时都去竞争一把锁，导致效率非常低下。ConcurrentHashMap可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，允许多个修改操作并发进行，其关键在于使用了锁分段技术。它使用了多个锁来控制对hash表的不同部分进行的修改。对于JDK1.7版本的实现, ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的Hashtable，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。JDK1.8的实现降低锁的粒度，JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）。

**实现原理**
对于JDK1.7版本的实现，ConcurrentHashMap 为了提高本身的并发能力，在内部采用了一个叫做 Segment 的结构，一个 Segment 其实就是一个类 Hash Table 的结构，Segment 内部维护了一个链表数组，我们用下面这一幅图来看下 ConcurrentHashMap 的内部结构,从下面的结构我们可以了解到，ConcurrentHashMap 定位一个元素的过程需要进行两次Hash操作，第一次 Hash 定位到 Segment，第二次 Hash 定位到元素所在的链表的头部，因此，这一种结构的带来的副作用是 Hash 的过程要比普通的 HashMap 要长，但是带来的好处是写操作的时候可以只对元素所在的 Segment 进行操作即可，不会影响到其他的 Segment，这样，在最理想的情况下，ConcurrentHashMap 可以最高同时支持 Segment 数量大小的写操作（刚好这些写操作都非常平均地分布在所有的 Segment上），所以，通过这一种结构，ConcurrentHashMap 的并发能力可以大大的提高。我们用下面这一幅图来看下ConcurrentHashMap的内部结构详情图，如下:
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/10.jpeg)
为什么要用二次hash，主要原因是为了构造分离锁，使得对于map的修改不会锁住整个容器，提高并发能力。当然，没有一种东西是绝对完美的，二次hash带来的问题是整个hash的过程比hashmap单次hash要长，所以，如果不是并发情形，不要使concurrentHashmap。

JAVA7之前ConcurrentHashMap主要采用锁机制，在对某个Segment进行操作时，将该Segment锁定，不允许对其进行非查询操作，而在JAVA8之后采用CAS无锁算法，这种乐观操作在完成前进行判断，如果符合预期结果才给予执行，对并发操作提供良好的优化.

参考：[一文读懂Java ConcurrentHashMap原理与实现](https://zhuanlan.zhihu.com/p/104515829)

### 5.可以使用CocurrentHashMap来代替Hashtable吗？
我们知道Hashtable是synchronized的，但是ConcurrentHashMap同步性能更好，因为它仅仅根据同步级别对map的一部分进行上锁。ConcurrentHashMap当然可以代替HashTable，但是HashTable提供更强的线程安全性。它们都可以用于多线程的环境，但是当Hashtable的大小增加到一定的时候，性能会急剧下降，因为迭代时需要被锁定很长的时间。因为ConcurrentHashMap引入了分割(segmentation)，不论它变得多么大，仅仅需要锁定map的某个部分，而其它的线程不需要等到迭代完成才能访问map。简而言之，在迭代的过程中，ConcurrentHashMap仅仅锁定map的某个部分，而Hashtable则会锁定整个map。
### 6.ConcurrentHashMap有什么缺陷吗？
ConcurrentHashMap 是设计为非阻塞的。在更新时会局部锁住某部分数据，但不会把整个表都锁住。同步读取操作则是完全非阻塞的。好处是在保证合理的同步前提下，效率很高。坏处是严格来说读取操作不能保证反映最近的更新。例如线程A调用putAll写入大量数据，期间线程B调用get，则只能get到目前为止已经顺利插入的部分数据。

### 7.ConcurrentHashMap在JDK 7和8之间的区别

JDK1.8的实现降低锁的粒度，JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）
JDK1.8版本的数据结构变得更加简单，使得操作也更加清晰流畅，因为已经使用synchronized来进行同步，所以不需要分段锁的概念，也就不需要Segment这种数据结构了，由于粒度的降低，实现的复杂度也增加了
JDK1.8使用红黑树来优化链表，基于长度很长的链表的遍历是一个很漫长的过程，而红黑树的遍历效率是很快的，代替一定阈值的链表，这样形成一个最佳拍档
### 9.Java中HashMap和HashTable的区别？
**相同点**
HashMap 和 HashTable 都是基于哈希表实现的，其内部每个元素都是 key-value 键值对，HashMap 和 HashTable 都实现了 Map、Cloneable、Serializable 接口。

**不同点**
- **（1）继承的父类不同**
HashMap是继承自AbstractMap类，而HashTable是继承自Dictionary类。不过它们都同时实现了map、Cloneable（可复制）、Serializable（可序列化）这三个接口
- **（2） 线程安全性不同**
Hashtable是线程安全的，它的每个方法中都加入了Synchronize方法。在多线程并发的环境下，可以直接使用Hashtable，不需要自己为它的方法实现同步
HashMap不是线程安全的，在多线程并发的环境下，可能会产生死锁等问题。具体的原因在下一篇文章中会详细进行分析。使用HashMap时就必须要自己增加同步处理，虽然HashMap不是线程安全的，但是它的效率会比Hashtable要好很多。
- **（3）空值不同**
- HashMap 允许空的 key 和 value 值，HashTable 不允许空的 key 和 value 值。HashMap 会把 Null key 当做普通的 key 对待。不允许 null key 重复。
- **（4） 初始容量大小和每次扩充容量大小的不同**
Hashtable默认的初始大小为11，之后每次扩充，容量变为原来的2n+1。HashMap默认的初始化大小为16。之后每次扩充，容量变为原来的2倍。
### 10.HashMap 和 HashSet 的区别
HashSet 继承于 AbstractSet 接口，实现了 Set、Cloneable,、java.io.Serializable 接口。HashSet 不允许集合中出现重复的值。HashSet 其实就是用HashMap来实现的，所有对 HashSet 的操作其实就是对 HashMap 的操作。所以 HashSet 也不保证集合的顺序，也不是线程安全的容器。

### 11.请说出 ArrayList和LinkedList的区别？
1)ArrayList和LinkedList可想从名字分析，它们一个是Array(动态数组)的数据结构，一个是Link(链表)的数据结构，此外，它们两个都是对List接口的实现。
前者是数组队列，相当于动态数组；后者为双向链表结构，也可当作堆栈、队列、双端队列
2)当随机访问List时（get和set操作），ArrayList比LinkedList的效率更高，因为LinkedList是线性的数据存储方式，所以需要移动指针从前往后依次查找。
3)当对数据进行增加和删除的操作时(add和remove操作)，LinkedList比ArrayList的效率更高，因为ArrayList是数组，所以在其中进行增删操作时，会对操作点之后所有数据的下标索引造成影响，需要进行数据的移动。
4)从利用效率来看，ArrayList自由性较低，因为它需要手动的设置固定大小的容量，但是它的使用比较方便，只需要创建，然后添加数据，通过调用下标进行使用；而LinkedList自由性较高，能够动态的随数据量的变化而变化，但是它不便于使用。
5)ArrayList主要控件开销在于需要在lList列表预留一定空间；而LinkList主要控件开销在于需要存储结点信息以及结点指针信息。

**12.Java 中 Set 与 List 有什么不同?**
List,Set都是继承自Collection接口
List特点：元素有放入顺序，元素可重复 ，
Set特点：元素无放入顺序，元素不可重复（注意：元素虽然无放入顺序，但是元素在set中的位置是有该元素的HashCode决定的，其位置其实是固定的）
List接口有三个实现类：LinkedList，ArrayList，Vector ，
Set接口有两个实现类：HashSet(底层由HashMap实现)，LinkedHashSet

**13.请说出 ArrayList和Vector的区别？**
1）不同的在于序列化方面：ArrayList比Vector安全
在ArrayList集合中：
//采用elementData数组来存储集合元素
private transient Object[] elementData;
在Vector集中中：
//采用elementData数组来保存集合元素
private Object[] elementData;
从源码可以看出，ArrayList提供的writeObject和readObject方法来实现定制序列化，而Vector只是提供了writeObject方法，并没有完全实现定制序列化。
2）不同点在于Vector是线性安全的，ArrayList是非线性安全的
ArrayList和Vector的绝大部分方法都是一样的，甚至连方法名都一样,只是Vector的方法大都添加关键之synchronized修饰。
在add方法中，Vector电泳的是insertElementAt(element，index)；
public synchronized void isnertElementAt(E obj,int index);
将 ArrayList中的add（int index，E element）方法和Vector的isnertElementAt（E Obj，int index）方法进行对比，可以发现vectorde insertElementAt(E obj,int index)方法只是多了synchronized修饰。
3）扩容上区别
ArrayList集合和Vector集合底层都是数组实现的，在数组容量不足的时候采取的扩容机制不同。
ArrayList集合容量不足，采取在原有容量基础上扩充为原来的1.5倍。
而Vector则多了一个选择：当capacityIncrement实例变量大于0时，扩充为原有容量加上capacityIncrement的容量值。否则采取在原有容量基础上扩充为原来的1.5倍。

**14.说出ArrayList,Vector, LinkedList的存储性能和特性**
ArrayList 和Vector都是使用数组方式存储数据，此数组元素数大于实际存储的数据以便增加和插入元素，它们都允许直接按序号索 引元素，但是插入元素要涉及数组元素移动等内存操作，所以索引数据快而插入数据慢，Vector由于使用了synchronized方法（线程 安全），通常性能上较ArrayList差，而LinkedList使用双向链表实现存储，按序号索引数据需要进行前向或后向遍历，但是插入数 据时只需要记录本项的前后项即可，所以插入速度较快。

### 15.Collection 和 Collections的区别？**
Collection是集合类的上级接口，继承与他的接口主要有Set 和List。
Collections是针对集合类的一个帮助类，他提供一系列静态方法实现对各种集合的搜索、排序、线程安全化等操作。

## 三、JVM
### JVM的内存分配
[深入JVM--Java运行时内存区域详解](https://zhpanvip.gitee.io/2020/09/04/26.JVM%20memory/)

### Java的垃圾回收机制
[深入JVM--垃圾回收机制全面解析 |](https://zhpanvip.gitee.io/2020/09/19/29.Java%20GC/)

### JVM类加载的过程

[深入JVM--探索Java虚拟机的类加载机制 |](https://zhpanvip.gitee.io/2020/12/25/33.jvm-class-load/)


## 四、多线程与并发

### 谈一谈对volatile关键字的理解
[面试官没想到一个Volatile，我都能跟他扯半小时](https://juejin.cn/post/6844904149536997384)

### 谈一谈syncronized的理解，底层是如何实现的？
[死磕synchronized底层实现](https://juejin.cn/post/6844904196676780040)

### 请谈谈进程和线程有什么区别？
1）首先是定义
进程：是执行中一段程序，即一旦程序被载入到内存中并准备执行，它就是一个进程。进程是表示资源分配的的基本概念，又是调度运行的基本单位，是系统中的并发执行的单位。
线程：线程是操作系统能够进行运算调度的最小单位，它被包含在进程之中，是进程中的实际运作单位。
2）一个线程只能属于一个进程，但是一个进程可以拥有多个线程。多线程处理就是允许一个进程中在同一时刻执行多个任务。
3）线程是一种轻量级的进程，与进程相比，线程给操作系统带来侧创建、维护、和管理的负担要轻，意味着线程的代价或开销比较小。
4）线程没有地址空间，线程包含在进程的地址空间中。线程上下文只包含一个堆栈、一个寄存器、一个优先权，线程文本包含在他的进程 的文本片段中，进程拥有的所有资源都属于线程。所有的线程共享进程的内存和资源。 同一进程中的多个线程共享代码段(代码和常量)，数据段(全局变量和静态变量)，扩展段(堆存储)。但是每个线程拥有自己的栈段， 寄存器的内容，栈段又叫运行时段，用来存放所有局部变量和临时变量。
5）父和子进程使用进程间通信机制，同一进程的线程通过读取和写入数据到进程变量来通信。
6）进程内的任何线程都被看做是同位体，且处于相同的级别。不管是哪个线程创建了哪一个线程，进程内的任何线程都可以销毁、挂起、恢复和更改其它线程的优先权。线程也要对进程施加控制，进程中任何线程都可以通过销毁主线程来销毁进程，销毁主线程将导致该进程的销毁，对主线程的修改可能影响所有的线程。
7）子进程不对任何其他子进程施加控制，进程的线程可以对同一进程的其它线程施加控制。子进程不能对父进程施加控制，进程中所有线程都可以对主线程施加控制。
相同点：
进程和线程都有ID/寄存器组、状态和优先权、信息块，创建后都可更改自己的属性，都可与父进程共享资源、都不鞥直接访问其他无关进程或线程的资源。

### 2. java 中 sleep() 跟 wait() 区别，项目中 Thread sleep 的应用场景
![这里写图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/11.jpeg)
所以sleep()和wait()方法的最大区别是：
sleep()睡眠时，保持对象锁，仍然占有该锁；
而wait()睡眠时，释放对象锁。
但是wait()和sleep()都可以通过interrupt()方法打断线程的暂停状态，从而使线程立刻抛出InterruptedException（但不建议使用该方法）。

### 如何在Java中实现线程？
在语言层面有两种方式。java.lang.Thread 类的实例就是一个线程但是它需要调用java.lang.Runnable接口来执行，由于线程类本身就是调用的Runnable接口所以你可以继承java.lang.Thread 类或者直接调用Runnable接口来重写run()方法实现线程。
可以通过继承Thread类或者调用Runnable接口来实现线程，问题是，那个方法更好呢？什么情况下使用它？这个问题很容易回，Java不支持类的多重继承，但允许实现多个接口。所以如果你要继承其他类，当然是调用Runnable接口好了。

### Thread 类中的start() 和 run() 方法有什么区别?
这个问题经常被问到，但还是能从此区分出面试者对Java线程模型的理解程度。start()方法被用来启动新创建的线程，而且start()内部调用了run()方法，这和直接调用run()方法的效果不一样。当你调用run()方法的时候，只会是在原来的线程中调用，没有新的线程启动，start()方法才会启动新线程。

### 6.Java中notify 和 notifyAll有什么区别？
这又是一个刁钻的问题，因为多线程可以等待单监控锁，Java API 的设计人员提供了一些方法当等待条件改变的时候通知它们，但是这些方法没有完全实现。notify()方法不能唤醒某个具体的线程，所以只有一个线程在等待的时候它才有用武之地。而notifyAll()唤醒所有线程并允许他们争夺锁确保了至少有一个线程能继续运行。
### 7.为什么wait, notify 和 notifyAll这些方法不在thread类里面？
这是个设计相关的问题，它考察的是面试者对现有系统和一些普遍存在但看起来不合理的事物的看法。回答这些问题的时候，你要说明为什么把这些方法放在Object类里是有意义的，还有不把它放在Thread类里的原因。一个很明显的原因是JAVA提供的锁是对象级的而不是线程级的，每个对象都有锁，通过线程获得。如果线程需要等待某些锁那么调用对象中的wait()方法就有意义了。如果wait()方法定义在Thread类中，线程正在等待的是哪个锁就不明显了。简单的说，由于wait，notify和notifyAll都是锁级别的操作，所以把他们定义在Object类中因为锁属于对象。
### 8.什么是ThreadLocal？说一说ThreadLocal的实现原理。
ThreadLocal是一个线程内部的数据存储类，通过它可以在制定的线程中存储数据，数据存储以后，只有在指定线程中才可以获取到存储的数据，对于其它线程来说无法获取到数据。
 ***9.如何避免死锁？***
 　Java多线程中的死锁 死锁是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。这是一个严重的问题，因为死锁会让你的程序挂起无法完成任务，死锁的发生必须满足以下四个条件：
互斥条件：一个资源每次只能被一个进程使用。
请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
不剥夺条件：进程已获得的资源，在末使用完之前，不能强行剥夺。
循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。
避免死锁最简单的方法就是阻止循环等待条件，将系统中所有的资源设置标志位、排序，规定所有的进程申请资源必须以一定的顺序（升序或降序）做操作来避免死锁。这篇教程有代码示例和避免死锁的讨论细节。

 **9.sychronized 锁住方法后方法能被中断吗？**
不能被中断，Lock 可以被中断
**10.synchronized和 Lock 区别**
***1）synchronized和lock的用法区别***
 synchronized：在需要同步的对象中加入此控制，synchronized可以加在方法上，也可以加在特定代码块中，括号中表示需要锁的对象。
 lock：需要显示指定起始位置和终止位置。一般使用ReentrantLock类做为锁，多个线程中必须要使用一个ReentrantLock类做为对象才能保证锁的生效。且在加锁和解锁处需要通过lock()和unlock()显示指出。所以一般会在finally块中写unlock()以防死锁。
 用法区别比较简单，这里不赘述了，如果不懂的可以看看Java基本语法。
 ***2）synchronized和lock性能区别***
 synchronized是托管给JVM执行的，而lock是java写的控制锁的代码。在Java1.5中，synchronize是性能低效的。因为这是一个重量级操作，需要调用操作接口，导致有可能加锁消耗的系统时间比加锁以外的操作还多。相比之下使用Java提供的Lock对象，性能更高一些。但是到了Java1.6，发生了变化。synchronize在语义上很清晰，可以进行很多优化，有适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等。导致在Java1.6上synchronize的性能并不比Lock差。官方也表示，他们也更支持synchronize，在未来的版本中还有优化余地。
 说到这里，还是想提一下这2中机制的具体区别。据我所知，synchronized原始采用的是CPU悲观锁机制，即线程获得的是独占锁。独占锁意味着其他线程只能依靠阻塞来等待线程释放锁。而在CPU转换线程阻塞时会引起线程上下文切换，当有很多线程竞争锁的时候，会引起CPU频繁的上下文切换导致效率很低。
 而Lock用的是乐观锁方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。乐观锁实现的机制就是CAS操作（Compare and Swap）。我们可以进一步研究ReentrantLock的源代码，会发现其中比较重要的获得锁的一个方法是compareAndSetState。这里其实就是调用的CPU提供的特殊指令。
 现代的CPU提供了指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而 compareAndSet() 就用这些代替了锁定。这个算法称作非阻塞算法，意思是一个线程的失败或者挂起不应该影响其他线程的失败或挂起的算法。
 ***3）synchronized和lock用途区别***
 先说第一种情况，ReentrantLock的lock机制有2种，忽略中断锁和响应中断锁，这给我们带来了很大的灵活性。比如：如果A、B2个线程去竞争锁，A线程得到了锁，B线程等待，但是A线程这个时候实在有太多事情要处理，就是一直不返回，B线程可能就会等不及了，想中断自己，不再等待这个锁了，转而处理其他事情。这个时候ReentrantLock就提供了2种机制，第一，B线程中断自己（或者别的线程中断它），但是ReentrantLock不去响应，继续让B线程等待，你再怎么中断，我全当耳边风（synchronized原语就是如此）；第二，B线程中断自己（或者别的线程中断它），ReentrantLock处理了这个中断，并且不再等待这个锁的到来，完全放弃。

**请说出你所知道的线程同步的方法**
wait():使一个线程处于等待状态，并且释放所持有的对象的lock。
sleep():使一个正在运行的线程处于睡眠状态，是一个静态方法，调用此方法要捕捉InterruptedException异常。
notify():唤醒一个处于等待状态的线程，注意的是在调用此方法的时候，并不能确切的唤醒某一个等待状态的线程，而是由JVM确定唤醒哪个线程，而且不是按优先级。
allnotity():唤醒所有处入等待状态的线程，注意并不是给所有唤醒线程一个对象的锁，而是让它们竞争。

**同步和异步有何异同，在什么情况下分别使用他们？举例说明。**
如果数据将在线程间共享。例如正在写的数据以后可能被另一个线程读到，或者正在读的数据可能已经被另一个线程写过了，那么这些数据就是共享数据，必须进行同步存取。
当应用程序在对象上调用了一个需要花费很长时间来执行的方法，并且不希望让程序等待方法的返回时，就应该使用异步编程，在很多情况下采用异步途径往往更有效率。


### java中有几种方法可以实现一个线程？用什么关键字修饰同步方法? stop()和suspend()方法为何不推荐使用？
有两种实现方法，分别是继承Thread类与实现Runnable接口用synchronized关键字修饰同步方法。
反对使用stop()，是因为它不安全。它会解除由线程获取的所有锁定，而且如果对象处于一种不连贯状态，那么其他线程能在那种状 态下检查和修改它们。结果很难检查出真正的问题所在。suspend()方法容易发生死锁。调用suspend()的时候，目标线程会停下来，但却仍然持有在这之前获得的锁定。此时，其他任何线程都不能访问锁定的资源，除非被"挂起"的线程恢复运行。对任何线程来说， 如果它们想恢复目标线程，同时又试图使用任何一个锁定的资源，就会造成死锁。所以不应该使用suspend()，而应在自己的Thread 类中置入一个标志，指出线程应该活动还是挂起。若标志指出线程应该挂起，便用 wait()命其进入等待状态。若标志指出线程应当恢复，则用一个notify()重新启动线程。

## 五、计算机网络
### TCP/IP协议
#### 你能说出TCP/IP协议吗？
OSI 参考模型与 TCP/IP参考模型在分层模块上稍有区别。
OSI参考模型为：应用层、表示层、会话层、传输层、网络层、数据链路层、物理层
TCP/IP参考模型为四层：应用层、传输层、网络层和网络接口层。

OSI 参考模型注重“通信协议必要的功能是什么”，而 TCP/IP 则更强调“在计算机上实现协议应该开发哪种程序”。

![OSI参考模型与TCP/IP参考模型](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/12.jpeg)
TCP/IP 协议是一个协议族，它包含了IP 或 ICMP、TCP 或 UDP、TELNET 或 FTP、以及 HTTP 等一系列协议。

####  TCP协议与UDP协议的区别
TCP/IP 中有两个具有代表性的传输层协议，分别是 TCP 和 UDP。

TCP 是面向连接的、可靠的流协议。流就是指不间断的数据结构，当应用程序采用 TCP 发送消息时，虽然可以保证发送的顺序，但还是犹如没有任何间隔的数据流发送给接收端。TCP 为提供可靠性传输，实行“顺序控制”或“重发控制”机制。此外还具备“流控制（流量控制）”、“拥塞控制”、提高网络利用率等众多功能。
UDP 是不具有可靠性的数据报协议。细微的处理它会交给上层的应用去完成。在 UDP 的情况下，虽然可以确保发送消息的大小，却不能保证消息一定会到达。因此，应用有时会根据自己的需要进行重发处理。
TCP 和 UDP 的优缺点无法简单地、绝对地去做比较：TCP 用于在传输层有必要实现可靠传输的情况；而在一方面，UDP 主要用于那些对高速传输和实时性有较高要求的通信或广播通信。TCP 和 UDP 应该根据应用的目的按需使用。

#### TCP协议的三次握手
TCP 提供面向有连接的通信传输。面向有连接是指在数据通信开始之前先做好两端之间的准备工作。
所谓三次握手是指建立一个 TCP 连接时需要客户端和服务器端总共发送三个包以确认连接的建立。在socket编程中，这一过程由客户端执行connect来触发。
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/13.png)

第一次握手：客户端将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给服务器端，客户端进入SYN_SENT状态，等待服务器端确认。
第二次握手：服务器端收到数据包后由标志位SYN=1知道客户端请求建立连接，服务器端将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给客户端以确认连接请求，服务器端进入SYN_RCVD状态。
第三次握手：客户端收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给服务器端，服务器端检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，客户端和服务器端进入ESTABLISHED状态，完成三次握手，随后客户端与服务器端之间可以开始传输数据了。




#### TCP协议的四次挥手
四次挥手即终止TCP连接，就是指断开一个TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开。在socket编程中，这一过程由客户端或服务端任一方执行close来触发。
由于TCP连接是全双工的，因此，每个方向都必须要单独进行关闭，这一原则是当一方完成数据发送任务后，发送一个FIN来终止这一方向的连接，收到一个FIN只是意味着这一方向上没有数据流动了，即不会再收到数据了，但是在这个TCP连接上仍然能够发送数据，直到这一方向也发送了FIN。首先进行关闭的一方将执行主动关闭，而另一方则执行被动关闭。
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/14.png)


中断连接端可以是客户端，也可以是服务器端。
第一次挥手：客户端发送一个FIN=M，用来关闭客户端到服务器端的数据传送，客户端进入FIN_WAIT_1状态。意思是说"我客户端没有数据要发给你了"，但是如果你服务器端还有数据没有发送完成，则不必急着关闭连接，可以继续发送数据。
第二次挥手：服务器端收到FIN后，先发送ack=M+1，告诉客户端，你的请求我收到了，但是我还没准备好，请继续你等我的消息。这个时候客户端就进入FIN_WAIT_2 状态，继续等待服务器端的FIN报文。
第三次挥手：当服务器端确定数据已发送完成，则向客户端发送FIN=N报文，告诉客户端，好了，我这边数据发完了，准备好关闭连接了。服务器端进入LAST_ACK状态。
第四次挥手：客户端收到FIN=N报文后，就知道可以关闭连接了，但是他还是不相信网络，怕服务器端不知道要关闭，所以发送ack=N+1后进入TIME_WAIT状态，如果Server端没有收到ACK则可以重传。服务器端收到ACK后，就知道可以断开连接了。客户端等待了2MSL后依然没有收到回复，则证明服务器端已正常关闭，那好，我客户端也可以关闭连接了。最终完成了四次握手。


#### IP 协议相关技术
IP 旨在让最终目标主机收到数据包，但是在这一过程中仅仅有 IP 是无法实现通信的。必须还有能够解析主机名称和 MAC 地址的功能，以及数据包在发送过程中异常情况处理的功能。

**1 DNS**
我们平常在访问某个网站时不适用 IP 地址，而是用一串由罗马字和点号组成的字符串。而一般用户在使用 TCP/IP 进行通信时也不使用 IP 地址。能够这样做是因为有了 DNS （Domain Name System）功能的支持。DNS 可以将那串字符串自动转换为具体的 IP 地址。
这种 DNS 不仅适用于 IPv4，还适用于 IPv6。

**2 ARP**
只要确定了 IP 地址，就可以向这个目标地址发送 IP 数据报。然而，在底层数据链路层，进行实际通信时却有必要了解每个 IP 地址所对应的 MAC 地址。
ARP 是一种解决地址问题的协议。以目标 IP 地址为线索，用来定位下一个应该接收数据分包的网络设备对应的 MAC 地址。不过 ARP 只适用于 IPv4，不能用于 IPv6。IPv6 中可以用 ICMPv6 替代 ARP 发送邻居探索消息。
RARP 是将 ARP 反过来，从 MAC 地址定位 IP 地址的一种协议。

**3 ICMP**
ICMP 的主要功能包括，确认 IP 包是否成功送达目标地址，通知在发送过程当中 IP 包被废弃的具体原因，改善网络设置等。
IPv4 中 ICMP 仅作为一个辅助作用支持 IPv4。也就是说，在 IPv4 时期，即使没有 ICMP，仍然可以实现 IP 通信。然而，在 IPv6 中，ICMP 的作用被扩大，如果没有 ICMPv6，IPv6 就无法进行正常通信。

**4 DHCP**
如果逐一为每一台主机设置 IP 地址会是非常繁琐的事情。特别是在移动使用笔记本电脑、只能终端以及平板电脑等设备时，每移动到一个新的地方，都要重新设置 IP 地址。
于是，为了实现自动设置 IP 地址、统一管理 IP 地址分配，就产生了 DHCP（Dynamic Host Configuration Protocol）协议。有了 DHCP，计算机只要连接到网络，就可以进行 TCP/IP 通信。也就是说，DHCP 让即插即用变得可能。
DHCP 不仅在 IPv4 中，在 IPv6 中也可以使用。

[一篇文章带你熟悉 TCP/IP 协议（网络协议篇二）](https://juejin.cn/post/6844903510509633550)

### HTTP与HTTPS
#### Http的get和post的主要有什么区别？**
Form中的get和post方法，在数据传输过程中分别对应了HTTP协议中的GET和POST方法。二者主要区别如下：
1）Get是用来从服务器上获得数据，而Post是用来向服务器上传递数据。
2）Get将表单中数据的按照variable=value的形式，添加到action所指向的URL	后面，并且两者使用“?”连接，而各个变量之间使用“&”连接；Post是将表单	中的数据放在form的数据体中，按照变量和值相对应的方式，传递到action所指向URL。
3）Get是不安全的，因为在传输过程，数据被放在请求的URL中，而如今现有的很多服务器、代理服务器或者用户代理都会将请求URL记录到日志文件中，然后放在某个地方，这样就可能会有一些隐私的信息被第三方看到。另外，用户也可以在浏览器上直接看到提交的数据，一些系统内部消息将会一同显示在用户面前。Post的所有操作对用户来说都是不可见的。
4）Get传输的数据量小，这主要是因为受URL长度限制；而Post可以传输大量的数据，所以在上传文件只能使用Post（当然还有一个原因，将在后面的提到）。
5）Get限制Form表单的数据集的值必须为ASCII字符；而Post支持整个ISO10646字符集。
6）Get是Form的默认方法。

#### HTTPS的实现原理
HTTPS（Hypertext Transfer Protocol Secure）是一种通过计算机网络进行安全通信的传输协议。HTTPS经由HTTP进行通信，但利用TLS来加密数据包。HTTPS开发的主要目的，是提供对网站服务器的身份认证，保护交换数据的隐私与完整性。
TLS是传输层加密协议，前身是SSL协议。由网景公司于1995年发布。后改名为TLS。常用的 TLS 协议版本有：TLS1.2, TLS1.1, TLS1.0 和 SSL3.0。其中 SSL3.0 由于 POODLE 攻击已经被证明不安全。TLS1.0 也存在部分安全漏洞，比如 RC4 和 BEAST 攻击。

由于HTTP协议采用明文传输，我们可以通过抓包很轻松的获取到HTTP所传输的数据。因此，采用HTTP协议是不安全的。这才催生了HTTPS的诞生。HTTPS相对HTTP提供了更安全的数据传输保障。主要体现在三个方面：

1， 内容加密。客户端到服务器的内容都是以加密形式传输，中间者无法直接查看明文内容。
2， 身份认证。通过校验保证客户端访问的是自己的服务器。
3， 数据完整性。防止内容被第三方冒充或者篡改。

其实为了提高安全性和效率HTTPS结合了对称和非对称两种加密方式。即客户端使用对称加密生成密钥（key）对传输数据进行加密，然后使用非对称加密的公钥再对key进行加密。因此网络上传输的数据是被key加密的密文和用公钥加密后的密文key，因此即使被黑客截取，由于没有私钥，无法获取到明文key，便无法获取到明文数据。所以HTTPS的加密方式是安全的。

**对称加密**
对称加密，顾名思义就是加密和解密都是使用同一个密钥，常见的对称加密算法有 DES、3DES 和 AES 等，其优缺点如下：
优点：算法公开、计算量小、加密速度快、加密效率高，适合加密比较大的数据。
缺点：交易双方需要使用相同的密钥，也就无法避免密钥的传输，而密钥在传输过程中无法保证不被截获，因此对称加密的安全性得不到保证。
每对用户每次使用对称加密算法时，都需要使用其他人不知道的惟一密钥，这会使得发收信双方所拥有的钥匙数量急剧增长，密钥管理成为双方的负担。对称加密算法在分布式网络系统上使用较为困难，主要是因为密钥管理困难，使用成本较高
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/15.png)
**非对称加密**
非对称加密，顾名思义，就是加密和解密需要使用两个不同的密钥：公钥（public key）和私钥（private key）。公钥与私钥是一对，如果用公钥对数据进行加密，只有用对应的私钥才能解密；如果用私钥对数据进行加密，那么只有用对应的公钥才能解密。非对称加密算法实现机密信息交换的基本过程是：甲方生成一对密钥并将其中的一把作为公钥对外公开；得到该公钥的乙方使用公钥对机密信息进行加密后再发送给甲方；甲方再用自己保存的私钥对加密后的信息进行解密。如果对公钥和私钥不太理解，可以想象成一把钥匙和一个锁头，只是全世界只有你一个人有这把钥匙，你可以把锁头给别人，别人可以用这个锁把重要的东西锁起来，然后发给你，因为只有你一个人有这把钥匙，所以只有你才能看到被这把锁锁起来的东西。常用的非对称加密算法是 RSA 算法，想详细了解的同学点这里：RSA 算法详解一、RSA 算法详解二，其优缺点如下：

优点：算法公开，加密和解密使用不同的钥匙，私钥不需要通过网络进行传输，安全性很高。
缺点：计算量比较大，加密和解密速度相比对称加密慢很多。
由于非对称加密的强安全性，可以用它完美解决对称加密的密钥泄露问题，效果图如下：
![在这里插入图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/16.png)
在上述过程中，客户端在拿到服务器的公钥后，会生成一个随机码 (用 KEY 表示，这个 KEY 就是后续双方用于对称加密的密钥)，然后客户端使用公钥把 KEY 加密后再发送给服务器，服务器使用私钥将其解密，这样双方就有了同一个密钥 KEY，然后双方再使用 KEY 进行对称加密交互数据。在非对称加密传输 KEY 的过程中，即便第三方获取了公钥和加密后的 KEY，在没有私钥的情况下也无法破解 KEY (私钥存在服务器，泄露风险极小)，也就保证了接下来对称加密的数据安全。而上面这个流程图正是 HTTPS 的雏形，HTTPS 正好综合了这两种加密算法的优点，不仅保证了通信安全，还保证了数据传输效率。
**HTTPS的加密实现**
1.客户端请求 HTTPS 网址，然后连接到 server 的 443 端口 (HTTPS 默认端口，类似于 HTTP 的80端口)。

2.采用 HTTPS 协议的服务器必须要有一套数字 CA (Certification Authority)证书，证书是需要申请的，并由专门的数字证书认证机构(CA)通过非常严格的审核之后颁发的电子证书 (当然了是要钱的，安全级别越高价格越贵)。颁发证书的同时会产生一个私钥和公钥。私钥由服务端自己保存，不可泄漏。公钥则是附带在证书的信息中，可以公开的。证书本身也附带一个证书电子签名，这个签名用来验证证书的完整性和真实性，可以防止证书被篡改。

3.服务器响应客户端请求，将证书传递给客户端，证书包含公钥和大量其他信息，比如证书颁发机构信息，公司信息和证书有效期等。Chrome 浏览器点击地址栏的锁标志再点击证书就可以看到证书详细信息。
4.客户端解析证书并对其进行验证。如果证书不是可信机构颁布，或者证书中的域名与实际域名不一致，或者证书已经过期，就会向访问者显示一个警告，由其选择是否还要继续通信。
如果证书没有问题，客户端就会从服务器证书中取出服务器的公钥A。然后客户端还会生成一个随机码 KEY，并使用公钥A将其加密。

5.客户端把加密后的随机码 KEY 发送给服务器，作为后面对称加密的密钥。

6.服务器在收到随机码 KEY 之后会使用私钥B将其解密。经过以上这些步骤，客户端和服务器终于建立了安全连接，完美解决了对称加密的密钥泄露问题，接下来就可以用对称加密愉快地进行通信了。

7.服务器使用密钥 (随机码 KEY)对数据进行对称加密并发送给客户端，客户端使用相同的密钥 (随机码 KEY)解密数据。

8.双方使用对称加密愉快地传输所有数据。


**排序都有哪几种方法？请列举。口述用JAVA实现快速排序。**

排序的方法有：插入排序（直接插入排序、希尔排序），交换排序（冒泡排序、快速排序），选择排序（直接选择排序、堆排序）， 归并排序，分配排序（箱排序、基数排序）快速排序的伪代码。
//使用快速排序方法对a[0:n-1]排序
从a[0:n-1]中选择一个元素作为middle，该元素为支点
把余下的元素分割为两段left和right，使得left中的元素都小于等于支点，而right中的元素都大于等于支点
递归地使用快速排序方法对left进行排序
递归地使用快速排序方法对right进行排序
所得结果为left+middle+right


## 六、Java异常机制
![这里写图片描述](https://gitee.com/zhpanvip/images/raw/master/project/article/javainterview/17.png)
**1.Java异常分类**
*1)Exception：*
可以是可被控制(checked) 或不可控制的(unchecked)。
表示一个由程序员导致的错误。
应该在应用程序级被处理。
Checked exception: 这类异常都是Exception的子类。异常的向上抛出机制进行处理，假如子类可能产生A异常，那么在父类中也必须throws A异常。可能导致的问题：代码效率低，耦合度过高。
Unchecked exception: 这类异常都是RuntimeException的子类，虽然RuntimeException同样也是Exception的子类，但是它们是非凡的，它们不能通过client code来试图解决，所以称为Unchecked exception 。
*2)Error：*
总是不可控制的(unchecked)。
经常用来用于表示系统错误或低层资源的错误。
如何可能的话，应该在系统级被捕捉。

**2.JAVA语言如何进行异常处理，关键字：throws,throw,try,catch,finally分别代表什么意义？在try块中可以抛出异常吗？**

Java 通过面向对象的方法进行异常处理，把各种不同的异常进行分类，并提供了良好的接口。在Java中，每个异常都是一个对象，它是Throwable类或其它子类的实例。当一个方法出现异常后便抛出一个异常对象，该对象中包含有异常信息，调用这个对象的方法可以捕获到这个异常并进行处理。Java的异常处理是通过5个关键词来实现的：try、catch、throw、throws和finally。一般情况下是用try来执行一段程序，如果出现异常，系统会抛出（throws）一个异常，这时候你可以通过它的类型来捕捉（catch）它，或最后（finally）由缺省处理器来处理。
用try来指定一块预防所有"异常"的程序。紧跟在try程序后面，应包含一个catch子句来指定你想要捕捉的"异常"的类型。
throw语句用来明确地抛出一个"异常"。
throws用来标明一个成员函数可能抛出的各种"异常"。
Finally为确保一段代码不管发生什么"异常"都被执行一段代码。
可以在一个成员函数调用的外面写一个try语句，在这个成员函数内部写另一个try语句保护其他代码。每当遇到一个try语句，"异常"的框架就放到堆栈上面，直到所有的try语句都完成。如果下一级的try语句没有对某种"异常"进行处理，堆栈就会展开，直到遇到有处理这种"异常"的try语句。

**3.说一说你在开发过程中最常见到的 runtime exception 异常？**
NullPointerException,
ClassCastException,
IndexOutOfBoundsException,
StackOverflowException,
IllegalArgumentException,
SecurityException,
BufferOverflowException,
NoSuchElementException,
EmptyStackException,
ArithmeticException,
BufferUnderflowException,
