---
layout: article
index_img: https://gitee.com/zhpanvip/images/raw/master/blog/img/generics.png
title: Java进阶--详解Java中的泛型（Generics）
date: 2021-01-16 22:14:25
categories:
- Java进阶
tags:
- Java泛型
---

Java泛型是在JDK1.5中引进来的一个概念。泛型意为泛化的参数类型，英文为**Generics** ，翻译过来其实就是通用类型的意思。泛型在平时开发中经常用到，例如常用的集合类、Class类等都是JDK给我们提供的泛型类，更多的时候我们还会使用自定义泛型。可见，泛型在Java体系中还是一个很重要的知识。那么，本篇文章我们就来系统的学习一下Java的泛型。

## 一、为什么要引入泛型
上边已经提到，泛型是在JDK 1.5引进来的一个概念。我们知道，现在声明一个List集合是需要指定List的泛型的，指定了List的泛型后，List就只能接受我们指定的类型。而在JDK 1.5之前，由于没有泛型的概念，List集合接受的是一个Object类型。我们知道Object是Java中所有类的父类，那么这也就意味着声明一个List集合后这个集合可以用来存放任意类型的数据。

举个例子，我们声明一个没有泛型的List集合，并尝试添加不同的数据类型。代码如下：

```handlebars
		List list=new ArrayList();
        list.add(123);
        list.add("abc");
        list.add(new Object());
        for (Iterator iterator = list.iterator(); iterator.hasNext();) {
			Object object = (Object) iterator.next();
	        System.out.println(object);
		}
```
在上边代码中，我们在List的集合种添加了三种不同的数据类型，而这样的写法在IDE中是不会由任何错误提示的。并且可以正常运行程序并打印出数据。

> 123
>abc
>java.lang.Object@4926097b

这样的代码不用想也知道是非常危险的，假设我们期望在集合中存放的是Integer类型，但却在集合中存入了String，那么在使用集合数据的时候把数据都当成Integer处理，程序必然会崩溃。也就是在这样的情况下，为了提高Java语言的安全性以及程序的健壮性，Java在1.5的版本种提供了泛型的支持。有了泛型之后，便可以确保集合中存放的一定是我们所期望的类型。

修改上面的代码，将List泛型指定为Integer，当我们添加非整数类型的参数时IDE就会提示相应的错误。并且在编译时编译器也会抛出错误致使程序无法编译成字节码文件。如下图所示。

IDE提示异常：
![IDE提示类型错误](https://img-blog.csdnimg.cn/20210116170553325.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70)
编译时编译器抛出异常：
![编译器编译错误](https://img-blog.csdnimg.cn/20210116171903386.png)
通过这一个例子可以认识到泛型在Java中有着多么重要的意义。当然，泛型的用途远不止这一点，比如我们可以通过自定义的泛型类结合多态来提高程序的可可扩展性等。

## 二、泛型基本使用
既然泛型这么重要，那么先来学习一下泛型的使用。泛型可以定义在类、接口或者方法上。定义的地方不同，泛型的作用域也不同，比如将泛型定义在类上，那么这个泛型可以作用于整个类。而如果将泛型定义在方法上，那么这个泛型的作用域仅限于这个方法。

### 1.泛型类
首先，我们来看如何定义一个泛型类。

```handlebars
public class Response<T> {

	private T data;

	public T getData() {
		return data;
	}

	public void setData(T data) {
		this.data = data;
	}

}
```
上述代码我们定义了一个Response类，并为其声明了一个泛型参数T。那么在这个Response类中，我们就可以将T作为一个数据类型来使用。比如可将T当作一个成员变量声明在Response中；可以作为方法的返回值类型，也可以作为方法的参数。但是，至于这个T指代的是什么类型，此时还并不能确定，因为这需要我们在使用的时候来自行指定T的类型。

如下，我们在声明Response的时候将T指定为String类型，那么此时Response中的T就确定了，一定是一个String类型。

```handlebars
Response<String> response = new Response<>();
response.setData("abc");
String data = response.getData();
```
指定泛型的类型后，我们便可以理所当然的把T当作String进行set和get，且无需再进行类型转换。
### 2.泛型接口
泛型除了可以指定在类上也可以指定在接口上，使用方式和类泛型是一模一样的。

```handlebars
public interface IResponse<T> {
	
	T getData();
	
	void setData(T t);
}
```
上边代码中我们定义了一个IResponse的接口，并为其声明了一个泛型，在接口中添加了两个抽象方法，分别将T作为方法的返回值类型，和参数类型。接着我们来看一下泛型接口的实现类：

假如说实现IResponse接口的类已经确定了泛型的类型。比如，事先我们已经知道返回的类型是一个String。则可有如下代码：

```handlebars
public class StringResponse implements IResponse<String> {

	@Override
	public String getData() {
		return null;
	}

	@Override
	public void setData(String t) {
		
	}

}
```
上边代码我们定义了一个StringResponse 类并实现了IResponse接口，而IResponse指定了泛型为String。那么在StringResponse 类中重写getData和setData方法的返回值和参数类型都为String。

但是，假如我们现在不确定Response的是一个什么类型的数据，那么则可以继续声明一个Response的泛型类，并实现IResponse接口，并将接口的泛型指定为Response的泛型。代码如下：

```handlebars
public class Response<T> implements IResponse<T>{

	private T data;

	public T getData() {
		return data;
	}

	public void setData(T data) {
		this.data = data;
	}
}
```
此时的代码其实就是声明了一个Response的泛型类，并将Response的泛型T作为了IResponse的泛型。

### 3.泛型方法
除了在类和接口上可以声明泛型外，在方法上也是可以声明泛型的。前边提到了类和接口的泛型都是作用于整个类的。而在方法上声明泛型，泛型的作用于则只作用于这个方法。我们来看一个例子：

```handlebars
public class Model {	
	public <M> void setData(M data) {
		System.out.println(data.toString());
	}
}
```
在Model中有一个setData的方法，由于该方法接受的参数类型不确定，因此我们将这个方法定义成了泛型方法。在调用Model的setData方法的时候需要指定这个方法接受的类型。如下：

```handlebars
Model model=new Model();
model.<String>setData("string");
```
由于在Java8中对于泛型方法的调用做了优化，可以省略指定泛型的类型，直接传入参数即可。

```handlebars
Model model=new Model();
model.setData("string");
```

当然，这里其实编译器根据我们传入的参数做了类型推断。

有同学可能会有疑问，那我直接把泛型声明到类上，然后在这个方法中使用不行吗？当然是可以的，其实跟在方法上声明泛型是一样的效果。我们前文也已经提到了，方法上的泛型与类上的泛型只是作用于不同罢了。但是，如果这个泛型参数仅仅只在方法中使用了，我们是没必要把泛型声明到类上去的。

#### 4.限定泛型的类型
我们仍以Response的场景为例，假如我希望Response类接受的参数不是任意类型的，而是希望Response接受的数据类型是BaseData或者BaseData的子类。这种情况下我们就需要在指定Response的泛型的时候对泛型参数做一个约束。

定义一个BaseData类，类中有一个token，如下：

```handlebars
public class BaseData {
	private String token;

	public String getToken() {
		return token;
	}
}
```
将Response的泛型声明为\<T extends BaseData>

```handlebars
public class Response<T extends BaseData>{![在这里插入图片描述](https://img-blog.csdnimg.cn/20210116193503911.png)


	private T data;

	public String getToken() {
		if(data!=null) {
			return data.getToken();
		}
		return null;
	}
}
```
那么此时Response类中的成员变量T就只能是一个BaseData类型，并且T拥有BaseData的方法，如上代码可以直接通过T调用getToken方法。

接下来，我们来测试一下Response泛型的作用范围，将String作为Response的泛型参数，IDE则会提示mismatch的错误，并且无法通过编译。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210116193511309.png)
只有指定Response的泛型为BaseData或者其子类才能正常编译。

### 5.泛型与通配符
Java中的通配符大家应该都不陌生，在Java中可以使用"?"来表示通配符，用来指代未知类型。而通配符与泛型的搭配也是开发中经常用到的。

众所周知，在Java中Object类是所有类的父类，例如String的顶级父类一定是Object。但是，并不能说List\<String>的顶级父类是List\<Object>。下面我们来看一个例子：

有Person、Student和Teacher三个类，它们的继承关系如下：

```java
//  Person类
public class Person {

}

// Student类
public class Student extends Person{![在这里插入图片描述](https://img-blog.csdnimg.cn/20210116200353115.png)


}

// Teacher类
public class Teacher extends Person {

}
```
接下来分别实例化出Student与Teacher，并分别赋值给Person，代码如下：
```java
Student student = new Student();
Teacher teacher = new Teacher();

Person person;	
person = student;
person = teacher;
```
熟悉Java多态的同学都应该知道上边的代码是没有任何错误的。那么再来看下边的代码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021011620061790.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70)

上述代码IDE却提示了一个mismatch的错误。但是如果我们就是希望personList能够接受studentList也能够接受teacherList应该怎么办呢？这种情况在现实开发中可是非常常见的。此时，我们就可以用通配符来解决，将personList的泛型修改为\<? extends Person>即可，代码如下：

```java
List<? extends Person> personList;
List<Student> studentList = new ArrayList<>();
List<Teacher> teacherList = new ArrayList<>();

personList = studentList;
personList = teacherList;
```
另外，我们还可以通过\<? super Student>来指定接收Student或者Student的父类，代码如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210116202145499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwNTIxNTcz,size_16,color_FFFFFF,t_70#pic_center)
可以看到上述代码中listSuper的泛型声明为了\<? super Student>，此时listSuper就只能接收Student以及其父类的集合。所以可以看到，代码中将studentList与personList以及ObjectList正常赋值给listSuper，但是teacherList赋值给listSuper则会报错。

## 三、泛型的类型擦除
到这里关于泛型的基础知识差不多已经讲完了。可以发现，上边讲到的内容都是程序编译前的代码。程序中一旦有不符合规范的代码IDE都会提示错误，并且编译器在编译源代码时就会抛出异常。那接下来要讲的泛型擦除就是代码编译后的知识了。

前文已经提到，泛型是在JDK1.5中引入的概念，在JDK1.5之前的源码中像List这些泛型类都是使用Object来实现的。那问题来了，JDK1.5版本是否能够兼容JDK1.4或者之前的版本呢？答案是肯定的。之所以能够实现JDK的向下兼容就是因为在编译期间编译器进行了类型擦除。

我们应该怎么理解类型擦除呢？其实就是编译器在对源代码进行编译的时候将泛型换成了泛型指定的上限类，如果没有指定泛型的上限，编译器则会使用Object类替代。简单的说在编译完成后的字节码文件中其实是没有泛型的概念的，源代码中的泛型被编译器用Object或者泛型指定的类给替换掉了。这也是为什么JDK1.5能够向下兼容的原因。


我们可以通过javap的反汇编来证明编译期间的类型擦除。定一个Model类，代码如下：

```java
public class Model {

    public void test() {
    	List<String> list=new ArrayList();
    	 list.add("abc");
    }
    
    public void test2() {
    	List list=new ArrayList();
    	list.add("abc");
    }
   
}
```
可以看到，这个类中有两个方法，这两个方法不同的地方在于test方法中指定了List的泛型为String，而test2方法中未指定List的泛型。我们先将Model.java通过javac工具编译成Model.class文件，然后通过javap反汇编Model.class文件，得到结果如下：

```java
$ javap -c Model.class
Compiled from "Model.java"
public class com.test.reflection.Model {
  public com.test.reflection.Model();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void test();
    Code:
       0: new           #2                  // class java/util/ArrayList
       3: dup
       4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #4                  // String abc
      11: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      16: pop
      17: return

  public void test2();
    Code:
       0: new           #2                  // class java/util/ArrayList
       3: dup
       4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #4                  // String abc
      11: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      16: pop
      17: return
}
```
通过反汇编指令可以看到test1方法与test2方法并无任何区别，并且可以看到第18和30行的注释：

>// InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z

都是向List集合中添加了一个Object对象。

另外，我们也可以通过反射证实类型擦除的存在。上篇文章[《Java进阶--深入理解Java的反射机制》](https://blog.csdn.net/qq_20521573/article/details/111751156)我们学习了Java的反射，知道Class对象是在类加载时候生成的，并且反射是在程序运行时的操作。那我们来通过反射List的Class对象，并尝试添加向List中添加不同的类型看是否能够成功。代码如下：

```java
		List<String> list=new ArrayList<>();
		list.add("abc");
		
		Class<List> listClass=List.class;
		
		try {
			Method addMethod=listClass.getDeclaredMethod("add", Object.class);
			addMethod.invoke(list, 123);
			
		} catch (NoSuchMethodException | SecurityException|IllegalAccessException
				| IllegalArgumentException | InvocationTargetException e) {
			e.printStackTrace();
		}
		
		System.out.println("list.size() = "+list.size());
		
		for (Object obj : list) {
			System.out.println(obj.toString());
		}
```

输出结果：

```java
list.size() = 2
abc
123
```


上述代码我们声明了一个泛型为String的List集合，并向集合中添加了一个字符串“abc"，接着通过反射向集合list中添加了一个整数类型123，通过输出结果可以看到两个值都被添加到了集合中。这一结果也印证了泛型的类型擦除。

## 四、总结
泛型是开发中使用频率非常高的一个技术点，泛型的引入使得Java语言更加安全，也增强程序的健壮性。通过本篇文章我们系统的学习Java的泛型。同时也明白了Java的泛型仅仅是在编译期间由IDE和编译器来进行检查和校验的。在编译后的字节码文件以及运行期间的JVM中是没有泛型的概念的。也正是因为这一原因，其实我们可以通过编辑字节码文件或者在运行时通过反射来绕过泛型的校验，完成代码编写期间不能实现的操作。
