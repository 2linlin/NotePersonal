---
title: 笔记《Oracle官方并发教程》2.线程对象
date: 2019-07-20 16:44:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

### 新建线程

两种方式：

* 继承Thread()

``` java
public class Thread1 extends Thread {
    @Override
    public void run() {
        System.out.println("---------");
    }
}
new Thread1().start();
```

* 实现Runnable()接口，并作为Thread()构造方法参数传入

```java
public class Runnable1 implements Runnable {
    @Override
    public void run() {
        System.out.println("-----Runnable1--");
    }
}
new Thread(new Runnable1()).start();
```

其实以上两种方式没有区别，因为Thread类实现了Runnable接口。Thread中的run()方法传入Runnnable对象时使用入参的run()方法，否则使用自己的run()方法：

``` java
public class Thread implements Runnable {
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }// target就是从构造方法传入的Runnable对象。
｝
```

### Thread.sleep()方法

让当前线程暂停一段时间。从而让其他线程获得cpu时间片。

由于OS的限制，休眠时间不是精确的。而且休眠时可以被interrupts终止，当其他线程中断当前线程时，sleep()方法就会抛出InterruptedException。

sleep()方法不会释放锁。代码示例如下：

```java
public class SyncTest {
    public static synchronized void lock() {
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

    public static void main(String[] args) {
        for (int i = 0; i < 500; i++) {
            new Thread(new Runnable1()).start();
        }
    }

    public static class Runnable1 implements Runnable {

        @Override
        public void run() {
            while (true) {
                System.out.println("before-------------------" + Thread.currentThread().getName());
                SyncTest.lock();
                System.out.println("after--------------------" + Thread.currentThread().getName());
            }
        }
    }
}
// 输出：
// 可以看到，每个线程的4条语句都是整体执行完的，不会有其他线程的语句中间插进来的情况。
after--------------------Thread-82
before-------------------Thread-82
)))))))))))))))))))Thread-82
)))))))))))))))))))Thread-82
)))))))))))))))))))Thread-82
)))))))))))))))))))Thread-82
after--------------------Thread-82
)))))))))))))))))))Thread-83
before-------------------Thread-82
)))))))))))))))))))Thread-83
)))))))))))))))))))Thread-83
)))))))))))))))))))Thread-83
after--------------------Thread-83
before-------------------Thread-83
// 如果lock()方法不加synchronized锁，就会像下面这样，乱七八糟的
after--------------------Thread-473
before-------------------Thread-473
)))))))))))))))))))Thread-473
before-------------------Thread-486
)))))))))))))))))))Thread-486
)))))))))))))))))))Thread-469
after--------------------Thread-469
before-------------------Thread-469
```

### 中断(Interrupts): t.interrupt()

中断是给线程的一个指示，告诉它应该停止正在做的事并去做其他事情。线程如何响应中断取决于程序员（即抛出中断异常时如何处理），我们一般直接让它终止。

通过调用线程对象的interrupt()方法就可以让它中断，但是要被中断的线程支持自己的中断时(即它自己处理中断)，才会生效。

``` java
Thread t = new Tread();
t.interrupt();	// 调用interrupt()方法发送中断信号。
```

#### 中断支持

线程如何支持自身中断呢？如果线程调用了会抛InterruptedException的方法，那么在捕获异常后，它就从run()方法返回。

``` java
for (int i = 0; i < importantInfo.length; i++) {
    // Pause for 4 seconds
    try {
       Thread.sleep(4000);
    } catch (InterruptedException e) {
       // We've been interrupted: no more messages.
      return;
    ｝
 }
 // Print a message
 System.out.println(importantInfo[i]);
}
```

许多会抛InterruptedException异常的方法(如sleep()），被设计成接收到中断后取消它们当前的操作，并在立即返回。sleep()方法是native的，未能看到处理细节。

``` java
public static native void sleep(long millis) throws InterruptedException;
```

那么，如果没有会抛InterruptedException的方法，我们怎么支持中断呢？我们可以用Thread.interrupted()来检测：

``` java
for (int i = 0; i < inputs.length; i++) {
    heavyCrunch(inputs[i]);
    if (Thread.interrupted()) {	// 中断标记位会被清除
        // We've been interrupted: no more crunching.
        throw new InterruptedException();   
        // 或者直接 return;
    }
}
```

#### 中断状态标记: Thread.interrupted(), t.isInterrupted()

中断机制通过使用称为中断状态的内部标记来实现。

调用**threadObject.interrupt()**设置这个标记。

``` java
// Thread源码
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}

private native void interrupt0();
```

当线程通过调用静态方法**Thread.interrupted()**检测中断时，中断状态会被清除。

``` java
// Thread源码    
public static boolean interrupted() {
        return currentThread().isInterrupted(true);
}
public static native Thread currentThread();
private native boolean isInterrupted(boolean ClearInterrupted);
```

非静态的**isInterrupted()**方法被线程用来检测其他线程的中断状态，不改变中断状态标记。

``` java
// Thread源码 
public boolean isInterrupted() {
        return isInterrupted(false);
}
private native boolean isInterrupted(boolean ClearInterrupted);
```

按照惯例，任何通过抛出一个InterruptedException异常退出的方法，当抛该异常时会清除中断状态。

### Joins: t.join(), t.join(time)

join()方法可以让线程等待另一个线程执行完毕。

如果t线程正在执行，那么`t.join();`会让当前线程暂停，直到t执行完毕。

加入时间参数可以让当前线程只等待n毫秒，然后继续执行当前线程。`t.join(1000);`

和sleep(time)方法一样，等待的时间不是绝对精确的。

和sleep()方法一样，join()方法响应中断并在中断时抛出InterruptedException。

``` java
// Thread源码
public final void join() throws InterruptedException {
    join(0);
}

public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}

public final native boolean isAlive();
public final native void wait(long timeout) throws InterruptedException;
```

### 综合代码示例

MessageLoop现场将会打印一系列的信息。如果中断在它打印完所有信息前发生，它将会打印一个特定的消息并退出。

``` java
public class SimpleThreads {

    // Display a message, preceded by
    // the name of the current thread
    static void threadMessage(String message) {
        String threadName =
                Thread.currentThread().getName();
        System.out.format("%s: %s%n",
                threadName,
                message);
    }

    private static class MessageLoop
            implements Runnable {
        public void run() {
            String importantInfo[] = {
                    "Mares eat oats",
                    "Does eat oats",
                    "Little lambs eat ivy",
                    "A kid will eat ivy too"
            };
            try {
                for (int i = 0;
                     i < importantInfo.length;
                     i++) {
                    // Pause for 4 seconds
                    Thread.sleep(4000);
                    // Print a message
                    threadMessage(importantInfo[i]);
                }
            } catch (InterruptedException e) {
                threadMessage("I wasn't done!");
            }
        }
    }

    public static void main(String args[])
            throws InterruptedException {

        // Delay, in milliseconds before
        // we interrupt MessageLoop
        // thread (default one hour).
        long patience = 1000 * 10;

        // If command line argument
        // present, gives patience
        // in seconds.
        if (args.length > 0) {
            try {
                patience = Long.parseLong(args[0]) * 1000;
            } catch (NumberFormatException e) {
                System.err.println("Argument must be an integer.");
                System.exit(1);
            }
        }

        threadMessage("Starting MessageLoop thread");
        long startTime = System.currentTimeMillis();
        Thread t = new Thread(new MessageLoop());
        t.start();

        threadMessage("Waiting for MessageLoop thread to finish");
        // loop until MessageLoop
        // thread exits
        while (t.isAlive()) {
            threadMessage("Still waiting...");
            // Wait maximum of 1 second
            // for MessageLoop thread
            // to finish.
            t.join(1000);
            t.interrupt();
            if (((System.currentTimeMillis() - startTime) > patience)
                    && t.isAlive()) {
                threadMessage("Tired of waiting!");
                t.interrupt();
                System.out.println("t.isAlive():" + t.isAlive()); // true
                // Shouldn't be long now
                // -- wait indefinitely
                t.join(); // 调用t.interrupt()本质上只是设定了t线程的中断标志位，理论上还需要等待某个间隔之后t线程才能检测到中断，才会终止。所以这里再join(等待一下t线程)以便确认其结束
                System.out.println("t.isAlive():" + t.isAlive()); // false
            }
        }
        threadMessage("Finally!");
    }
}
```

参考资料

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>