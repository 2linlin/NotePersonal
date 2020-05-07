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

##### 2.1.定义任务Runnable

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

线程池用`Executor`的静态方法`newXXXXPool()`创建，常用的有以下几种：

- CachedThreadPool：为每个任务都创建一个线程。
- FixedThreadPool：给定n个线程。
- SingleThreadExecutor：只有一个线程。提交多个任务的话，它会按照提交顺序排队执行线程。**执行完一个线程，才执行下一个**。它自己会维护一个隐式的队列。
- WorkStealingPool
- ScheduledThreadPoolExecutor

##### 2.4.定义有返回值的任务Callable

`runnable.run()`任务没有返回值。如果希望有返回值，那么用`callable.call()`(Callable<T>和public T call())实现。

这个接口是JavaSE5引入的。只能使用`executorService.submit()`调用（区别于`executorService.execute()`）。

`submit()`返回`Future<T>`对象，其中`T`是`call()`方法的返回值。

用法示例：

```java
public class TaskWithResult implements Callable<String> {
    private int taskId;

    public TaskWithResult(int id) {
        this.taskId = id;
    }

    public String call() throws Exception {
        return "callable task id: " + taskId;
    }
}

public class UseCallable {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();

        List<Future<String>> futures = new ArrayList<Future<String>>();

        for (int i = 0; i < 5; i++) {
            futures.add(executorService.submit(new TaskWithResult(i)));
        }

        futures.stream().forEach(future -> {
            try {
                System.out.println(future.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
        executorService.shutdown();
    }
}
```

其中，调用`Future.get()`，`get`将阻塞，直到有结果出来。

也可以调用`future.get(long)`，超时直接返回。

还可以调用`future.isDone()`，看下有没有执行完了。

`Future`接口源码：

```java
public interface Future<V> {
	boolean cancel(boolean mayInterruptIfRunning);
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
    
    boolean isCancelled();
    boolean isDone();
}
```

##### 2.5 Thread.sleep()方法

`Thread.field()`方法是让出当前CPU使用权。`sleep()`则是当前线程阻塞等待一会儿。

由于`sleep()`使线程进入了等待状态，所以可以抛出`InterruptedException`异常。

示例代码：

```java
public class UseThread {
    public static void main(String[] args) {
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < 5; i++) {
            exec.execute(new Liftoff());
            System.out.println("Waiting for Liftoff...");
        }
        exec.shutdown();
    }
}

public class Liftoff implements Runnable {
    private static int taskCount = 0;   // 用来记载新建的线程数量
    private final int id = taskCount++;

    protected int countDown = 10;

    public void run() {
        while (countDown-- > 0) {
            System.out.print("#" + id + "(" + (countDown > 0 ? countDown : "Liftoff!") + "), ");
            try {
                Thread.sleep(1000); // 等待一会儿
                // TimeUnit.SECONDS.sleep(1); // 或者可用Java5新引入的这个类，显示指定等待单位
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

##### 2.6 设置线程优先级

tips: 可以用`Thread.currentThread()`获取当前线程对象。

用`thread.getPriority()`，`thread.setPriority(int)`来获取/设置线程优先级。

优先级高只能说该线程可能会更优先，并不能一定保证它最优先执行。

`JDK`有10个优先级，而不同的操作系统优先级不定，它们一般不能很好的映射。唯一能靠谱点的用法是，仅用`Thread.MAX_PRIORITY`、`Thread.NORM_PRIORITY`、`Thread.MIN_PRIORITY`来设置优先级。

以下为示例代码：

```java
public class TreadPriority implements Runnable {
    private int priority;
    private volatile double dub;    // 加上volatile防止编译器优化

    public TreadPriority(int priority) {
        this.priority = priority;
    }

    @Override
    public void run() {
        Thread.currentThread().setPriority(priority);
        // 耗时运算，看是不是优先级高的先算完
        for (int i = 0; i < 100000; i++) {
            dub += Math.PI + Math.E / (double)i;
            if (i % 1000 == 0) {
                Thread.yield(); // 定期声明让出控制权，重新按优先级分配
            }
        }
        System.out.println(Thread.currentThread());
    }

    public static void main(String[] args) {
        ExecutorService  executorService = Executors.newCachedThreadPool();

        for (int i = 0; i < 10; i++) {
            executorService.execute(
                    new TreadPriority(i % 2 == 0 ? Thread.MIN_PRIORITY : Thread.MAX_PRIORITY));
        }
        executorService.shutdown(); // 记得关闭线程池
    }
}
// 结果：
Thread[pool-1-thread-2,10,main]
Thread[pool-1-thread-8,10,main]
Thread[pool-1-thread-10,10,main]
Thread[pool-1-thread-6,10,main]
Thread[pool-1-thread-4,10,main]
Thread[pool-1-thread-3,1,main]
Thread[pool-1-thread-1,1,main]
Thread[pool-1-thread-5,1,main]
Thread[pool-1-thread-9,1,main]
Thread[pool-1-thread-7,1,main]
```

可以看到，优先级高的5个线程都是最先完成运算的。

##### 2.7 Thread.yeild()方法

这个方法前面已经用过了。注意几点：

- `yeild()`只是声明自己让出控制权，只是对调度器的一种建议。
- 调用`yeild()`时，是建议**具有相同优先级**的其他线程可以运行。

一般对于重要的控制，都不会依赖于`yeild()`。

##### 2.8 后台线程

后台线程就是一直在后台跑，**直到所有线程退出，它才退出的线程**。通过在线程启动之前调用`thread.setDaemon()`方法，把一个线程设置为后台线程。

显示的把一个线程设置成后台线程。代码示例：

```java
public class DaemonThread implements Runnable {

    @Override
    public void run() {
        while (true) {
            try {
                Thread.sleep(100L);
                System.out.println(Thread.currentThread() + " " + this);
            } catch (InterruptedException e) {
                System.out.println("Sleep interruptered.");
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 5; i++) {
           Thread daemonThread = new Thread(new DaemonThread());
           daemonThread.setDaemon(true);
           daemonThread.start();
        }
        System.out.println("All daemons started.");
        Thread.sleep(200L);	// 如果这里时间改短，会发现后台线程可能不会完全启动
    }
}
// 执行结果
All daemons started.
Thread[Thread-1,5,main] DaemonThread@45f342d8
Thread[Thread-0,5,main] DaemonThread@994e424
Thread[Thread-2,5,main] DaemonThread@2d58251c
Thread[Thread-4,5,main] DaemonThread@45194fb9
Thread[Thread-3,5,main] DaemonThread@6bf344b9
```

通过创建后台线程工厂，将线程工厂作为入参传给Executor创建后台线程。代码示例：

```java
public class DaemonThreadExecutor {
    // 后台线程工厂
    public static class DaemonThreadFactory implements ThreadFactory {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setDaemon(true);
            return t;
        }
    }

    public static class RunnableR implements Runnable {
        @Override
        public void run() {
            try {
                Thread.sleep(100L);
                System.out.println(Thread.currentThread() + " " + this);
            } catch (InterruptedException e) {
                System.out.println("Interrupted");
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ExecutorService execService = Executors.newCachedThreadPool(
                new DaemonThreadFactory());
        for (int i = 0; i < 5; i++) {
            execService.execute(new RunnableR());
        }
        System.out.println("All daemons started");
        Thread.sleep(400L);// 如果这里时间改短，会发现后台线程可能不会完全启动
    }
}
```

由于传入了线程工厂，线程池的构造方法被重写了。源码如下：

```java
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

**可以通过调用`thread.isDaemon()`查看这个线程是不是后台线程。后台线程的所有子线程都是后台线程。**

```java
public class DaemonRun implements Runnable {
    private Thread[] threads = new Thread[5];
    @Override
    public void run() {
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new RunnableR());
            threads[i].start();
        }
        for (int i = 0; i < threads.length; i++) {
            System.out.println("threads[" + i + "].isDaemon():" + threads[i].isDaemon());
        }
        while (true) {
            Thread.yield();
        }
    }

    static class RunnableR implements Runnable {
        @Override
        public void run() {
            while (true) {
                Thread.yield();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(new DaemonRun());
        t.setDaemon(true);
        t.start();
        TimeUnit.SECONDS.sleep(1L);
    }
}
// 输出：
threads[0].isDaemon():true
threads[1].isDaemon():true
threads[2].isDaemon():true
threads[3].isDaemon():true
threads[4].isDaemon():true
```

**在其他线程结束后，JVM会立即退出后台线程，不会执行`finall`字句里面的代码。**

```java
public class DaemonFinally {
    public static class RunnableR implements Runnable {
        @Override
        public void run() {
            try {
                System.out.println("Runnable run");
                Thread.sleep(100L);
            } catch (InterruptedException e) {
                System.out.println("Be interrupted");
            } finally {
                System.out.println("This should always execute.");
            }
        }
    }

    public static void main(String[] args) {
        Thread t = new Thread(new RunnableR());
        t.setDaemon(true);  // 如果把这行去掉，就会执行finally里面的字句
        t.start();
        System.out.println("Main execute finished");
    }
}
// 输出：
Main execute finished
Runnable run
```

