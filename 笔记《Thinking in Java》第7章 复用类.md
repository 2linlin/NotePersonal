---
title: 笔记《Thinking in Java》第7章 复用类
date: 2019-08-01 02:34:26
categories:
- JavaSE
- Java基础
tags:
- 学习笔记
- Java基础
top:
---

### 第七章 复用类

总共有这几种方法：组合，继承，代理。

#### 1.组合

略。

#### 2.继承

如果没有显示的声明继承，那么这个类是隐式继承Object的。

即使一个程序里很多类都有main方法，也只有命令行运行的那个类的main方法能够运行。

当创建子类对象时，它内部其实是隐式的包含了一个父类的对象的。

子类的构造方法的第一条命令，隐式的包含有一条super()；语句。

#### 3.代理

代理对象持有私有的子对象。并用自己的方法对子对象的方法进行封装。一般是通过代理对象和子对象实现同一个接口来处理。

#### 4.protect关键字

父类的属性想让子类访问时，就用protected。这个是一个常见的使用方式：

```java
public class Parent {
	private String superPrivateName;	// 子类只能通过get方法访问
	protected String superProtectedName;// 子类可直接访问
	
	protected String getSuperPrivateName() {
		return superPrivateName;
	}
｝
public class Son extends Parent {
	Son() {
		super.superProtectedName = "get super protected~";
		super.getSuperPrivateName();
	}
｝

```

#### 5.final关键字

总述：

* final修饰类，类不能被继承

* final修饰方法，方法不能被重写

* final修饰变量，变量不能被改变(修饰引用变量时，该变量不能再指向另一个对象)。可理解为只能赋值一次。

##### 修饰变量

使用static final修饰基本类型：static的重点是强调只有一份，final的重点是强调它是常量。如果只用final修饰，那么new一个新对象的时候可以改变。用了static final，那么就是类对象的，一旦初始化以后，再new一个新对象也改变不了它。

不能因为某数据是final就认为编译时就已经确认值。还可以在运行时再确认。

```java
public void method(final int args) {
    final String methodVar;
}
```

static和非static的数据，在类初始化时是一视同仁的。

Java允许空白final，即声明final时不赋值：

* 对于类变量，不管是static还是非static，编译器一定要你保证在后续静态块或者构造方法(也是隐式的静态方法)里赋值。

  为什么类变量有要求，局部变量没有要求呢？其实这个是变量初始化的问题。因为编译器需要保证类变量初始化，无需保证局部变量初始化。然后呢，你声明了final，变量只能赋值一次，也就说编译器不能给它赋默认值，如果编译器赋值过了，你还怎么玩？所以编译器告诉你，你的final影响到我的类变量初始化赋值了，所以你来给我保证初始化。至于局部变量的话，我不负责它的初始化赋值，所以你爱怎么玩怎么玩。

```java
public class Son extends Parent {
	private final String finalVar;// 需要构造方法保证后续初始化
	private static final String staticFinalVar;// 需要静态块保证后续初始化
	
	static {
        // 如果没有在声明时初始化，就必须在静态块里初始化。不然编译器就报错。
        // 在静态方法里赋值都不行。必须是构造方法执行前赋值。
		staticFinalVar = "";
	}
	
	Son() {
        // 如果没有在声明时初始化，就必须在所有构造方法里初始化。不然编译器就报错。
		finalVar = "";	
		System.out.println("子类构造方法执行");
    }
```

* 对于局部变量，则没有这种赋值的要求。局部变量可以是静态/非静态代码块，静态/非静态方法这些所有的情况，以及静态/非静态方法入参。

```java
public class Son extends Parent {
    // 静态代码块
	static {
		final String methodVar;
		staticFinalVar = "";
	}
    // 静态方法及入参
    Son(final String methodVarIn) {
		final String methodVar;
    ｝
    // 非静态代码块
	{
		final String methodVar;
	}
	// 非静态方法及入参
	public void method(final String methodVarIn) {
		final String methodVar;
	}
```

* 修饰形参时，形参只是在方法内不会被改变。方法外当然还是可以改变的

```java
	public static void main(String[] args) {
		String a = "before: 1";
		System.out.println(a);	// "before: 1";
		method(a);
		a = new String("after: change!");	// 赋新值，并不报错，final的作用范围仅在方法内
		System.out.println(a);	// "after: change!";
	}
	
	public static void method(final String a) {
		System.out.println(a);	// "before: 1";
	}
```

有些人甚至建议将所有局部变量和形参变量声明为final，认为这样可以提高性能，便于编译器优化。其实没有必要，编译器没那么笨。不过final在多线程的环境下，适当使用，确实可以提升性能，因为不必再进行安全检查。

所有的private变量都被隐式的指定为了final。

##### 修饰方法

目的：

1.锁定方法，让它不再被重写。

2.提高效率。早期的Java实现，如果将一个方法声明为final，则就是同意编译器将针对这个方法的所有调用都转换为内部调用。也就是，用方法体中的实际代码副本来替代方法调用，这样就省去了方法调用的开销。不过现在这种用法不建议了，因为在方法体很大的时候，其实相比起方法执行时间，方法调用的时间占比已经很小了。另外，其实JVM已经很聪明了，能自动识别并处理这种情况。

正常的方法调用就是，参数压入栈，跳至方法代码处执行，跳回并清理栈中参数，处理返回值。

所有的private方法都被隐式的指定为了final。如果子类“继承”了父类的private方法，其实你会发现并没有继承，而是创建了一个新的方法而已。我们可以这样证明：生成子类对象，用子类调用该方法可以，向上转型成父类，再调用该方法，就不行了。

##### 修饰类

final修饰的类不能继承。

所以，该类里面的所有方法也是隐式final的。

#### 6.初始化及类加载

每个类的编译代码都存在它自己的独立文件中，只有第一次用到的时候才会加载。比如说用到static字段，或是构造方法。

父类子类的初始化顺序，见前文，总结为：

父类静态属性初始化 = 父类静态代码块 > 子类静态属性初始化 = 子类静态代码块 >

父类普通属性初始化 = 父类普通代码块 > 父类构造方法 >

子类普通属性初始化 = 子类普通代码块 > 子类构造方法