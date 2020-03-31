---
title: 笔记《Oracle官方并发教程》4.活跃度
date: 2019-07-20 18:37:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

活跃性就是指一个并发应用程序能及时执行的能力。本小节介绍最常见的活跃度问题：死锁(deadlock)。还有饥饿(starvation)和活锁(livelock)。

### 死锁

死锁就是说，两个或多个线程相互阻塞，都在等待对方释放资源。

例如，两个朋友鞠躬，A向B鞠躬时，必须保持姿势，直到B向A回鞠躬。这个时候，如果两个人同时鞠躬，那么就死锁了。

例子如下，<u>本质上是，A同步方法持有对象内部锁，A调用B同步方法，而B同步方法跟A方法处于同一个对象，是需要请求该对象的同步锁的，同步锁又在A拿着，就死锁了</u>：

``` java
   static class Friend {
        private final String name;
        public Friend(String name) {
            this.name = name;
        }
        public String getName() {
            return this.name;
        }
        public synchronized void bow(Friend bower) {
            System.out.format("%s: %s"
                + "  has bowed to me!%n",
                this.name, bower.getName());
            bower.bowBack(this);
        }
        public synchronized void bowBack(Friend bower) {
            System.out.format("%s: %s"
                + " has bowed back to me!%n",
                this.name, bower.getName());
        }
    }

    public static void main(String[] args) {
        final Friend alphonse =
            new Friend("Alphonse");
        final Friend gaston =
            new Friend("Gaston");
        new Thread(new Runnable() {
            public void run() { alphonse.bow(gaston); }
        }).start();
        new Thread(new Runnable() {
            public void run() { gaston.bow(alphonse); }
        }).start();
    }
}
```

### 饥饿和活锁

这两种情况没有死锁普遍，不过也可能遇到。

#### 饥饿

就是说，一个线程不能正常的访问共享资源，或者不能正常执行。比如说，共享资源被其他线程长期占用，自己很少有访问的机会。

#### 活锁

一个线程通常会有会响应其他线程的活动。如果其他线程也会响应另一个线程的活动，那么就有可能发生活锁。同死锁一样，发生活锁的线程无法继续执行。然而线程并没有阻塞——他们在忙于响应对方无法恢复工作。这就相当于两个在走廊相遇的人：Alphonse向他自己的左边靠想让Gaston过去，而Gaston向他的右边靠想让Alphonse过去。可见他们阻塞了对方。Alphonse向他的右边靠，而Gaston向他的左边靠，他们还是阻塞了对方。

参考资料：

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>