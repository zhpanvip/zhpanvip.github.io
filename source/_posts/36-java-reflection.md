---
layout: article
index_img: https://cdn.jsdelivr.net/gh/zhpanvip/images/blog/img/reflection.png
title: Java进阶--深入理解Java的反射机制
date: 2021-01-14 23:53:35
categories:
- Java进阶
tags:
- Java反射
---

在上篇文章[《深入JVM--探索Java虚拟机的类加载机制》](https://blog.csdn.net/qq_20521573/article/details/109193527)中我们深入探讨了JVM的类加载机制。我们知道，在实例化一个类时，如果这个类还没有被虚拟机加载，那么虚拟机会先执行类加载过程，将该类所对应的字节码读取到虚拟机，并生成一个与这个类对应的Class对象。而在类加载的过程中，由于有双亲委派机制的存在，虚拟机保证了同一个类会被同一个类加载器所加载，进而保证了在虚拟机中只存在一个被加载类所对应的Class实例。而这个Class实例与我们今天要讲的反射有着莫大的关系。



## 一、Java反射机制概述
在学习反射之前，我们先来搞清楚几个概念：
- Class类是什么？
- Class对象是什么？
- Class对象与我们的Java类有什么关系？

假设现在有一个Person类，代码如下：
```java
package com.test.reflection;

public class Person {
	
    private String name;
    protected int age;
    public String sex;

    private Person() {

    }

    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }

    private void testPrivateMethod() {
        System.out.println("testPrivateMethod被调用");
    }
}
```
你是否能通过Person类来解释清楚上边提到的三个问题呢？我们不妨接着往下看。
### 1.Class类与Class对象
提到Class类，大家多多少少都应该有些接触，即使你没有使用过反射，也不可避免的接触到Class类。例如，在Android中进行Activity页面跳转的时候，我们需要一个Intent，而实例化Intent则需用到Intent的构造方法，代码如下：

```java
public Intent(Context packageContext, Class<?> cls) {
        mComponent = new ComponentName(packageContext, cls);
    }
```
可以看到，Intent的构造方法中的第二个参数，接受的就是一个Class对象。到这里，我们应该都明白，Class就是JDK为我们提供的一个普普通通的Java类，它跟我们自己定义一个Person类其实并无任何本质上的区别。我们进入Class类的源码可以看到，Class类是一个泛型类，并且它实现了若干个接口，源码如下：

```java
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement,
                              TypeDescriptor.OfField<Class<?>>,
                              Constable {
	// ...省略主体代码
}
```
既然Class就是一个普普通通的Java类，那在使用它的时候一定需要实例化出来一个Class对象。但奇怪的是，我们在平时写代码的时候好像从来没有通过new关键字来实例化过Class对象？那它到底是在哪里被实例化的呢？了解类加载机制的同学想必应该都清楚，当然，我们在文章开头也已经提到了，Class对象是在类加载的时候由虚拟机自动生成的。

我们以上边的Person类为例，当我们使用new关键字实例化Person对象的时候，如果Person类的字节码还没有被加载到虚拟机，那么虚拟机首先启动类加载器将Person类的字节码读取到虚拟机中，并为其生成一个Class\<Person>的实例，而类加载器的双亲委派模型保证了虚拟机中只会生成一个Class\<Person>的实例。而如果在实例化Person对象的时候，Person已经被加载到了虚拟机，则无需再进行Person的类加载过程，直接实例化Person即可。到这里，我们似乎可以感觉到Person对象跟Class\<Person>一定存在着某种关系。我们接着往下看。

### 2.Person类与Class\<Person>对象的关系
现在，我们回想一下我们在进行Activity页面跳转的时候Intent构造方法的第二个参数传的是什么呢？是不是像下边这样：

```java
	Intent intent=new Intent(this,MainActivity.class);
```
通过MainActivity.class我们可以得到MainActivity对应的Class对象：

```java
	Class<MainActivity> mainActivityClass = MainActivity.class;
```
而mainActivityClass 对象就是虚拟机在加载MainActivity的时候生成的，并且虚拟机保证了mainActivityClass在虚拟机中是唯一的。
这一过程对于Person类也是一样的，我们可以通过Person .class来拿到虚拟机中唯一的一个Class\<Person>实例。

```java
	Class<Person> personClass=Person.class;
```
另外，我们还可以通过Person 的实例对象来获得Class\<Person>对象，如下：

```java
	Person person=new Person();
	Class<Person> personClass=(Class<Person>)person.getClass();
```
当然，除了上述两种方法之外，我们还可以通过Person类的包名来获得Class\<Person>的实例，代码如下：

```java
	try {
			Class<Person> personClass=(Class<Person>)Class.forName("com.test.reflection");
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
```

由于Class\<Person >对象在虚拟机中是唯一的，那么上述三种方法获取到的Class\<Person >一定是同一个实例。


## 二、什么是反射？
好了，上边啰嗦了这么多，终于到了正题了。那到底什么是反射呢？我们来看下百度百科给出的定义：

> Java的反射（reflection）机制是指在程序的运行状态中，可以构造任意一个类的对象，可以了解任意一个对象所属的类，可以了解任意一个类的成员变量和方法，可以调用任意一个对象的属性和方法。这种动态获取程序信息以及动态调用对象的功能称为Java语言的反射机制。

虽然上述定义对反射的描述已经非常清楚。但是对于没有了解过反射的同学来说看了之后可能还是一头雾水。下面，在第一章的基础上来说下我来说下我对反射的理解：

**在Java中，所有已经被虚拟机的类加载器加载过的类（称为T）都会在虚拟机中生成一个唯一的与T类所对应的Class\<T>对象。在程序运行时，通过这个Class\<T>对象，我们可以实例化出来一个T对象；可以通过Class\<T>对象访问T对象中的任意成员变量，调用T对象中的任意方法，甚至可以对T对象中的成员变量进行修改。我们将这一系列操作称为Java的反射机制。**

到这里我们发现，其实Java的反射也没有那么神秘了。说白了就是通过Class对象来操控我们的对象罢了。因此，接下来我们想要弄懂反射只需要来详细的认识一下Class这个类给我们提供的API即可。

### 1.Java反射相关类

我们知道，一个Java类可以包含成员变量、构造方法、以及普通方法。同时，我们又知道Java是一种很纯粹的面向对象的语言。在Java语言中，万物皆对象，类的成员变量、构造方法以及普通方法在Java中也被封装成了对象。它们分别对应Field类、Constructor类以及Method类。这几个类与反射息息相关。因此，在开始之前，我们需要先了解下这几个与反射相关的类，如下图：
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86c57545a37b4c40bb31900eb1ba4f5c~tplv-k3u1fbpfcp-zoom-1.image)
- **Field 类**：位于Java.lang.reflect包下，在Java反射中Field用于获取某个类的属性或该属性的属性值。
- **Constructor 类：** 位于java.lang.reflect包下，它代表某个类的构造方法，用来管理所有的构造函数的类。
- **Method 类：** 位于java.lang.reflect包下,在Java反射中Method类描述的是类的方法信息（包括：方法修饰符、方法名称、参数列表等等）。
- **Class 类：** Class类在上文我们已经多次提到，它表示正在运行的 Java 应用程序中的类的实例。
- **Object 类：** Object类大家应该都比较熟悉了。它是所有 Java 类的父类。所有对象都默认实现了 Object 类的方法。在Object对象中，可以通过getClass()来获得该类对应的Class实例。

### 2.Java反射常用API
通过上文我们已经知道，所谓的反射，其实就是通过API操作Class对象。因此，在进行反射操作的第一步我们应该首先拿到Class的实例。在第一种中我们已经知道可以通过三种方式来获得Class的实例。以获取Person类的Class对象为例，三种方法分别如下：

```java
	// 通过类的class获得Class实例
	Class<Person> personClass=Person.class;
	// 通过类的包名获得Class实例
	Class<Person> personClass=(Class<Person>)Class.forName("com.test.reflection");
	// 通过对象获得Class实例
	Person person=new Person();
	Class<Person> personClass=(Class<Person>)person.getClass();
```
在拿到Person类的Class实例后，我们就可以通过Class实例获取到Person类中的任意成员，包括构造方法、普通方法、成员变量等。

#### 2.1获取所有构造方法

 Class类中为我们提供了两个获取构造方法的API，这两个方法如下：
 

 - Constructor[] getDeclaredConstructors() 用于获取当前类中所有构造方法。但不包括包括父类中的构造方法。
 - Constructor[]  getConstructors()  用于获取本类中所有public修饰的构造方法,不包括父类的构造方法。

**（1）getDeclaredConstructors()** 

以Person类为例，我们来先来尝试getDeclaredConstructors方法的使用：

```java
		Class<Person> personClass=Person.class;
		Constructor[] declaredConstructors= personClass.getDeclaredConstructors();
		for(Constructor declaredConstructor:declaredConstructors) {
			System.out.println(declaredConstructor);
		}
```
注意，我们在Person类中声明了两个构造方法，其中无参构造方法是一个私有的构造方法。我们来看下上述代码的打印结果：

> private com.test.reflection.Person()
public com.test.reflection.Person(java.lang.String,int)

可以看到，getDeclaredConstructors方法可以获取到类中包括私有构造方法在内的所有构造方法。

**（2） getConstructors()**  

接着我们将getDeclaredConstructors()方法换成getConstructors()方法:

```java
		Class<Person> personClass=Person.class;
		Constructor[] declaredConstructors= personClass.getConstructors();
		for(Constructor declaredConstructor:declaredConstructors) {
			System.out.println(declaredConstructor);
		}
```

再来看输出结果：

> public com.test.reflection.Person(java.lang.String,int)

此时，只有被声明了public的方法被打印了出来。

#### 2.2 获取指定构造方法

在Class中同样提供了两个获取指定构造方法的API，如下：

- Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)  该方法用来获取类中任意的构造方法，包括private修饰的构造方法。无法通过该方法获取到父类的构造方法。
- Constructor<T> getConstructor(Class<?>... parameterTypes)  该方法只能用来获取该类中public的构造方法。无法获取父类中的构造方法。

我们可以尝试使用getDeclaredConstructor方法来获取Person的私有构造方法与public的有参构造方法：

```java
       try {
            Constructor<Person> declaredConstructor = personClass.getDeclaredConstructor();
            Constructor<Person> declaredConstructor2 = personClass.getDeclaredConstructor(String.class,int.class);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
```
而如果使用getConstructor获取私有方法，则会抛出java.lang.NoSuchMethodException。

#### 2.3 使用反射实例化对象
通过反射实例化对象有多种途径，可以使用Class的newInstance方法，同时也可以使用Constructor类。

- 通过Class的newInstance实例化Person
- 使用Constructor实例化对象

**（1）通过Class的newInstance实例化对象**
这种方式使用起来非常简单，直接调用newInstance方法即可完成对象的实例化。代码如下：

```java
		Class<Person> personClass = Person.class;
        try {
            Person person = personClass.newInstance();
        } catch (IllegalAccessException | InstantiationException e) {
            e.printStackTrace();
        }
```
但是，通过这一方法有一定的局限性。即只能实例化无参构造方法的类，同时这个无参构造方法不能使用private修饰。否则会抛出异常。这个方法在Java 9中已经被声明为Deprecated，并且推荐使用Constructor来实例化对象。

**（2） 使用Constructor实例化对象**

```java
	try {
            Constructor<Person> declaredConstructor = personClass.getDeclaredConstructor(String.class, int.class);
            Person ryan = declaredConstructor.newInstance("Ryan", 18);
            System.out.println(ryan.getName() + "---" + ryan.getAge());
        } catch (NoSuchMethodException | InvocationTargetException
                | InstantiationException | IllegalAccessException e) {
            e.printStackTrace();
        }
```
如上代码，我们通过Person的有参Constructor实例化出来一个Person类，并输出如下结果：

> Ryan---18

而通过Constructor实例化私有的构造方法时，需要通过Constructor的setAccessible(true)来使Constructor可见，进而进行实例化。否则则会抛出IllegalAccessException异常。实例化私有构造方法的代码如下：

```java
	try {
            Constructor<Person> declaredConstructor = personClass.getDeclaredConstructor();
            declaredConstructor.setAccessible(true);
            Person ryan = declaredConstructor.newInstance();
        } catch (NoSuchMethodException | InvocationTargetException
                | InstantiationException | IllegalAccessException e) {
            e.printStackTrace();
        }
```

#### 2.4 使用反射获取类的所有成员变量
Class同样提供了两方法来获取类的成员变量，分别如下：
- getDeclaredFields()  获取该类中所有成员变量，无法获取到从父类中继承的成员变量。
- getFields()  获取类中所有public的成员变量，包括从父类中继承的public的成员变量。

（1）首先通过getDeclaredFields()来获取Person的成员变量，代码如下：
```java
		Class<Person> personClass = Person.class;
        Field[] declaredFields = personClass.getDeclaredFields();
        for (Field field : declaredFields) {
            System.out.println(field.toString());
        }
```
输出结果为：

> private java.lang.String com.test.reflection.Person.name
protected int com.test.reflection.Person.age
public java.lang.String com.test.reflection.Person.sex

可以看到，无论时private修饰的成员变量还是public修饰的成员变量都通过getDeclaredFields获取到。
（2）接着来看getFields()方法：

```java
		Class<Person> personClass = Person.class;
        Field[] fields = personClass.getFields();
        for (Field field : fields) {
            System.out.println(field.toString());
        }
```
输出结果为：

> public java.lang.String com.test.reflection.Person.sex

可以看到，通过getFields方法只获取到了Person类中public的成员变量。

#### 2.5 反射获取并修改类的成员变量
可以通过getDeclaredField方法与getField获取类中从成员变量，区别如下：

- getDeclaredField(String) 获取该类任意修饰符的成员变量，但不包括从父类中继承的成员变量。
- getField(String) 获取该类任意public修饰的成员变量，包括从父类中继承的public的成员变量。


获取Person类的私有成员变量，并通过反射来修改Person对象中的私有变量name,代码如下：

```java
        Class<Person> personClass = Person.class;
        Person person = new Person("Ryan", 18);
        try {
            System.out.println("反射修改前name为:" + person.getName());
            // 获取Person中的私有成员变量name
            Field name = personClass.getDeclaredField("name");
            // 将name设置为可见
            name.setAccessible(true);
            // 修改person实例中name的值
            name.set(person, "Helen");
            System.out.println("反射修改后name为:" + person.getName());
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

```
输出结果如下：

> 反射修改前name为:Ryan
反射修改后name为:Helen

#### 2.6 反射获取类中的所有方法
- getDeclaredMethods() 获取本类中所有方法，不包括从父类中继承的方法。
- getMethods()  获取类中所有public方法，包括从父类中继承的public方法。

**（1）getDeclaredMethods()** 

通过getDeclaredMethods获取Person中的所有方法（不包括父类）：

```java
        Class<Person> personClass = Person.class;
        // 获取类中的所有方法，包括私有方法，但不包括父类中的方法。
        Method[] declaredMethods = personClass.getDeclaredMethods();
        // 遍历并打印方法信息
        for (Method method :declaredMethods) {
            System.out.println("getDeclaredMethods:"+method.toString());
        }
```

输出结果如下：

> getDeclaredMethods:public java.lang.String com.test.reflection.Person.getName()
getDeclaredMethods:public int com.test.reflection.Person.getAge()
getDeclaredMethods:private void com.test.reflection.Person.testPrivateMethod()


**（2）getMethods**

通过getMethods获取Person中的所有public方法（包括父类）：
```java
        Class<Person> personClass = Person.class;
        //	获取Person类中的所有方法，包括父类的方法
        Method[] methods = personClass.getMethods();
        // 遍历methods并打印方法信息
        for (Method method : methods) {
            System.out.println("getMethods:"+method.toString());
        }
```
输出结果如下：
> getMethods:public java.lang.String com.test.reflection.Person.getName()
getMethods:public int com.test.reflection.Person.getAge()
getMethods:public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException
getMethods:public final void java.lang.Object.wait() throws java.lang.InterruptedException
getMethods:public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException
getMethods:public boolean java.lang.Object.equals(java.lang.Object)
getMethods:public java.lang.String java.lang.Object.toString()
getMethods:public native int java.lang.Object.hashCode()
getMethods:public final native java.lang.Class java.lang.Object.getClass()
getMethods:public final native void java.lang.Object.notify()
getMethods:public final native void java.lang.Object.notifyAll()


#### 2.7 使用反射调用对象的方法
**(1) 反射调用对象的私有方法**

通过反射调用Person类中的私有方法testPrivateMethod，代码如下：
```java
        try {
            // 获取Person类中的私有方法testPrivateMethod
            Method testPrivateMethod = personClass.getDeclaredMethod("testPrivateMethod");
            // 将testPrivateMethod方法设置为可见
            testPrivateMethod.setAccessible(true);
            // 反射调用testPrivateMethod方法
            testPrivateMethod.invoke(person);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
```
输出结果为：
> testPrivateMethod被调用


## 三、关于反射的一些问题

### 1.是否可以通过反射修改final类型的成员变量？
在写Java代码的时候，如果我们将一个成员变量声明为了final类型，那么就必须在声明时候或者在构造方法中为其赋初始值，否则程序是无法编译通过的。那我们是否可以通过反射来修改final类型的成员变量呢？不妨来尝试一下。我们将Person中的sex改为private与final修饰，并为其赋初始值为“male"。

```java
public class Person {
	private final String sex ="male";
	public String getSex() {
    	return sex;
    }
	// ...省略其它代码
}
```
下面通过反射来尝试将sex的值修改为”female"，代码如下：

```java
		Class<Person> personClass = Person.class;
        Person person = new Person("Ryan", 18);
        try {
            Field sex = personClass.getDeclaredField("sex");
            System.out.println("修改性别前：" + person.getSex());
            sex.set(person, "female");
            System.out.println("修改性别后：" + person.getSex());
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
```
运行之后程序并未报错，输出结果如下：

> 修改性别前：male
修改性别后：male

可以看到，我们通过反射并没有成功修改sex的值，这意味着final修饰的成员变量无法通过反射修改吗？这倒未必。我们接着来看下边的代码。

```java
public class Person {
	private final Object mObject = new Object();
	public Object getObject() {
    	return mObject ;
    }
	// ...省略其它代码
}
```
我们在Person中添加一个Object的成员变量，并将其声明为private final。接下来仍然通过反射来尝试修改mObject的值。代码如下：

```java
Class<Person> personClass = Person.class;
        Person person = new Person("Ryan", 18);
        try {
            Field object = personClass.getDeclaredField("mObject");
            System.out.println("修改Object前：" + person.getObject().toString());
            Object newObject = new Object();
            System.out.println("newObject：" + newObject.toString());
            object.setAccessible(true);
            object.set(person, newObject);
            System.out.println("修改Object后：" + person.getObject().toString());
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
```
输出结果如下：


> 修改Object前：java.lang.Object@4926097b
newObject：java.lang.Object@762efe5d
修改Object后：java.lang.Object@762efe5d

有没有很奇怪？上边的String通过反射修改没有成功，而将代码换成Object之后，同样的代码，Object的成员变量却被修改成功了，这是怎么回事呢？其实，了解Java编译的同学应该比较清楚。编译器在编译Java文件的时候会将final修饰的基本类型和String优化为一个常量。我们来看下反编译后的class文件就明白了。我们将Person编译的class文件在AS中打开，如下图：
![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d368dd6fcf4f4a2a9258d79949bf83f9~tplv-k3u1fbpfcp-zoom-1.image)

可以清楚的看到在class文件中getSex方法返回的是一个“male"字面量，而getObject返回的却是mObject。所以，即使通过代码将final修饰的String类型修改成功，在get的时候由于编译器的优化无法拿到修改后的值。

通过上边的例子**可以确定通过反射是能够修改final修饰的成员变量的。只是如果该成员变量是基本数据类型或者String类型会被编译器优化成字面量，从而无法获得修改后的值。**

### 2.为什么说反射会影响程序性能？
在项目开发中，我们在能不使用反射的情况下就不使用反射，因为反射会影响程序的性能。这是我们大家都熟知的。但是你知道为什么说反射会影响程序的性能吗？要解开这一个问题就需要我们深入反射的源码来查看反射过程中都做了什么操作。由于这块内容比较庞大，限于篇幅，关于反射影响性能的问题将在下一篇文章中详细分析。敬请期待。

