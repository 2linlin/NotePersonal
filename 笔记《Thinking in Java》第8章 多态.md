---
title: 笔记《Thinking in Java》第8章 多态
date: 2019-08-01 02:36:26
categories:
- JavaSE
- Java基础
tags:
- 学习笔记
- Java基础
top:
---

### 第八章 多态

除了继承和封装，多态通过分离作什么和怎么做，从另一个角度把接口和实现分开。多态是一项让程序员“将改变的事物与未变的事物分离开来”的技术。

**多态也称作动态绑定、后期绑定、运行时绑定。**与前期绑定相对应。将一个方法调用同一个方法主体关联起来，就被称作绑定。前期绑定时，方法的调用主体和方法是一一对应起来绑定的，C语言就只有前期绑定。**而动态绑定（多态）就是说，方法的调用主体和调用的方法要在运行时才绑定起来**，并不是一开始就绑定死的，具体来说，就是运用一种机制，以便在运行时判断对象类型，从而调用恰当的方法。

多态的基础：**Java里面除了static和final方法(private也是final方法)，所有的方法都是运行时绑定的**，也就是说，编译器不用知道对象类型，但是方法机制确能找到正确的应该调用的方法体。所以我们要做的，就仅仅是把消息丢给对象而已，让它自己去决定该怎么做。

另外，虽然说final能够使方法变成前期绑定，来提高效率，但是一般我们没必要这么做。

#### 1.多态注意事项（多态的坑）

* **私有方法并不能被继承**。在同名情况下，子类public “继承”(其实并不是继承) 父类private，向上转型后，执行的是父类方法。

```java
public static void main(String[] args) {
    Parent p = new Son();
    p.method();	// parent method!
}

public static class Parent {
    private void method() {	// 如果将此处改为public，则子类就继承了，main就是输出son method
        System.out.println("parent method!");
    }
}

public static class Son extends Parent {
    public void method() {	// 并不是继承，因为父类是private。这里加@Override会报错
        System.out.println("son method!");
    }
}
```

* **多态不适用于静态方法**（因为静态方法是与类绑定，而不是单个对象绑定的，所以与对象无关）。只有普通方法可以是多态的。

```java
public static class Parent {
    public static String staticGet() {
        return "Parent: staticGet()";
    }
    public String dynamicGet() {
        return "Parent: dynamicGet()";
    }
}

public static class Son extends Parent {
    public static String staticGet() {
        return "Son: staticGet()";
    }
    public String dynamicGet() {
        return "Son: dynamicGet()";
    }
}

public static void main(String[] args) {
    Parent p = new Son();	// 向上转型
    System.out.println("p.staticGet():	" + p.staticGet());	// Parent: staticGet() ：静态方法不自持多态(即运行期绑定)
    System.out.println("p.dynamicGet():	" + p.dynamicGet());// Son: dynamicGet()
}
```



* **多态也不适用于域**。如果直接访问某个域，这个域是在编译期就解析的了，并不会等到运行时。不过我们一般是直接将域设为private，然后通过方法访问。所以这种情况很少遇到。

```java
public static class Parent {
    int field = 1;
    public int getField() {return field;}
}

public static class Son extends Parent {
    int field = 2;
    public int getField() {return field;}
    public int getSuperField() {return super.field;}
}

public static void main(String[] args) {
    Parent p = new Son();	// 向上转型
    System.out.println("p.field:" + p.field);// p.field:1，这里就是混淆的地方，直接访问是父类的，多态并不适用于域
    System.out.println("p.getField():" + p.getField());//p.getField():2，用方法访问则是自己的

    Son s = new Son();
    System.out.println("s.field:" + s.field);			// s.field:2
    System.out.println("s.getField():" + s.getField());	// s.getField():2
    System.out.println("s.getSuperField():" + s.getSuperField());// s.getSuperField():1
}
```

#### 2.构造器和多态

重温一遍类初始化顺序：

**父类静态初始化 -> 子类静态初始化 -> 父类一般初始化 -> 父类构造方法 -> 子类一般初始化 -> 子类构造方法**

##### 继承与清理

组合和继承来创建新类时，不用自己清理。如果确实要清理，可使用使用两种方式：

1.使用自造的dispose()方法清理。注意：子类的dispose()内部一定要调用父类的dispose()。清理时，清理字段的顺序和字段声明的顺序相反。

2.当涉及多个引用时，使用引用计数器。对象的id声明为final，用static long counter来计数即可。为0时dispose()。

##### 构造器内部的多态坑

原则：**构造方法里面，唯一能安全调用的方法只有final方法**（也包括隐式的final方法private方法）。

为啥呢：因为多态的大。<u>*父类里面调用了方法，会多态成子类的方法*</u>。并且，<u>如果子类方法里用到了子类的属性，这个是子类的属性也就还是未初始化的属性</u>*！

```java
	public static class Parent {
		void say() {System.out.println("Parent.say()");}
		Parent() {
			System.out.println("Parent(): before say()");
			say();
			System.out.println("Parent(): after say()");
		}
	}
	public static class Son extends Parent {
		private String say = "I'm son......";
		void say() {System.out.println("Son.say():" + say);}
		Son() {
		}
	}
	public static void main(String[] args) {
		new Son();
        // 输出：
        // Parent(): before say()
		// Son.say():null	// 很显然，这里的方法多态调用为子类方法了。而且这时子类属性还未初始化！！
		// Parent(): after say()
	}
```

#### 3.协变返回类型

一句话：多态允许子类方法的返回值是父类方法返回值的子类

```java
// 父类方法
public Fruit method() {...}
// 子类方法，使用Apple返回值是合法的。这个是java1.5之后的改动，之前是必须返回Fruit
public Apple method() {...}
```

#### 4.使用多态的一些Tips

##### 用继承、组合多态实现状态模式

技巧：用继承来实现同一个方法的不同逻辑，并用组合表达字段的状态。再用多态使两者结合起来，多态使状态的改变产生了行为(方法逻辑)的改变。有点绕，很常用，一看源码就明白：

```java
	public static class Actor {
		public void act() {}
	}

	public static class HappyActor extends Actor {
		public void act() {System.out.println("Happy actor!");}
	}

	public static class SadActor extends Actor {
		public void act() {System.out.println("Sad actor!");}
	}

	public static class Stage {
		private Actor actor = new HappyActor();
		public void change() {actor = new SadActor();}
		public void pergormPlay() {actor.act();}
	}

	public static void main(String[] args) {
		Stage stage = new Stage();
		stage.pergormPlay();
		stage.change();
		stage.pergormPlay();
		// 输出:
		// Happy actor!
		// Sad actor!
	}
```

##### 向下转型不安全

```java
// 后续补代码
	public static class Useful {
		public void a() {};
		public void b() {};
	}
	
	public static class MoreUseful extends Useful {
		public void a() {};
		public void b() {};
		public void c() {};
	}
	
	public static void main(String[] args) {
		Useful[] x = {
			new Useful(),
			new MoreUseful()
		};
		x[0].a();
		x[1].b();
		((MoreUseful)x[1]).c();	// 正常执行
		((MoreUseful)x[0]).c();	// 报错：java.lang.ClassCastException
	}
```