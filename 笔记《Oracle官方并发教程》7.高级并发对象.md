---
title: 笔记《Oracle官方并发教程》7.高级并发对象
date: 2019-07-21 16:19:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

之前的文章中主要介绍了Java的低级API如何处理并发编程。本节将介绍一些处理并发编程的高级Java API。这些API是Java5.0新增的，大部分在JUC里面实现，Java集合框架也定义了新的并发数据结构。下面是本节要介绍的内容：

* 锁对象：多并发锁的另一种实现方式。
* Executors：用于加载和管理线程，提供了线程池功能。
* 并发集合：简化了多并发集合的操作，并提升了性能。
* 原子变量：减小同步粒度，避免内存一致性错误。
* 并发随机数(JDK7)：高效的多线程生成伪随机数。

### 锁对象

锁对象指的是`java.util.concurrent.locks`包提供的锁。和同步代码使用的synchronized隐式锁一样，每次只有一个线程可以获得锁对象。锁对象可以通过关联`Condition`对象，来支持wait/notify机制。锁对象相比于传统的sychronized来说，锁对象的优势是，<u>锁对象不可用或者超时的时候，`tryLock()`方法可以收回获取锁的请求</u>。<u>如果在锁获取前，另一个线程发送了一个中断，`lockInterruptibly()`方法也会收回获取锁的请求</u>。

由于**锁对象可以收回获取锁的请求**，因此我们可以用来灵活的解决很多问题。比如，解决之前我们遇到的死锁问题。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.Random;

public class Safelock {
    static class Friend {
        private final String name;
        private final Lock lock = new ReentrantLock();

        public Friend(String name) {
            this.name = name;
        }

        public String getName() {
            return this.name;
        }

        public boolean impendingBow(Friend bower) {
            Boolean myLock = false;
            Boolean yourLock = false;
            try {
                myLock = lock.tryLock();
                yourLock = bower.lock.tryLock();
            } finally {
                if (! (myLock && yourLock)) {
                    if (myLock) {
                        lock.unlock();
                    }
                    if (yourLock) {
                        bower.lock.unlock();
                    }
                }
            }
            return myLock && yourLock;
        }

        public void bow(Friend bower) {
            if (impendingBow(bower)) {	// 自己和对方的锁都拿到后，才动作。
                try {
                    System.out.format("%s: %s has"
                        + " bowed to me!%n",
                        this.name, bower.getName());
                    bower.bowBack(this);
                } finally {
                    lock.unlock();
                    bower.lock.unlock();
                }
            } else {
                System.out.format("%s: %s started"
                    + " to bow to me, but saw that"
                    + " I was already bowing to"
                    + " him.%n",
                    this.name, bower.getName());
            }
        }

        public void bowBack(Friend bower) {
            System.out.format("%s: %s has" +
                " bowed back to me!%n",
                this.name, bower.getName());
        }
    }
    // ----------------
    static class BowLoop implements Runnable {
        private Friend bower;
        private Friend bowee;

        public BowLoop(Friend bower, Friend bowee) {
            this.bower = bower;
            this.bowee = bowee;
        }

        public void run() {
            Random random = new Random();
            for (;;) {
                try {
                    Thread.sleep(random.nextInt(10));
                } catch (InterruptedException e) {}
                bowee.bow(bower);
            }
        }
    }

    public static void main(String[] args) {
        final Friend alphonse =
            new Friend("Alphonse");
        final Friend gaston =
            new Friend("Gaston");
        new Thread(new BowLoop(alphonse, gaston)).start();
        new Thread(new BowLoop(gaston, alphonse)).start();
    }
}
```

### 执行器（Executors）

对于大规模并发应用，最优实践是将线程的创建管理和程序的其他部分解耦。封装这些功能的对象就是执行器。以下从三个方面来阐述执行器部分：

* 执行器接口：定义了三种类型的执行器对象
* 线程池：最常用的执行器实现
* Fork/Join：JDK7加入的并发框架，把一个大任务分解为多个小任务，最终再汇总每个小任务的结果。

#### Executor接口

分为三部分：

* Executor：一个运行新任务的简单接口。就是执行线程的，把线程扔进去就不用管了。
* ExecutorService：扩展了Executor接口，添加了一些用来管理执行器生命周期和任务生命周期的方法。
* ScheduledExecutorService：扩展了ExecutorService，支持Future和定期执行任务。

上面三个接口是继承的关系。爷爷、爸爸、儿子的关系。

##### Executor接口

这个接口就只有一个`execute()`方法，用来替换通常创建启动线程的方法。例如，r是一个Runnable对象，e是一个Executor对象。那么可以使用

```java
e.execute(r);
```

来代替

``` java
new Thread(r).start();
```

不过`execute()`方法没有定义具体的实现方式。不同的executor实现，可能是直接创建一个新线程启动，更可能是用已有的工作线程r，或者把r放到队列中等待可用的工作线程。(后面线程池会讲)。

``` java
// 全部Executor源码
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

##### ExecutorService接口

ExecutorService接口除了提供`execute()`方法，还提供了更通用的`submit()`方法。

`submit()`方法除了和`execute()`方法一样可以接收`Runnable`对象外，还可以接收`Callable`对象作为入参。

`submit()`接收`Callable`作为入参时，可以使任务返回执行的结果。通过返回的Future对象可以读取任务的执行结果，或者管理`Callable`任务和`Runnable`任务的状态。下面一看`submit()`方法的源码就很清楚了：

```java
// ExecutorService源码
public interface ExecutorService extends Executor {
    // ..........
    /**
     * Submits a value-returning task for execution and returns a
     * Future representing the pending results of the task. The
     * Future's {@code get} method will return the task's result upon
     * successful completion.
     *
     * <p>
     * If you would like to immediately block waiting
     * for a task, you can use constructions of the form
     * {@code result = exec.submit(aCallable).get();}
     *
     * <p>Note: The {@link Executors} class includes a set of methods
     * that can convert some other common closure-like objects,
     * for example, {@link java.security.PrivilegedAction} to
     * {@link Callable} form so they can be submitted.
     *
     * @param task the task to submit
     * @param <T> the type of the task's result
     * @return a Future representing pending completion of the task
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if the task is null
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     * Submits a Runnable task for execution and returns a Future
     * representing that task. The Future's {@code get} method will
     * return the given result upon successful completion.
     *
     * @param task the task to submit
     * @param result the result to return
     * @param <T> the type of the result
     * @return a Future representing pending completion of the task
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if the task is null
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * Submits a Runnable task for execution and returns a Future
     * representing that task. The Future's {@code get} method will
     * return {@code null} upon <em>successful</em> completion.
     *
     * @param task the task to submit
     * @return a Future representing pending completion of the task
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if the task is null
     */
    Future<?> submit(Runnable task);
    // ..........
}
```

##### ScheduledExecutorService接口

ScheduledExecutorService接口继承了ExecutorService接口，并且增加了`schedule()`方法。调用`schedule()`方法可以在指定的延时后执行一个`Runnable`或者`Callable`任务。

ScheduledExecutorService接口还定义了按照指定时间间隔定期执行任务的`scheduleAtFixedRate()`方法和`scheduleWithFixedDelay()`方法。

```java
// 全部ScheduledExecutorService接口源码
public interface ScheduledExecutorService extends ExecutorService {

    /**
     * Creates and executes a one-shot action that becomes enabled
     * after the given delay.
     *
     * @param command the task to execute
     * @param delay the time from now to delay execution
     * @param unit the time unit of the delay parameter
     * @return a ScheduledFuture representing pending completion of
     *         the task and whose {@code get()} method will return
     *         {@code null} upon completion
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if command is null
     */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

    /**
     * Creates and executes a ScheduledFuture that becomes enabled after the
     * given delay.
     *
     * @param callable the function to execute
     * @param delay the time from now to delay execution
     * @param unit the time unit of the delay parameter
     * @param <V> the type of the callable's result
     * @return a ScheduledFuture that can be used to extract result or cancel
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if callable is null
     */
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);

    /**
     * Creates and executes a periodic action that becomes enabled first
     * after the given initial delay, and subsequently with the given
     * period; that is executions will commence after
     * {@code initialDelay} then {@code initialDelay+period}, then
     * {@code initialDelay + 2 * period}, and so on.
     * If any execution of the task
     * encounters an exception, subsequent executions are suppressed.
     * Otherwise, the task will only terminate via cancellation or
     * termination of the executor.  If any execution of this task
     * takes longer than its period, then subsequent executions
     * may start late, but will not concurrently execute.
     *
     * @param command the task to execute
     * @param initialDelay the time to delay first execution
     * @param period the period between successive executions
     * @param unit the time unit of the initialDelay and period parameters
     * @return a ScheduledFuture representing pending completion of
     *         the task, and whose {@code get()} method will throw an
     *         exception upon cancellation
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if command is null
     * @throws IllegalArgumentException if period less than or equal to zero
     */
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

    /**
     * Creates and executes a periodic action that becomes enabled first
     * after the given initial delay, and subsequently with the
     * given delay between the termination of one execution and the
     * commencement of the next.  If any execution of the task
     * encounters an exception, subsequent executions are suppressed.
     * Otherwise, the task will only terminate via cancellation or
     * termination of the executor.
     *
     * @param command the task to execute
     * @param initialDelay the time to delay first execution
     * @param delay the delay between the termination of one
     * execution and the commencement of the next
     * @param unit the time unit of the initialDelay and delay parameters
     * @return a ScheduledFuture representing pending completion of
     *         the task, and whose {@code get()} method will throw an
     *         exception upon cancellation
     * @throws RejectedExecutionException if the task cannot be
     *         scheduled for execution
     * @throws NullPointerException if command is null
     * @throws IllegalArgumentException if delay less than or equal to zero
     */
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
}
```

#### 线程池

在JUC包中多数的执行器实现都用到了由工作线程组成的线程池。工作线程独立于它要执行的Runnable任务和Callbale任务。因为在大规模并发应用中该，创建大量的Thread对象很浪费系统资源(占用内存、分配和回收麻烦)。

最常见的线程池就是固定大小的线程池。当一个线程终止了，系统会自动新增一个线程。需要执行的任务则是交给一个内部队列，队列再交给线程执行。使用固定线程的好处就是，可以优雅退化，并发量上来以后系统不会一下就崩掉。

创建一个线程池最简单的方法就是调用`java.util.concurrent.Executors`的`newFixedThreadPool`方法。此外，Executors还提供了一下方法：

* `newCachedThreadPool()`：可扩展的线程池，适用于启动多个短任务。
* `newSingleThreadExecutor()`：每次只执行一个任务的执行器。

* 除此之外还有一下创建`ScheduledExecutorService`执行器的方法。具体可以看`Executors`源码。

如果上面的方法都不满足需要，可以尝试`java.util.concurrent.ThreadPoolExecutor`或者`java.util.concurrent.ScheduledThreadPoolExecutor`。

因为其实`Executors`里的好多构建Executor的方法，都是调用`ThreadPoolExecutor`、`ScheduledThreadPoolExecutor`等等Executor实现类的构造方法来实现的，入参不一样罢了。比如说，最常用的newFixedThreadPool()方法：

```java
// Executors源码
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

#### Fork/Join

fork/join框架是`ExecutorService`接口的一种具体实现，目的是为了帮助你更好地利用多处理器带来的好处，它是为那些能够被递归地拆解成子任务的工作类型量身设计的。主要思路是把一个大任务分解为多个小任务，最终再汇总每个小任务的结果组成大任务的结果(分治法？)。

类似于`ExecutorService`接口的其他实现，fork/join框架会将任务分发给线程池中的工作线程。

<u>**fork/join框架的独特之处在与它使用工作窃取(work-stealing)算法**</u>。完成自己的工作而处于空闲的工作线程能够从其他仍然处于忙碌(busy)状态的工作线程处窃取等待执行的任务。

fork/join框架的核心是`ForkJoinPool`类，它是对`AbstractExecutorService`类的扩展。`ForkJoinPool`实现了工作偷取算法，并可以执行`ForkJoinTask`任务。

##### 基本使用方法

使用fork/join框架的第一步是编写执行一部分工作的代码。你的代码结构看起来应该与下面所示的伪代码类似：

```java
if (当前这个任务工作量足够小)
    直接完成这个任务
else
    将这个任务或这部分工作分解成两个部分
    分别触发(invoke)这两个子任务的执行，并等待结果
```

你需要将这段代码包裹在一个`ForkJoinTask`的子类中。不过，通常情况下会使用一种更为具体的的类型，或者是[`RecursiveTask`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/RecursiveTask.html)(会返回一个结果)，或者是[`RecursiveAction`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/RecursiveAction.html)。当你的`ForkJoinTask`子类准备好了，创建一个代表所有需要完成工作的对象，然后将其作为参数传递给一个`ForkJoinPool`实例的`invoke()`方法即可。

实际上`ForkJoinPool`实例的`invoke()`方法底层也是经过一系列处理后，调用submit()方法：

``` java
// ForkJoinPool源码
public class ForkJoinPool extends AbstractExecutorService {
// ..........
    
    public <T> T invoke(ForkJoinTask<T> task) {
        if (task == null)
            throw new NullPointerException();
        externalPush(task);
        return task.join();
    }
    
        /**
     * Tries to add the given task to a submission queue at
     * submitter's current queue. Only the (vastly) most common path
     * is directly handled in this method, while screening for need
     * for externalSubmit.
     *
     * @param task the task. Caller must ensure non-null.
     */
    final void externalPush(ForkJoinTask<?> task) {
        WorkQueue[] ws; WorkQueue q; int m;
        int r = ThreadLocalRandom.getProbe();
        int rs = runState;
        if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
            (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
            U.compareAndSwapInt(q, QLOCK, 0, 1)) {
            ForkJoinTask<?>[] a; int am, n, s;
            if ((a = q.array) != null &&
                (am = a.length - 1) > (n = (s = q.top) - q.base)) {
                int j = ((am & s) << ASHIFT) + ABASE;
                U.putOrderedObject(a, j, task);
                U.putOrderedInt(q, QTOP, s + 1);
                U.putIntVolatile(q, QLOCK, 0);
                if (n <= 1)
                    signalWork(ws, q);
                return;
            }
            U.compareAndSwapInt(q, QLOCK, 1, 0);
        }
        externalSubmit(task);
    }
    
// ..........
}
```

##### 使用示例：图像

想要了解fork/join框架的基本工作原理，接下来的这个例子会有所帮助。假设你想要模糊一张图片。原始的*source*图片由一个整数的数组表示，每个整数表示一个像素点的颜色数值。与*source*图片相同，模糊之后的*destination*图片也由一个整数数组表示。 对图片的模糊操作是通过对*source*数组中的每一个像素点进行处理完成的。处理的过程是这样的：将每个像素点的色值取出，与周围像素的色值（红、黄、蓝三个组成部分）放在一起取平均值，得到的结果被放入*destination*数组。因为一张图片会由一个很大的数组来表示，这个流程会花费一段较长的时间。如果使用fork/join框架来实现这个模糊算法，你就能够借助多处理器系统的并行处理能力。下面是上述算法结合fork/join框架的一种简单实现：

``` java
public class ForkBlur extends RecursiveAction {
private int[] mSource;
private int mStart;
private int mLength;
private int[] mDestination;

// Processing window size; should be odd.
private int mBlurWidth = 15;

public ForkBlur(int[] src, int start, int length, int[] dst) {
    mSource = src;
    mStart = start;
    mLength = length;
    mDestination = dst;
}

protected void computeDirectly() {
    int sidePixels = (mBlurWidth - 1) / 2;
    for (int index = mStart; index < mStart + mLength; index++) {
        // Calculate average.
        float rt = 0, gt = 0, bt = 0;
        for (int mi = -sidePixels; mi  <= sidePixels; mi++) {
            int mindex = Math.min(Math.max(mi + index, 0),
                                mSource.length - 1);
            int pixel = mSource[mindex];
            rt += (float)((pixel &amp; 0x00ff0000) >> 16) / mBlurWidth;
            gt += (float)((pixel &amp; 0x0000ff00) >>  8) / mBlurWidth;
            bt += (float)((pixel &amp; 0x000000ff) >>  0) / mBlurWidth;
        }

        // Reassemble destination pixel.
        int dpixel = (0xff000000     ) |
               (((int)rt) << 16) |
               (((int)gt) <<  8) |
               (((int)bt) <<  0);
        mDestination[index] = dpixel;
    }
}
```

接下来你需要实现父类中的`compute()`方法，它会直接执行模糊处理，或者将当前的工作拆分成两个更小的任务。数组的长度可以作为一个简单的阀值来判断任务是应该直接完成还是应该被拆分。

```java
protected static int sThreshold = 100000;

protected void compute() {	// RecursiveAction抽象方法实现。这里就是真正的线程跑起来会执行的逻辑
    if (mLength &lt; sThreshold) {
        computeDirectly();
        return;
    }

    int split = mLength / 2;

    invokeAll(new ForkBlur(mSource, mStart, split, mDestination),
              new ForkBlur(mSource, mStart + split, mLength - split,
                           mDestination));
}
```

这样任务对象就算建好了。因为这里 ForkBlur继承RecursiveAction， 然后RecursiveAction又继承ForkJoinTask\<Void>，所以直接把ForkBlur它丢到ForkJoinPool里面就行了。

``` java
// source image pixels are in src
// destination image pixels are in dst
ForkJoinTask<Void> fb = new ForkBlur(src, 0, src.length, dst);

ForkJoinPool pool = new ForkJoinPool();

pool.invoke(fb);
```

完整的源代码实现：

```java
/*
* Copyright (c) 2010, 2013, Oracle and/or its affiliates. All rights reserved.
*
* Redistribution and use in source and binary forms, with or without
* modification, are permitted provided that the following conditions
* are met:
*
*   - Redistributions of source code must retain the above copyright
*     notice, this list of conditions and the following disclaimer.
*
*   - Redistributions in binary form must reproduce the above copyright
*     notice, this list of conditions and the following disclaimer in the
*     documentation and/or other materials provided with the distribution.
*
*   - Neither the name of Oracle or the names of its
*     contributors may be used to endorse or promote products derived
*     from this software without specific prior written permission.
*
* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS
* IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO,
* THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
* PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
* CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
* EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
* PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
* PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
* LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
* NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
* SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
*/

import java.awt.image.BufferedImage;
import java.io.File;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;
import javax.imageio.ImageIO;

/**
 * ForkBlur implements a simple horizontal image blur. It averages pixels in the
 * source array and writes them to a destination array. The sThreshold value
 * determines whether the blurring will be performed directly or split into two
 * tasks.
 *
 * This is not the recommended way to blur images; it is only intended to
 * illustrate the use of the Fork/Join framework.
 */
public class ForkBlur extends RecursiveAction {

    private int[] mSource;
    private int mStart;
    private int mLength;
    private int[] mDestination;
    private int mBlurWidth = 15; // Processing window size, should be odd.

    public ForkBlur(int[] src, int start, int length, int[] dst) {
        mSource = src;
        mStart = start;
        mLength = length;
        mDestination = dst;
    }

    // Average pixels from source, write results into destination.
    protected void computeDirectly() {
        int sidePixels = (mBlurWidth - 1) / 2;
        for (int index = mStart; index < mStart + mLength; index++) {
            // Calculate average.
            float rt = 0, gt = 0, bt = 0;
            for (int mi = -sidePixels; mi <= sidePixels; mi++) {
                int mindex = Math.min(Math.max(mi + index, 0), mSource.length - 1);
                int pixel = mSource[mindex];
                rt += (float) ((pixel & 0x00ff0000) >> 16) / mBlurWidth;
                gt += (float) ((pixel & 0x0000ff00) >> 8) / mBlurWidth;
                bt += (float) ((pixel & 0x000000ff) >> 0) / mBlurWidth;
            }

            // Re-assemble destination pixel.
            int dpixel = (0xff000000)
                    | (((int) rt) << 16)
                    | (((int) gt) << 8)
                    | (((int) bt) << 0);
            mDestination[index] = dpixel;
        }
    }
    protected static int sThreshold = 10000;

    @Override
    protected void compute() {
        if (mLength < sThreshold) {
            computeDirectly();
            return;
        }

        int split = mLength / 2;

        invokeAll(new ForkBlur(mSource, mStart, split, mDestination),
                new ForkBlur(mSource, mStart + split, mLength - split, 
                mDestination));
    }

    // Plumbing follows.
    public static void main(String[] args) throws Exception {
        String srcName = "red-tulips.jpg";
        File srcFile = new File(srcName);
        BufferedImage image = ImageIO.read(srcFile);
        
        System.out.println("Source image: " + srcName);
        
        BufferedImage blurredImage = blur(image);
        
        String dstName = "blurred-tulips.jpg";
        File dstFile = new File(dstName);
        ImageIO.write(blurredImage, "jpg", dstFile);
        
        System.out.println("Output image: " + dstName);
        
    }

    public static BufferedImage blur(BufferedImage srcImage) {
        int w = srcImage.getWidth();
        int h = srcImage.getHeight();

        int[] src = srcImage.getRGB(0, 0, w, h, null, 0, w);
        int[] dst = new int[src.length];

        System.out.println("Array size is " + src.length);
        System.out.println("Threshold is " + sThreshold);

        int processors = Runtime.getRuntime().availableProcessors();
        System.out.println(Integer.toString(processors) + " processor"
                + (processors != 1 ? "s are " : " is ")
                + "available");

        ForkBlur fb = new ForkBlur(src, 0, src.length, dst);

        ForkJoinPool pool = new ForkJoinPool();

        long startTime = System.currentTimeMillis();
        pool.invoke(fb);
        long endTime = System.currentTimeMillis();

        System.out.println("Image blur took " + (endTime - startTime) + 
                " milliseconds.");

        BufferedImage dstImage =
                new BufferedImage(w, h, BufferedImage.TYPE_INT_ARGB);
        dstImage.setRGB(0, 0, w, h, dst, 0, w);

        return dstImage;
    }
}
```

##### 标准实现

除了能够使用fork/join框架来实现能够在多处理系统中被并行执行的定制化算法（如前文中的ForkBlur.java例子），在Java SE中一些比较常用的功能点也已经使用fork/join框架来实现了。

在Java SE 8中，`java.util.Arrays`类的一系列`parallelSort()`方法就使用了fork/join来实现。这些方法与`sort()`系列方法很类似，但是通过使用fork/join框架，借助了并发来完成相关工作。

在多处理器系统中，对大数组的并行排序会比串行排序更快。这些方法究竟是如何运用fork/join框架并不在本教程的讨论范围内。

 其他采用了fork/join框架的方法还包括`java.util.streams`包中的一些方法，此包是作为Java SE 8发行版中[`Project Lambda`](http://openjdk.java.net/projects/lambda/)的一部分。想要了解更多信息，请参见[`Lambda Expressions`](http://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)一节

### 并发集合

java.util.concurrent包囊括了Java集合框架的一些附加类。它们也最容易按照集合类所提供的接口来进行分类：

* `BlockingQueue`：先进先出的队列，队列满进行入队，或队列空进行出队，会阻塞或超时。
* `ConcurrentMap`：java.util.Map的子接口，定义了一套原子操作。key不存在时可新增，key存在时可修改删除。所有这些操作都是原子化的，从而避免同步的低效率或不灵活。`ConcurrentHashMap`是它的标准实现，是`HashMap`的并发模式。
* `ConcurrentNavigableMap`：ConcurrentMap的子接口，支持近似匹配。`ConcurrentSkipListMap`是它的标准实现，是`TreeMap`的并发模式。

上面的并发集合，都是通过在新增对象、访问/移除对象的操作间，定义一个happens-before的关系，来帮助程序员避免内存一致性错误。

### 原子变量

`java.util.concurrent.atomic`包定义了对单一变量进行原子操作的类。同一变量上的一个set操作对于任意后续的get操作存在happens-before关系。compareAndSet方法也是保证内存一致性的。我们来看下非原子变量和原子变量用法的区别：

非原子变量，保证线程安全的用法：

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

class SynchronizedCounter {
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

原子变量，天然提供了保证了线程安全的方法：

```java
import java.util.concurrent.atomic.AtomicInteger;
class AtomicCounter {
private AtomicInteger c = new AtomicInteger(0);
public void increment() {
c.incrementAndGet();
}

public void decrement() {
c.decrementAndGet();
}

public int value() {
return c.get();
}
```

### 并发随机数

在JDK7中，`java.util.concurrent`提供了`ThreadLocalRandom`，可以用它在多个线程或ForkJoinTasks中生成随机数。对于并发访问，使用TheadLocalRandom代替Math.random()可以减少竞争，从而获得更好的性能。

用法：调用ThreadLocalRandom.current()， 然后调用它的其中一个方法去获取一个随机数即可：

``` java
int r = ThreadLocalRandom.current().nextInt(4,77);
```



参考资料

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>