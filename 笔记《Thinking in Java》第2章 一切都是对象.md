---
title: 笔记《Thinking in Java》第2章 一切都是对象
date: 2019-08-01 02:28:26
categories:
- JavaSE
- Java基础
tags:
- 学习笔记
- Java基础
top:
---

### 第二章 一切都是对象

#### 1.数据可以存在哪

* 寄存器。因为它在CPU内部，所以最快。但是Java无法直接控制它。
* 栈。在RAM上，但是，CPU可以通过栈指针快速的分配存储，向下就分配新内存，向上就释放内存，所以速度很快。代价是，Java系统必须确切的知道数据在栈里的生命周期，所以灵活性有限。Java的对象引用存在这。
* 堆。也是在RAM上，不过跟栈比，编译器不用知道数据在堆里的生命周期，所以在堆里分配存储很灵活，代价是更耗时。Java所有的对象存在这。
* 常量存储。常量值直接放在程序代码内部，这样也是安全的。
* 非RAM存储。常见的两种：流对象和持久化对象。前者是字节流，后者是存在磁盘上。

#### 2.基本类型

基本类型存在栈里面，这样这个变量就直接存储“值”了，比起对象来更轻量，更快速。

| 基本类型 | 大小  | 范围               | 最大值            | 包装类        |
| -------- | ----- | ------------------ | ----------------- | ------------- |
| boolean  |       |                    |                   | Boolean       |
| byte     | 1字节 | -128               | 127               | Byte          |
| char     | 2字节 | Unicode0           | Unicode65535      | **Character** |
| short    | 2字节 | -32768             | 32767             | Short         |
| int      | 4字节 | -2,147,483,648     | 2,147,483,647     | Integer       |
| long     | 8字节 | 约-900亿亿\9*10^18 | 约900亿亿\9*10^18 | Long          |
| float    | 4字节 |                    |                   | Float         |
| double   | 8字节 |                    |                   | Double        |
| void     |       |                    |                   | **Void**      |

浮点数与整型数转换时，注意精度丢失问题。float精度没有int高(?)。

关于浮点数精度及表示法，后续补充。

BigInteger和BigDecimal可以表示任意精度及大小的整数和定点数。

Java创建数组时，其实就是创建了一个引用数组。数组存不了基本型，而是包装类。后续详解。

#### 3.对象作用域

java的作用域是大花括号，并且有闭包。以下程序，C++里可以，但Java不合法：

```java
{
    int x = 12;
    {
        int x = 96;	// Illegal
    }
    
}
```

#### 4.类

类属性：基本类型会被初始化。引用类型初始化为null。

方法属性：不会被初始化。如果要使用，编译器会强制要求你初始化。

```java
	void method(int i, String str){
		int method_a;
		String method_str;
		
		System.out.println(i);
		System.out.println(str);
		
		System.out.println(method_a);	// Illegal
		System.out.println(method_str);	// Illegal
		
		System.out.println(clazz_i);
		System.out.println(clazz_a);
	}
```

可以通过类名直接调用类方法和字段(最佳实践)，也可以通过对象调用类方法(少用)。

```java
	static String clazz_a = "clazz_a";
	static void classMethod() {
		System.out.println("Class method invoke!");
	}

	new Test201957().classMethod();	// Class method invoke!
	System.out.println(new Test201957(1).clazz_a);	// clazz_a
```

#### 5.其他

java.lang包是默认导入到程序的，无需显示导入。

#### 6.注释和嵌入文档(了解即可)

private的javaDoc会被忽略

javaDoc也可以嵌入html来使用

@deprecated用来加删除线