---
title: 笔记《Thinking in Java》第5章 初始化与清理
date: 2019-08-01 02:32:26
categories:
- JavaSE
- Java基础
tags:
- 学习笔记
- Java基础
top:
---

### 第五章 初始化与清理

#### 1.构造方法

如果没有构造方法，则实行默认的构造方法。如果自己创建了有参的构造方法，那么默认的无参构造方法就失效了。需要的话要自己创建。

构造方法是没有返回值的(跟返回void不一样)。不过在构造方法里还是可以单独的使用return来表示程序结束。

```java
	public Test201957(int integerIn){
		if (integerIn >= 1) {
			return;			
		}
		System.out.println("integerIn < 1!");
	}
```

构造方法只能在构造方法中调用，其他方法无法调用构造方法。

#### 2.方法重载

可以用不同的参数列表来重载。但是不能用不同的返回值来重载。因为当参数列表一致时，如果直接调用方法而无需返回值，则无法区分该调用哪个方法。

```java
...
method(String str);
...
```

重载时的自动转型逻辑：

* 如果入参比参数列表里的值范围大，则要强制转换类型才不报错。

* 如果输入的值比参数列表里面的值范围小，重载时选择最接近的。例如，byte 在short和int中选short。
* 对于char，如果没有找到接收char参数的方法，则把它当作int来处理。
* 常数也是被当作int来处理。

```java
	static void f1(short x) {System.out.println("f1(short)");}
	static void f1(int x) {System.out.println("f1(int)");}
	static void f1(long x) {System.out.println("f1(long)");}
	
	public static void main(String[] args) {
		System.out.print("constant_: ");
		f1(5);				// constant_: f1(int)
		char char_ = 'c';
		System.out.print("char: ");
		f1(char_);			// char: f1(int)
		byte byte_ = 127;
		System.out.print("byte_: ");
		f1(byte_);			// byte_: f1(short)
	}
```

#### 3.this关键字

##### 正常使用

平时使用时，就把它当作当前对象的引用来用就行了。

##### 作为构造方法使用

this后面直接跟括号，可以作为构造方法使用。有以下原则：

* 只能在构造方法中使用，且必须是第一句程序使用，且只能用一次。

* 不用this()而直接用构造方法名调用，是调不了的。
* 除了构造方法，静态方法、非静态方法都调用不了构造方法。

```java
	// 构造方法
	public Test201957(){
		System.out.println("defalut!");
	}

	// 构造方法
	public Test201957(int integerIn){
		//Test201957(); 非法
		this();// 只能这样用：在构造方法中，用this()调用，在第一行用，且只用一次。
		if (integerIn >= 1) {
			return;			
		}
		//this(); 非法
		System.out.println("integerIn < 1!");
	}	

	// 静态方法
	static void classMethod() {
		//this(); 非法
		//Test201957(); 非法
		System.out.println("Class method invoke!");
	}
	
	// 非静态方法
	void method(int i, String str){
		//this(); 非法
		//Test201957();	非法
    ｝
```

##### static关键字

就是没有this的方法，是类级别的。

#### 4.清理和垃圾回收

##### finalize()方法

是什么？垃圾回收器在准备释放对象时，会先调用对象的finalize()方法，然后在下一次垃圾回收动作时回收对象占用的内存。

解决了什么问题？垃圾回收器只会释放由new分配的内存，即。那么有一个意外，就是如果用非Java的方式，比如本地方法用C语言的malloc()分配了内存，那么垃圾回收器其实回收不了这个内存。所以这个时候，可以在finalize()方法里面，也用本地方法调用C语言对应的free()方法释放内存，从而完成垃圾回收。

怎么用？除了上述的使用场景，一般都不会用到这个方法。由于这个方法执行情况无法预料（JVM不耗尽内存就不会去进行垃圾回收），甚至可以说它是多余的。

##### 垃圾回收器如何工作

由于垃圾回收器的存在，使得Java从堆空间分配内存的速度，可以和其他语言从栈分配内存的速度媲美。为什么呢？比如说，在C++里，堆空间是复用的，在分配堆内存时，有一个查找可用空间的过程。而在Java里，由于垃圾回收会重新紧凑排列对象，整理出新的可用内存，所以Java分配内存，只是简单的将“堆指针”移动到没有分配内存的区域就OK了。节省了查找可用空间的消耗。

常见的垃圾回收算法：

* 引用计数。对象引用统计，为0时回收。问题一：需要不断的遍历所有对象，整个对象的生命周期都存在开销。问题二：两个对象相互引用时，它们都该回收时，计数器不为零。要想发现这种情况，开销很大。所以一般很少用这种处理方式。
* 停止-复制。从栈或者静态存储区，循环遍历所有引用。遍历到的复制走，没遍历到的干掉。缺点：1.需要两套堆内存。2.程序稳定时，其实垃圾较少，但是还是会复制所有对象，很浪费。解决方案：自适应：JVM检测到没有新垃圾产生，就转到标记-清扫算法。
* 标记-清扫。从栈或静态存储区，循环遍历所有引用，活的就标记。遍历完后，没标记的全部干掉。缺点：剩下的堆空间不连续，需要重新整理。

JVM的实际操作：

内存分配以较大的“块”做单位，块被引用，则代数增加。垃圾回收器对上次回收后的新分配的块做整理。JVM进行监视，所有对象都稳定，就切换到“标记-清扫”。如果碎片很多，就切换到“停止-复制”。

JIT：

Just-In-Time。即时编译器技术。这种技术可以将程序预先全部/部分翻译成本地机器码，就不用JVM再来解释执行了，从而提升程序运行速度。就是说，当你要加载一个类的时候，首先是找到.class文件是吧，找到以后，你可以选择是否将字节码编译成本地机器码。一般是必要时才编译，因为全部文件都编译的话，本地机器码比字节码长很多，会导致页面调度，降低程序速度。Java HotSpot用了这种技术，所以代码执行越多，优化越多，速度越快。

#### 5.类变量初始化

对于方法局部变量，编译器强制程序员进行初始化。

对于类变量，基本类型会自动初始化为响应的值，引用类型初始化为null。

有一个好玩的是，类变量可以直接通过方法赋值初始化，而且可以“向前引用”：

```java
public class methodInit {
    // int j = g(i); 报错，因为i这个变量还没定义
    int i = f(5);
    int j = g(i);// 写在后面就没事，因为前面i定义了。而且不给它赋值也会默认赋初值。
    
    int f(int m) {return m;}
    int g(int n) {return n;}
}
```

#### 6.构造方法及类初始化顺序

总结为：

父类静态属性初始化 = 父类静态代码块 > 子类静态属性初始化 = 子类静态代码块 >

父类普通属性初始化 = 父类普通代码块 > 父类构造方法 >

子类普通属性初始化 = 子类普通代码块 > 子类构造方法

即：

类加载：静态属性初始化 > 生成对象：属性初始化 + 构造方法

第一步：静态属性初始化。第二步：属性初始化。第三步：调用构造方法生成对象。其他：(静态/普通)代码块和(静态/普通)属性初始化等价，执行顺序看在程序中的先后顺序。

代码示例如下：

```java
public class TestInit {
	public static void main(String[] args) {
		Son test = new Son();
	}

	public static class Print {
		Print(String str) {
			System.out.println(str);
		}
	}
}

public class Parent {
	Parent() {
		System.out.println("父类构造方法执行");
	}
	Print print = new Print("父类属性1初始化");
	static Print printS = new Print("静态：父类静态属性1初始化");
	{
		System.out.println("父类代码块执行");
	}
	static {
		System.out.println("静态：父类静态代码块执行");
	}
	static Print print2S = new Print("静态：父类静态属性2初始化");
	Print print2 = new Print("父类属性2初始化");
}

public class Son extends Parent {
	Son() {
		System.out.println("子类构造方法执行");
	}
	Print print = new Print("子类属性1初始化");
	{
		System.out.println("子类代码块执行");
	}
	static Print printS = new Print("静态：子类静态属性1初始化");
	static {
		System.out.println("静态：子类静态代码块执行");
	}
	static Print print2S = new Print("静态：子类静态属性1初始化");
	Print print2 = new Print("子类属性2初始化");
}

// 输入如下：
静态：父类静态属性1初始化
静态：父类静态代码块执行
静态：父类静态属性2初始化
静态：子类静态属性1初始化
静态：子类静态代码块执行
静态：子类静态属性1初始化
父类属性1初始化
父类代码块执行
父类属性2初始化
父类构造方法执行
子类属性1初始化
子类代码块执行
子类属性2初始化
子类构造方法执行
```

#### 7. 数组初始化

##### 数组初始化

和C++不同，java的数组添加了数组越界的检查，有一个固定的数组成员length。这样虽然增加了少许检查的开销，但是增强了安全性。

数组初始化的两种方法：

（1）直接赋值：

```java
int[] array = {1, new Integer(2), 3, 4, 5,};	// 1.可以自动拆装箱 2.最后的逗号可选，多了一个也没关系，相当于没有
```

（2）使用new

```java
int[] arrayA = new int[20];
Integer[] arrayB = new Integer[20];
Integer[] arrayC = new Integer[]{1,2,};// 逗号可选

System.out.println("arrayA: int[]:" + arrayA[2]);	// 0
System.out.println("arrayB: Integer[]:" + arrayB[2]);// null
System.out.println("arrayC: Integer[]:" + arrayC[2]);// 2
```

此时数组的值会自动初始化为相应的空值。数组内的对象并不初始化。另一个例子：

```java
public static class User{
    public User() {
        System.out.println("no args init method done!");
    }

    public User(String msg) {
        System.out.println("msg init method done!" + msg);
    }

}

public static void testArrayInit() {
    User[] users = new User[20];
    users[0] = new User("0: Go in array!!!");
    users[1] = new User("1: Go in array!!!");
}

// 只输出了以下内容，说明引用类型的数据不会在数组初始化时也初始化：
msg init method done!0: Go in array!!!
msg init method done!1: Go in array!!!
```

##### 可变参数列表

可变参数列表可以传0个或多个值。

底层是用数组实现的。

如果入参传的已经是数组，那就直接赋值到数组内，不再自动转换为数组。

```java
// 可变参数列表
public static void method(Object... objs){
    System.out.print("length:" + objs.length);
    for (Object obj : objs) {	//不传参也不报错，因为是 array[0]
        System.out.print(":" + obj);
    }
    
}

public static void main(String[] args) {	
    method();
    method(new Integer[]{1,6});
}
// 输出如下，可见根据入参，自动确定长度。入参是数组，则自动赋值为数组。
length:0
length:2
:1:6
```

基本类型也能用，因为底层是数组嘛

```java
public void method(int... intArgs){}
```

底层不依赖自动拆装箱。不过可以自动升为包装类。

```java
// 可变参数列表
public static void method2(int... ints){
    System.out.print("length:" + ints.length);
    System.out.println(" class:" + ints.getClass());

}

public static void method3(Integer... ints){
    System.out.print("length:" + ints.length);
    System.out.println(" class:" + ints.getClass());
}

public static void main(String[] args) {	
		method2();
		//method2(new Integer[]{1,6}); // 报错，不能用包装类代替基本类
		method3(1,2);	// 但是可以用基本类替换包装类
}
// 输出如下。可以看到，底层并没有自动拆装包。前者是基本类型，后者是引用类型。
length:0 class:class [I
length:2 class:class [Ljava.lang.Integer;
```

重载时建议只用一个可变参数列表，不然容易混淆。



##### 枚举类型

本质是类。支持switch。

#### 8.总结

Java的安全，最容易出问题的就是初始化和清理。Java初始化由构造器极其自动初始化等方法来支持，清理则是由垃圾回收器和try catch等机制来处理。这样，从这两方面来保证了Java的安全。本章也就是着重介绍了这两部分的内容。