---
title: 笔记《Thinking in Java》第9章 接口
date: 2019-08-06 23:34:26
categories:
- JavaSE
- Java基础
tags:
- 学习笔记
- Java基础
top:
---

### 第九章 接口

#### 1.抽象类和抽象方法

包含抽象方法的类叫抽象类。（也就是如果有abstract方法，类上不加abstract编译器会报错）

编译器保证了抽象方法一定会被实现：继承一个抽象类时，不实现抽象方法会强制为还是抽象类。因为用含有抽象方法的类来生成对象是没有意义的。

**我们也可以定义不包含抽象方法的抽象类，因为有时候不想让一个类生成对象。**

```java
	public static abstract class Parent {
		public void method() { System.out.println("Parent do something");}
	}
	
	public static class Son extends Parent {}
	
	public static abstract class AbstractSon extends Parent {}
	
	public static void main(String[] args) {
		// new Parent().method();		// 编译器报错，实现了我们不想让抽象类创建新对象的目的
		new Son().method();				// 输出：Parent do something
		// new AbstractSon().method();	// 报错
	}
```

抽象类可以让公共方法沿继承层次向上移动，用来重构也很爽。

练习三：解释下列现象。

解答：多态，又叫运行时方法绑定。还是多态的坑：在父类的构造方法里调用方法，会调用到子类重写的方法。如果这个方法里还用到了子类的类变量，那么这个类变量也是还没有初始化赋值的。

```java
public class Test610 {
	public static abstract class Parent {
		public Parent() {
			this.print();	// 调用的是子类重写后的方法。原因：多态的本质：方法运行时(根据对象)绑定。
		}
		
		public abstract void print();
	}
	
	public static class Son extends Parent {
		private int i = 5;
		@Override
		public void print() {
			System.out.println("Son print():" + i);
		}
	}
	
	public static void main(String[] args) {
		new Son().print();				
		// 输出：
		// Son print():0
		// Son print():5
	}
}
```

#### 2.接口

新版本的java为接口提供了default方法。

接口的类变量是隐式的static final的，接口的方法是隐式的public的。

引申：策略设计模式：创建一个能根据所传递的参数对象的不同，而具有不同行为的方法。即，入参用接口代替，根据不同的实现类获得不同的执行方法。通过这种方法，就可以让任何类对该方法进行适配。

#### 3.通过继承来扩展接口

接口C可以同时继承接口A和接口B。当然，实现接口C时，就要实现接口A、B、C中的所有方法。

```java
interface A{}
interface B{}
interface C extends A, B {}
```

这里有一个小陷阱，继承的类和实现的接口，如果方法名和入参相同，而返回值不同，这个时候是无法区分两个方法的。这个时候编译器就报错：

```java
interface I1   {void f();}
interface I2   {int f(int i);}
interface I3   {int f();}
class C {public int f() {return 1;}}

class C1 extends C implements I3 {public int f() {return 2;}} // 没有问题，相同的方法
class C2 extends C implements I1 {} // 编译器报错，方法入参和名字一样，但返回值不一样，无法区分
interface I4 implements I1, I3 {}   // 编译器报错，方法入参和名字一样，但返回值不一样，无法区分

```

#### 4.嵌套接口

接口可以使用类似内部类的嵌套，这种情况下还可以设置为private的。而且private的情况下还能被实现为public类。

```java
class clazz {
    interface B {}
    public class BImpl implements B {}
    private class BImpl2 implements B {}
    
    public interface C {}
    class CImpl implements C {}
	private class CImpl2 implements C {}
    
    private interface D {}		// 注意这里，类里可以嵌套private的interface
    private class DImpl implements D {}	// 这里可以被privae的类实现，没毛病
    public class DImpl2 implements D {} // 这里居然可以被public的类实现。但是确只能类内自己用。
    
    public D getD {return new DImpl2();}
    private D dRef;
    public void receiveD(D d) {dRef = d;}
}
interface E {
    interface G {}
    public interface H {}
    
    void g();
    
    // private interface() I {} 编译器报错，接口里不能嵌套private的interface
}
public class NestingInterfaces {
    public class BImpl implements clazz.B {}
    class CImpl implements clazz.C {}
    // class DImpl implements clazz.D {} 编译器报错，不能实现private的interface
    class EImpl implements E {}	// 实现某个接口时，无需实现它内部嵌套的接口
    class EGImpl implements E.G {}
    class EImpl2 implements E {class EG implements E.G {}}\
        
    public static void main(String[] args) {
        Clazz clazz = new Clazz();
        // Clazz.D clazzD = clazz.getD(); 编译器报错，调用不到Clazz.D
        // Clazz.DImpl2 di2 = clazz.getD();
        // clazz.getD().f(); 使用不了里面的方法
        // private interface的public实现类，只能由clazz对象或另一个clazz对象来使用。
        // 所以尽管public的class“实现”了一个private的interface，但实际会有限制，我们并没有实现它。
        Clazz clazz2 = new Clazz();
        clazz2.receiveD(clazz.getD());	
    }
}
```



#### 5.接口与工厂

下面是使用接口实现工厂方法的示例：

```java
// ----------
interface Service {
    void method1();
    void method2();
}
interface ServiceFactory {
    Service getService();
}
// ----------
class Impl1 implements Service {
    void method1() {System.out.println("Impl1:method1()");}
    void method2() {System.out.println("Impl1:method2()");}
}
class Impl1Factory implements ServiceFactory {
    Service getService() {return new Impl1();}
}
// ----------
class Impl2 implements Service {
    void method1() {System.out.println("Impl2:method1()");}
    void method2() {System.out.println("Impl2:method2()");}
}
class Impl2Factory implements ServiceFactory {
    Service getService() {return new Impl2();}
}
// ----------
public class Factories {
    // 策略模式
    public static void serviceConsumer(ServiceFactory factory) {
        Service s = factory.getService();
        s.method1();
        s.method2();
    }
    public static void main() {
        serviceConsumer(new Impl1Factory());
        serviceConsumer(new Impl2Factory());
    }
    // 输出
    // Impl1:method1()
    // Impl1:method2()
    // Impl2:method1()
    // Impl2:method2()
}
```