---
title: 笔记《Oracle官方并发教程》5.GuardedBlocks
date: 2019-07-21 14:29:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

多线程之间需要协同工作，最常见的方式就是Guarded Blocks。方法是，循环检查一个条件，直到条件发生变化后，跳出循环继续执行。在使用Guarded Blocks时要注意以下步骤：

### 最简单的方法：循环检测

例如，如果guardedJoy()方法必须要等待另一线程为共享变量joy设值才能继续执行，那么理论上我们只要用一个循环来不断监测joy的状态就行了：

``` java
public void guardedJoy() {
    // Simple loop guard. Wastes
    // processor time. Don't do this!
    while(!joy) {}
    System.out.println("Joy has been achieved!");
}
```

不过上面这种方法，太浪费资源。

### 更高效的方法：wait()、notifyAll()

更高效的方法是，用Object.wait把当前线程挂起，然后直到另一个线程发起事件通知（尽管通知的事件不一定是当前线程等待的事件）。

相比于Thread.sleep()方法不会释放锁，object.wait()是释放掉锁的。

``` java
public synchronized void guardedJoy() {
    // This guard only loops once for each special event, which may not
    // be the event we're waiting for.
    while(!joy) {
        try {
            wait();
        } catch (InterruptedException e) {}
    }
    System.out.println("Joy and efficiency have been achieved!");
}
```

注意：wait()方法是在循环里调用的。

wait()方法和其他可以暂停线程执行的方法一样（例如sleep(), join()），会抛出InterruptedException。

为啥上面的`guardedJoy()`是用`synchronized`修饰的呢？假设d是调用wait()的对象，那么当一个线程调用d.wait()的时候，它必须要有d的内部锁，不然会抛异常。那么获取d的内部锁最简单的方法，就是在一个同步方法里调用wait()啦。

当一个线程调用wait()的时候，它释放锁并挂起。然后另外一个线程请求并获得这个锁并调用`Object.notifyAll`通知所有等待该锁的线程。

```java
public synchronized notifyJoy() {
    joy = true;
    notifyAll();
}
```

当第二个线程释放这个锁后，第一个线程再次请求该锁，从wait方法返回并继续执行。

注意：还有另外一个通知方法，notify()，它只会唤醒一个线程。不过唤醒是随机的，所以一般只在大规模并发应用中使用(系统有大量相似线程，哪个执行无所谓)。

### 生产者消费者代码示例

现在我们使用Guarded blocks创建一个生产者/消费者应用。生产者生产数据、消费者消费数据。两个线程通过共享对象通信。规则：生产者发布数据前，消费者不能读取数据。消费者没有读取旧数据前，生产者不能发布新数据。

``` java
// 在下面的例子中，数据通过Drop对象共享的一系列文本消息
public class Drop {
    // Message sent from producer
    // to consumer.
    private String message;
    // True if consumer should wait
    // for producer to send message,
    // false if producer should wait for
    // consumer to retrieve message.
    private boolean empty = true;

    public synchronized String take() {
        // Wait until message is
        // available.
        while (empty) {
            try {
                wait();
            } catch (InterruptedException e) {}
        }
        // Toggle status.
        empty = true;
        // Notify producer that
        // status has changed.
        notifyAll();
        return message;
    }

    public synchronized void put(String message) {
        // Wait until message has
        // been retrieved.
        while (!empty) {
            try {
                wait();
            } catch (InterruptedException e) {}
        }
        // Toggle status.
        empty = false;
        // Store message.
        this.message = message;
        // Notify consumer that status
        // has changed.
        notifyAll();
    }
}

// Producer是生产者线程，发送一组消息，字符串DONE表示所有消息都已经发送完成。为了模拟现实情况，生产者线程还会在消息发送时随机的暂停。
import java.util.Random;

public class Producer implements Runnable {
    private Drop drop;

    public Producer(Drop drop) {
        this.drop = drop;
    }

    public void run() {
        String importantInfo[] = {
            "Mares eat oats",
            "Does eat oats",
            "Little lambs eat ivy",
            "A kid will eat ivy too"
        };
        Random random = new Random();

        for (int i = 0;
             i &lt; importantInfo.length;
             i++) {
            drop.put(importantInfo[i]);
            try {
                Thread.sleep(random.nextInt(5000));
            } catch (InterruptedException e) {}
        }
        drop.put("DONE");
    }
}

// Consumer是消费者线程，读取消息并打印出来，直到读取到字符串DONE为止。消费者线程在消息读取时也会随机的暂停。
import java.util.Random;

public class Consumer implements Runnable {
    private Drop drop;

    public Consumer(Drop drop) {
        this.drop = drop;
    }

    public void run() {
        Random random = new Random();
        for (String message = drop.take();
             ! message.equals("DONE");
             message = drop.take()) {
            System.out.format("MESSAGE RECEIVED: %s%n", message);
            try {
                Thread.sleep(random.nextInt(5000));
            } catch (InterruptedException e) {}
        }
    }
}

// ProducerConsumerExample是主线程，它启动生产者线程和消费者线程。
public class ProducerConsumerExample {
    public static void main(String[] args) {
        Drop drop = new Drop();
        (new Thread(new Producer(drop))).start();
        (new Thread(new Consumer(drop))).start();
    }
}
```

参考资料：

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>

