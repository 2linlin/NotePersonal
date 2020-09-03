---
title: 源码-Java容器-1.总体认识
date: 2019-08-15 00:37:25
categories:
- JavaSE
- Java容器
tags:
- Java容器
top:
---

Java容器部分所有的内容都是基于Java1.8分析的。

#### 1.总体设计思路

Java容器框架，由Collection和Map组成。其中，Collection又分化为Set、List和Queue。Map整体继承结构则较为简单。此外还有Java1.2版本之前的HashTable，后续会简单提一下。我们先简单的介绍下框架的整体情况，然后后续会做具体的分析。

首先我们简单说下Collection部分的总体设计思路：

* 第一层实现：将Collection需要具备的特征抽出来汇总到抽象类AbstractCollection。将Set、List、Queue需要具备的功能作为接口抽象出来分别汇总到Set、List、Queue接口。
* 第二步实现：在上一步的基础上，继承抽象类AbstractCollection，并实现Set、List、Queue接口，从而汇总出更具体一些的AbstractSet、AbstractList、AbstractQueue。
* 第三步实现：根据具体的需要，分别继承AbstractSet、AbstractList、AbstractQueue，同样的也实现Set、List、Queue接口，从而生成HashSet、LiinkedHashSet、TreeSet、ArrayList、LinkedList、Vector、Stack、PriorityQueue、ArrayDeque这9个具体的容器类。

**一句话总结就是，抽出抽象类和接口，然后继承抽象类和接口形成新的抽象类，再继承新的抽象类和接口形成实现类。**

然后简单说下Map部分。其实Set就是用没有value的Map来实现的，所以Map的结构其实和Set是相近的。Map的总体设计思路：

* 第一步实现：将Map需要具备的功能抽出来汇总到抽象类AbstractMap。
* 第二步实现：根据具体的需要，继承AbstractMap，并实现Map接口，从而生成HashMap、LinkedHashMap、WeakHashMap、IdentityHashMap、EnumHashMap、TreeMap这6个容器类。
* 比较特别的HashTable则是直接实现Map接口，并继承Dictionary抽象类来实现。

顺口提一句，Java1.8为了加入stream等新功能，加入了接口专用的default关键字。后面讲到Collection方法的时候会简单分析接口中static和default关键字的使用。

再顺口提一句，除了以上提到的实现类，在JUC包里的容器类也是对List<E>接口、AbstractSet等抽象类有实现的，这个忽略，以后讲到JUC包的时候再分析。

#### 2.源码继承结构分析

下面是Java集合体系(不包括JUC)里面涉及到的所有接口、抽象类、实现类的继承关系：

Collection继承体系：

```java
public interface Collection<E> extends Iterable<E> {}
public abstract class AbstractCollection<E> implements Collection<E> {}
		
public interface Set<E> extends Collection<E> {}
	public abstract class AbstractSet<E> extends AbstractCollection<E> implements Set<E> {}
		public interface SortedSet<E> extends Set<E> {}
			public interface NavigableSet<E> extends SortedSet<E> {}
				public class TreeSet<E> extends AbstractSet<E> implements NavigableSet<E>, Cloneable, java.io.Serializable
				public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable
					public class LinkedHashSet<E> extends HashSet<E> implements Set<E>, Cloneable, java.io.Serializable {}
					
public interface List<E> extends Collection<E> {}
	public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>{}
		public abstract class AbstractSequentialList<E> extends AbstractList<E> {
		public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable
		public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
	public class Vector<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
		public class Stack<E> extends Vector<E> {}
		
public interface Queue<E> extends Collection<E> {}
	public abstract class AbstractQueue<E> extends AbstractCollection<E> implements Queue<E> {}
		public class PriorityQueue<E> extends AbstractQueue<E> implements java.io.Serializable {}
	public interface Deque<E> extends Queue<E> {}
		public class ArrayDeque<E> extends AbstractCollection<E> implements Deque<E>, Cloneable, Serializable {}
```

Map继承体系：

```java
public interface Map<K,V> {}
	public abstract class AbstractMap<K,V> implements Map<K,V> {}
		public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {}
			public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {}
			
			public class WeakHashMap<K,V> extends AbstractMap<K,V> implements Map<K,V> {}
			public class IdentityHashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, java.io.Serializable, Cloneable {}
			public class EnumMap<K extends Enum<K>, V> extends AbstractMap<K, V> implements java.io.Serializable, Cloneable {}
			
			
	public interface SortedMap<K,V> extends Map<K,V> {}
		public interface NavigableMap<K,V> extends SortedMap<K,V> {}
		public class TreeMap<K,V> extends AbstractMap<K,V> implements NavigableMap<K,V>, Cloneable, java.io.Serializable {}
	
	public class Hashtable<K,V> extends Dictionary<K,V> implements Map<K,V>, Cloneable, java.io.Serializable {}
```

#### 3.整体继承关系图

以下关系图是由上述2中源代码分析而来的，已包含集合所有实现类(除JUC外)。

Map继承关系图：

![](https://md-images-1259736917.cos.ap-guangzhou.myqcloud.com/md-images/20190814181937.png)

Collection继承关系图：

![](https://md-images-1259736917.cos.ap-guangzhou.myqcloud.com/md-images/20190815003032.png)

以上就是整体分析。之后的文章我会按这个思路，分析Collection，再分析AbstractCollection和Set、List、Queue，再分析AbstractSet、AbstractList、AbstractQueue，最后分析具体实现类。Map的分析也是相近的思路。我们会先分析Map，再分析Collection。因为Set是基于Map实现的。