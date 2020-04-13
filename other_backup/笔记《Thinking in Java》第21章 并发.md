---
title: 笔记《Thinking in Java》第21章 并发
date: 2019-12-09 21:44:16
categories:
- JavaSE
- Java基础
tags:
- 学习笔记
- Java基础
top:
---

[TOC]

### 第21章 并发

基本的Web类库：Servlet具有天生的多线程性。因为web服务员一般是有多个处理器的，而并发可以充分地使用它们。充分利用多CPU的性能，这就是并发的意义。

#### 1.并发可以用来干啥

##### 1.1.并发可以提高程序的执行速度

在多处理器上使用并发，可以极大的提升吞吐量。在单处理器上使用，也能提升性能！

**<u>并发还可以提高单处理器上程序的性能！</u>**这个听起来很奇怪，按理说，单处理器用并发，是要多花费上下文切换的时间的，比起单线程，速度应该更慢才对。其实关键词是：**阻塞**。是这样的，把程序分成多块并发执行，那么其中的某一块阻塞了，其他部分还可以继续推荐，不会因为阻塞了某一个地方而导致整个程序都处于阻塞状态。所以如果有阻塞的话，单处理器我们用并发也可以提升程序速度。

实现并发最直接的方式就是使用操作系统级别的进程。因为进程是在它自己的地址空间自己玩的，所以不会有共享资源冲突的问题。不过Java用的是线程级别的并发，线程间内存和I/O都是共享的，要考虑资源冲突问题。

一些语言是设计成可以把并发任务隔离的，这种语言一般叫做***函数型语言***。这种语言，每个函数的调用都不会产生副作用(所以也不能干涉其他函数)，所以可以当作独立的任务来驱动。像Erlang就是这种语言，它还包含了一套任务间通信的安全机制。如果程序某些地方要大量使用并发的话，我们可以用Erlang来搞定。例如RabbitMQ。

Java是在顺序型语言的基础上提供线程支持的。它的并发是用线程机制来实现——在单一的进程中创建任务。这样做的好处是，不要求OS一定要支持分叉外部进程的机制。因为有些OS是不支持多任务的，所以Java使用线程机制才能实现“编写一次、到处运行”。

##### 1.2.并发可以让代码设计更优雅

在有些类型的问题下，并发可以大幅度简化程序设计。比如说，在CPU上进行仿真，门、石头...都要有自己的想法，都可以单独执行。不用并发设计的话，处理起来很麻烦，会疯掉的。

多线程系统对线程数量是有限制的，一般是几十或者几百。在Java中的话，依赖于Java版本。总之就是，假设你仿真需要几万个线程，这个时候，系统限制的几百的个线程数量肯定是不够的。

怎么解决这个问题呢？用***协作多线程***。Java的线程是抢占式的，线程会被周期性地中断。但是在协作式系统中，每个任务是自动放弃控制的，这个就要求程序员要在每个任务中插入某种类型的让步语句。协作式系统的优势有两个，一是上下文切换成本很低，二是理论上来说线程数没有限制。

#### 2.Java的线程机制

使用线程机制的好处就是，它是对操作系统透明的。如果性能不够用，那么加一个多处理器，就很容易加快程序速度。多CPU的系统就是要玩多线程才有意思啊。

##### 2.1.定义任务

就是定义多线程执行的任务，直接实现`Runnable`就行：

```java
// 一个发射定时器，从10倒数到0，然后发射
public class Liftoff implements Runnable {
    private static int taskCount = 0;   // 用来记载新建的线程数量
    private final int id = taskCount++;

    protected int countDown = 10;

    public void run() {
        while (countDown-- > 0) {
            System.out.print("#" + id + "(" + (countDown > 0 ? countDown : "Liftoff!") + "), ");
            Thread.yield(); // 主动放弃当前cpu使用权
        }
    }
}
```

##### 2.2.使用Thread执行任务

直接实现`Runnable`接口。然后作为`Thread`的构造函数参数输入就行了。

或者用一个子类继承`Thread`，重写`run`方法也行。

两种方式是等效的，看`Thread`源代码就知道了，`Thread`也是实现了`Runnable`接口的：

```java
public class Thread implements Runnable {
    private Runnable target;
    
    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
    
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        ...
        this.target = target;
        ...
    }
    
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
}
```

下面是用最原始的方式实现多线程的示例，也就是直接用`Runnable`传入`Thread`构造方法新建线程：

关键代码：`new Thread(new Liftoff()).start()`

```java
public class UseThread {
    public static void main(String[] args) {
        for (int i = 0; i < 5; i++) {
            new Thread(new Liftoff()).start();	// 关键代码
            System.out.println("Waiting for Liftoff...");
        }
    }
}
```

##### 2.3.使用Executor执行任务

Executor是JUC包提供的线程池接口。这样就不用手动的管理线程周期啦(不用手动start()等)，直接把任务丢进去执行就行。使用示例：

关键代码：`exec.execute(new Liftoff())`，`exec.shutdown()`

```java
public class UserThread {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < 5; i++) {
            exec.execute(new Liftoff()); // 关键代码
            System.out.println("Waiting for Liftoff...");
        }
        exec.shutdown();
    }
}
```

`ExecutorService`实现了`Executor`接口。`shutdown()`方法被调用后，它之前的任务继续执行直至结束。

线程池用`Executor`的静态方法`newXXXXPoll()`创建，常用的有以下几种：

- CachedThreadPool：为每个任务都创建一个线程。
- FixedThreadPool：给定n个线程。
- SingleThreadExecutor：只有一个线程。提交多个任务的话，它会按照提交顺序排队执行线程。**执行完一个线程，才执行下一个**。它自己会维护一个隐式的队列。
- WorkStealingPool
- ScheduledThreadPoolExecutor

##### 2.4.定义有返回值的任务

未完待续...

