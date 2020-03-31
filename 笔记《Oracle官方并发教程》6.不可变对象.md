---
title: 笔记《Oracle官方并发教程》6.不可变对象
date: 2019-07-21 15:42:48
categories:
- JavaSE
- Java并发
tags:
- 学习笔记
- Java并发
top:
---



[TOC]

不可变对象就是创建后不能修改的对象。由于不可变对象不存在线程干扰和内存一致性问题，因此用不可变对象进行并发编程很有用。

不可变对象虽然有创建新对象的开销，但是因为减低了垃圾回收的开销，并且减少了用来保证可变对象出现并发错误的代码，因此并不需要担心它的开销问题。

下面举例，一个可变对象的类，然后转化为不可变对象的类。

### 一个同步类的例子

`SynchronizedRGB`是表示颜色的类，每一个对象代表一种颜色，使用三个整形数表示颜色的三基色，字符串表示颜色名称。

``` java
public class SynchronizedRGB {

    // Values must be between 0 and 255.
    private int red;
    private int green;
    private int blue;
    private String name;

    private void check(int red,
                       int green,
                       int blue) {
        if (red < 0 || red > 255
            || green < 0 || green > 255
            || blue < 0 || blue > 255) {
            throw new IllegalArgumentException();
        }
    }

    public SynchronizedRGB(int red,
                           int green,
                           int blue,
                           String name) {
        check(red, green, blue);
        this.red = red;
        this.green = green;
        this.blue = blue;
        this.name = name;
    }

    public void set(int red,
                    int green,
                    int blue,
                    String name) {
        check(red, green, blue);
        synchronized (this) {
            this.red = red;
            this.green = green;
            this.blue = blue;
            this.name = name;
        }
    }

    public synchronized int getRGB() {
        return ((red << 16) | (green << 8) | blue);
    }

    public synchronized String getName() {
        return name;
    }

    public synchronized void invert() {
        red = 255 - red;
        green = 255 - green;
        blue = 255 - blue;
        name = "Inverse of " + name;
    }
}
```

上面的类，多线程的时候要小心，使用不当可能导致内存不一致问题：

```java
SynchronizedRGB color =
    new SynchronizedRGB(0, 0, 0, "Pitch Black");
...
int myColorInt = color.getRGB();      //Statement 1
String myColorName = color.getName(); //Statement 2
```

上面的代码中，线程A执行了Statement1，紧接着线程B调用Set或者Invert方法改变了name，那么线程A再执行Statement2，拿到的数据就不匹配了。所以要加上synchronized:

```java
synchronized (color) {
    int myColorInt = color.getRGB();
    String myColorName = color.getName();
}
```

所以说，可变对象就是可能发生这种问题。

### 定义不可变对象

下面是一些创建不可变对象的简单策略。并不一定要全部遵守，但是需要经过充分的分析，确保对象创建后不可修改。简单的思路就是，把所有会改变对象域的操作都堵死：

1. **不提供setter方法**，包括修改字段和修改字段引用对象的方法。很好理解。

2. 不允许子类重写方法。简单的办法是**声明类为final**，更优雅的做法是构造方法私有，然后用工厂生成对象。这个也好理解，防止子类重写方法来修改字段值。

3. **将所有类字段定义成final、private**。定义成final当然是防止字段改变，定义成public则是为了防止外部代码改变引用对象的值。因为final的引用对象只是引用不会改变，值还是会改变。

4. **如果类的字段是对可变对象的引用，则不允许修改被引用对象**。

   ​		不提供修改可变对象的方法。

   ​    	不共享可变对象的引用。当引用从构造方法传入时，不保存引用，要保存的话只保存它的拷贝。当方法需要返回引用对象时，不直接返回，只返回拷贝。

然后将上面的原则用在我们的例子上，主要可以做以下修改：

1. SynchronizedRGB类有两个setter方法。第一个set方法直接删掉，第二个invert方法修改为创建一个新对象，而不是在原有对象上修改。
2. 所有的字段都已经是private的，加上final即可。
3. 将类声明为final的
4. 只有一个字段是对象引用，并且被引用的对象也是不可变对象。

``` java
final public class ImmutableRGB {

    // Values must be between 0 and 255.
    final private int red;
    final private int green;
    final private int blue;
    final private String name;

    private void check(int red,
                       int green,
                       int blue) {
        if (red < 0 || red > 255
            || green < 0 || green > 255
            || blue < 0 || blue > 255) {
            throw new IllegalArgumentException();
        }
    }

    public ImmutableRGB(int red,
                        int green,
                        int blue,
                        String name) {
        check(red, green, blue);
        this.red = red;
        this.green = green;
        this.blue = blue;
        this.name = name;
    }

    public int getRGB() {
        return ((red << 16) | (green << 8) | blue);
    }

    public String getName() {
        return name;
    }

    public ImmutableRGB invert() {
        return new ImmutableRGB(255 - red,
                       255 - green,
                       255 - blue,
                       "Inverse of " + name);
    }
}
```

参考资料

[并发编程网 – ifeve.com/oracle-java-concurrency-tutorial](<http://ifeve.com/oracle-java-concurrency-tutorial/>)

<https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html>