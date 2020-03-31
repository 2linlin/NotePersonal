---
title: 笔记《Oracle官方并发教程》3.同步
date: 2019-07-20 17:11:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

线程间的通信主要有两种方式：共享域、引用相同对象。但是这样就会引发两种错误：**线程干扰、内存一致性错误**。然后我们用同步来解决这个问题。

本小节讨论以下问题：

- **线程干扰**讨论了当多个线程访问共享数据时错误是怎么发生的。
- **内存一致性错误**讨论了不一致的共享内存视图导致的错误。
- **同步方法**讨论了一种能有效防止线程干扰和内存一致性错误的常见做法。
- **内部锁和同步**讨论了更通用的同步方法，以及同步是如何基于内部锁实现的。
- **原子访问**讨论了不能被其他线程干扰的操作的总体思路。

### 线程干扰

大概的意思就是，两个线程对同一个数据进行交替操作，互相影响了对方的结果。如下，A线程调用increment()，B线程调用decrement()。两个线程同时操作c的值，操作完毕后可能会覆盖对方的结果。

即使只有一条简单的指令，也可能被JVM转换成多个步骤。

```java
class Counter {
    private int c = 0;
    public void increment() {
        c++;
    }
    public void decrement() {
        c--;
    }
    public int value() {
        return c;
    }
}
// 假设线程A调用increment()的同时线程B调用decrement().如果c的初始值为0，线程A和B之间的交替执行顺序可能是下面这样：
//        线程A：获取c；
//        线程B：获取c；
//        线程A：对获取的值加1，结果为1；
//        线程B：对获取的值减1，结果为-1；
//        线程A：结果写回到c,c现在是1；
//        线程B：结果写回到c,c现在是-1；
// 线程A的结果因为被线程B覆盖而丢失了。
```

### 内存一致性错误

当不同的线程对数据的视图不一致时，就会发生内存一致性错误。具体的机制比较复杂，我们只要知道如何避免内存一致性错误就行了。

避免一致性错误，关键是理解happens-before关系。这个关系能确保一个特定语句的写内存操作，对另一个特定语句是可见的。

```java
int counter = 0;	// 语句1
counter++;			// 语句2
System.out.println(counter);// 语句3
```

以上，如果语句2和语句3在同一个线程，那么输出是1。如果不在同一个线程，那么输出可能就是0。

用 同步 就可以建立happens-before关系。见后文。

另外，我们已经见过两个可以简历happens-before关系的操作了：

* 当调用Thread.start()方法时，创建新线程之前的代码执行结果，对新线程可见。
* 在A线程中执行t.join()。t线程终止并返回时，t.线程的执行结果对A线程可见。

### 同步方法: synchronized

在方法前加synchronized即可。

```java
public class SynchronizedCounter {
    private int c = 0;

    public synchronized void increment() {
        c++;
    }

    public synchronized void decrement() {
        c--;
    }

    public synchronized int value() {
        return c;
    }
}
```

上述同步方法有两个作用：

* 防止线程干扰：**A线程用对象调用该同步方法时，所有其他调用该同步方法的线程都会被阻塞，直到A线程处理完该对象**。
* 防止内存一致性错误：**一个同步方法退出后，会自动和该对象同步方法的任意后续调用建立happens-before关系。也就确保了对象状态的改变对所有线程是可见的**。

构造方法不能同步，没有意义，因为这个对象被创建时，只有创建对象的线程能访问它。

如果一个对象对多个线程可见，且对象域上所有的操作都是由同步方法完成的(final域不用)，那么它就可以防止线程干扰和内存一致性错误。不会引起活跃度问题（后续会说）。

### 内部锁与同步

同步机制是基于其内部一个内部锁(也叫监视锁、监视器)实体来实现的。内部锁主要起两个作用：对一个对象的排他性访问。建立一种happens-before关系，从而解决可见性问题。

每个对象都有一个与之关联的内部锁。当一个线程需要排他性的访问该对象的域时，就需要请求该对象的内部锁，访问结束时释放内部锁。

**获得了内部锁就能锁定该对象的所有域，所有试图获取该对象内部锁的操作都会具有排他性。不过非同步方法还是可以随意访问域**。

**当一个线程拥有一个内部锁时，所有尝试获取该内部锁的操作都会阻塞**。**当线程释放一个内部锁时，该操作与对该内部锁的后续请求将建立happens-before关系**。

也就是说，A线程调用a方法时，B线程同时调用b方法，此时b方法会等a方法执行完再执行，不会存在两种方法交替执行的情况。

```java
package com.h2linlin.laboratory.utilbox;

import java.util.concurrent.Executor;

/**
 * @Desc XXXXX
 * @Author zh wu
 * @Date 2019/7/17 14:37
 */
public class SyncTest {
    public static synchronized void lock1() {
        System.out.println(")))))))))))))))))))" + Thread.currentThread().getName());
        try {
            Thread.sleep(600);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(")))))))))))))))))))" + Thread.currentThread().getName());
        try {
            Thread.sleep(1500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(")))))))))))))))))))" + Thread.currentThread().getName());
        try {
            Thread.sleep(400);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(")))))))))))))))))))" + Thread.currentThread().getName());
    }

    public static synchronized void lock2() {
        System.out.println("^^^^^^^^^^^^^^^^^^^" + Thread.currentThread().getName());
        try {
            Thread.sleep(600);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("^^^^^^^^^^^^^^^^^^^" + Thread.currentThread().getName());
        try {
            Thread.sleep(1500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("^^^^^^^^^^^^^^^^^^^" + Thread.currentThread().getName());
        try {
            Thread.sleep(400);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("^^^^^^^^^^^^^^^^^^^" + Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        for (int i = 0; i < 500; i++) {
            new Thread(new Runnable1()).start();
            new Thread(new Runnable2()).start();
        }
    }

    public static class Runnable1 implements Runnable {

        @Override
        public void run() {
            while (true) {
                System.out.println("before-------------------" + Thread.currentThread().getName());
                SyncTest.lock1();
                System.out.println("after--------------------" + Thread.currentThread().getName());
            }
        }
    }

    public static class Runnable2 implements Runnable {

        @Override
        public void run() {
            while (true) {
                System.out.println("before-------------------" + Thread.currentThread().getName());
                SyncTest.lock2();
                System.out.println("after--------------------" + Thread.currentThread().getName());
            }
        }
    }
}
// 结果示例，可以看到a、b方法是交替执行的，不会混到一起：
^^^^^^^^^^^^^^^^^^^Thread-987
^^^^^^^^^^^^^^^^^^^Thread-987
^^^^^^^^^^^^^^^^^^^Thread-987
^^^^^^^^^^^^^^^^^^^Thread-987
)))))))))))))))))))Thread-986
after--------------------Thread-987
before-------------------Thread-987
)))))))))))))))))))Thread-986
)))))))))))))))))))Thread-986
)))))))))))))))))))Thread-986
after--------------------Thread-986
^^^^^^^^^^^^^^^^^^^Thread-985
before-------------------Thread-986
^^^^^^^^^^^^^^^^^^^Thread-985
^^^^^^^^^^^^^^^^^^^Thread-985
^^^^^^^^^^^^^^^^^^^Thread-985
after--------------------Thread-985
before-------------------Thread-985
)))))))))))))))))))Thread-984
)))))))))))))))))))Thread-984
)))))))))))))))))))Thread-984
)))))))))))))))))))Thread-984
after--------------------Thread-984
// 如果把lock2的synchronized去掉，会产生这种情况。同步方法中的代码还是会分先后，但是会被非同步方法交替。也就是说，锁对非同步方法是没用的。
)))))))))))))))))))Thread-0
^^^^^^^^^^^^^^^^^^^Thread-1
before-------------------Thread-3
^^^^^^^^^^^^^^^^^^^Thread-3
before-------------------Thread-4
before-------------------Thread-5
^^^^^^^^^^^^^^^^^^^Thread-5
before-------------------Thread-6
before-------------------Thread-7
^^^^^^^^^^^^^^^^^^^Thread-7
before-------------------Thread-8
before-------------------Thread-9
^^^^^^^^^^^^^^^^^^^Thread-9
^^^^^^^^^^^^^^^^^^^Thread-3
^^^^^^^^^^^^^^^^^^^Thread-1
)))))))))))))))))))Thread-0
^^^^^^^^^^^^^^^^^^^Thread-7
^^^^^^^^^^^^^^^^^^^Thread-9
^^^^^^^^^^^^^^^^^^^Thread-5
)))))))))))))))))))Thread-0
^^^^^^^^^^^^^^^^^^^Thread-3
^^^^^^^^^^^^^^^^^^^Thread-1
^^^^^^^^^^^^^^^^^^^Thread-7
^^^^^^^^^^^^^^^^^^^Thread-9
^^^^^^^^^^^^^^^^^^^Thread-5
^^^^^^^^^^^^^^^^^^^Thread-3
after--------------------Thread-3
before-------------------Thread-3
^^^^^^^^^^^^^^^^^^^Thread-3
)))))))))))))))))))Thread-0
```

以上也表明，获取一个对象的内部锁时，那么不同线程的该对象的同一个同步方法之间的代码不会交替执行，不同线程的该对象的不同的同步方法代码也不会交替执行。但是，不同线程的非同步方法，是会和同步方法的代码交替执行的。总之就是，**同一个对象，同步方法的排他性适用于本同步方法，及该对象的其他同步方法。不适用于该对象的不同步方法。**由此也可以推出，*如有对象有一个变量，要想这个变量不受线程干扰，那么需要把所有访问它的方法都加上synchronized，否则虽然同步方法访问它时具有排他性，但是非同步方法是可以随意访问它的*。

然后在同步代码中调用其他同步方法可能会引起死锁的问题，后续会讲。

然后如果在同步方法里调用了自己的非同步方法，那么没影响，等价于非同步方法的代码都在同步方法里。因为同步方法的作用域覆盖了非同步方法。

上述代码例子用的是静态同步方法，但得出的结论对普通同步方法也是使用的。以下为证明代码：

````java
package com.h2linlin.laboratory.utilbox;

/**
 * @Desc XXXXX
 * @Author zh wu
 * @Date 2019/7/19 14:34
 */
public class SyncTestB {
    private int num1;
    private int num2;

    public synchronized void addA() {
        System.out.println(Thread.currentThread().getName()+"----------start");
        try {
            Thread.sleep(60);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        num1 ++;
        System.out.println(Thread.currentThread().getName()+"----------num1++");
        try {
            Thread.sleep(150);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+"----------num1++");
        try {
            Thread.sleep(40);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+"----------end");
    }

    public synchronized void addB() {
        System.out.println(Thread.currentThread().getName()+"**********start");
        try {
            Thread.sleep(60);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        num2 ++;
        System.out.println(Thread.currentThread().getName()+"**********num2++");
        try {
            Thread.sleep(150);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+"**********num2++");
        try {
            Thread.sleep(40);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName()+"**********end");
    }

    public static void main(String[] args) {
        SyncTestB syncTestB = new SyncTestB();
        for (int i = 0; i < 500; i++) {
            new Thread() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName());
                    for (int i = 0; i < 400; i++) {
                        syncTestB.addA();
                    }
                }
            }.start();
            new Thread() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName());
                    for (int i = 0; i < 400; i++) {
                        syncTestB.addB();
                    }
                }
            }.start();
        }
    }
}
// 输出示例，可以看到，不同线程同一个同步方法具有排他性(前8行),不同线程不同的同步方法也有排他性(后8行)。
Thread-932----------start
Thread-932----------num1++
Thread-932----------num1++
Thread-932----------end
Thread-930----------start
Thread-930----------num1++
Thread-930----------num1++
Thread-930----------end
Thread-929**********start
Thread-929**********num2++
Thread-929**********num2++
Thread-929**********end
````



#### 同步方法中的锁

线程调用一个同步方法时，它会自动请求该方法所在对象的内部锁。方法返回或抛出未捕获异常时，内部锁释放。

如果线程调用的是静态方法，那么自动获得该类对象的内部锁。

#### 同步块

用同步块的话要指明具体要获得内部锁的对象：

```java
public void addName(String name) {
    synchronized(this) {
        lastName = name;
        nameCount++;
    }
    nameList.add(name);
}
```

同步块可以获得更细的控制粒度。用同步方法是获取对象自己的内部锁，会锁住对象自己的所有的域。那么如果对象的两个域其实没有相关性，想分开控制，怎么办呢？就可以用同步块。以下是示例：

```java
public class MsLunch {
    private long c1 = 0;
    private long c2 = 0;
    private Object lock1 = new Object();
    private Object lock2 = new Object();

    public void inc1() {
        synchronized(lock1) {
            c1++;
        }
    }

    public void inc2() {
        synchronized(lock2) {
            c2++;
        }
    }
}
```

#### 可重入同步

线程不可以获得其他线程拥有的锁，但是可以获得自己已经拥有的锁。这样就防止了线程阻塞自己。

### 原子访问

原子操作是说，所有操作，要么全做、要目全不做，且在所有操作完成前，不会有看得见的副作用。

例如以下操作是原子的：

* 对引用变量和大部分基本类型变量（**除long和double之外**）的读写是原子的。
* 对所有声明为volatile的变量（包括long和double变量）的读写是原子的。

原子操作不用担心线程干扰，但是可能发生内存一致性错误。我们可以用volatile变量可以降低内存一致性错误的风险，因为volatile变量任意写操作，都会对后续任何对该变量的读操作建立happens-before关系。也就是说，volatile变量的修改对其他线程总是可见的。

用原子访问比用同步代码访问更高效，不过要注意避免内存一致性错误。

参考资料：

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>

