---
title: 笔记《Oracle官方并发教程》1.进程和线程
date: 2019-07-19 12:47:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

在并发编程中，有两个基本的执行单元：进程和线程。进程是资源分配的最小单位，线程时CPU调度的最小单位。

多处理器或者多核多处理器(一个处理器有多个核)的计算机现在很普遍，可以用多线程来提高性能。

不过在单核的计算机中，也有许多活动的进程和线程。在单核计算机中，任意时刻只有一个线程在运行，CPU通过时间片来分配。

### 进程

进程具有一个独立的执行环境。通常情况下，进程拥有一个完整的、私有的基本运行资源集合。特别地，每个进程都有自己的内存空间。

一个单独的应用可以一个进程，也可以是多个协作的进程集合。为了便于进程间通信，一般操作系统都支持IPC(进程间通信)，比如pipes和sockets。IPC不仅可以支持同一个系统上的通信，也可以支持不同系统间的进程通信。

大多数的Java虚拟机是单进程运行的(Most implementations of the Java virtual machine run as a single process. )。不过Java应用程序可以用[`ProcessBuilder`](https://docs.oracle.com/javase/8/docs/api/java/lang/ProcessBuilder.html) 对象来建立额外的进程。这里我们不讨论多进程的应用程序。

### 线程

线程有时候又叫做*轻量级进程(*lightweight processes*)*。进程和线程都会提供一套执行环境，不过创建线程需要的资源比创建进程要少。

线程是挂在进程下的，每个进程至少有一个线程。同一个进程下的不同线程，共享进程的资源，包括内存和打开的文件。这样效率很高，不过会有潜在的问题：线程间通信。

多线程执行是Java平台的基本特征。每个应用至少会有一个线程 - 或几个，如果算上“系统”线程的话，比如内存管理和信号处理等。不过从程序员的视角来看，你只是启动了一个线程：`main`线程。这个线程有能力创建其他额外的线程，我们之后会讲。



参考资料

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>

