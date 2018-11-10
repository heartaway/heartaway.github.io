---
layout: post
title: Java基础 之 super 和 this
categories: Java
description:
keywords: super this
date: 2013-03-18
---
super的定义：

>The super keyword enables a [subclass](http://java.about.com/od/s/g/subclass.htm) to call the methods and fields of its [superclass](http://java.about.com/od/s/g/superclass.htm). It is not an instance of the superclass object but a way to tell the compiler which methods or fields to reference. The effect is the same as if the subclass is calling one of its own methods.

this 是对自身对象的一个地址引用，调用的是 invokevirtual。

super 是类似引用 但不是引用的调用，因为 super 实际上是 调用invokespecial 。


```java
C extends B,  B extends A;
A a = new C();
```

那么为C在堆中分配的内存区域中，分为三块，一块是C自己的，一块是B自己的，一块是A的，在C、B、A中都会有连个变量，super 和 this ；this指向当前区域，super指向父类的内存区域；

从内存分配可以看出this和super变量是属于对象的，所以不能在静态块和静态方法中使用。

#### this使用场景：
1. 通过this调用另一个构造方法，用法是this(参数列表)，这个仅仅在类的构造方法中，别的地方不能这么用。
2. 函数参数或者函数中的局部变量和成员变量同名的情况下，成员变量被屏蔽，此时要访问成员变量则需要用“this.成员变量名”的方式来引用成员变量。当然，在没有同名的情况下，可以直接用成员变量的名字，而不用this，用了也不为错。
3. 在函数中，需要引用该函所属类的当前对象时候，直接用this。

#### super的用法：
1. 在子类构造方法中要调用父类的构造方法，用“super(参数列表)”的方式调用，参数不是必须的。同时还要注意的一点是：“super(参数列表)”这条语句只能用在子类构造方法体中的第一行。
2. 当子类方法中的局部变量或者子类的成员变量与父类成员变量同名时，也就是子类局部变量覆盖父类成员变量时，用“super.成员变量名”来引用父类成员变量。当然，如果父类的成员变量没有被覆盖，也可以用“super.成员变量名”来引用父类成员变量，不过这是不必要的。
3. 当子类的成员方法覆盖了父类的成员方法时，也就是子类和父类有完全相同的方法定义（但方法体可以不同），此时，用“super.方法名(参数列表)”的方式访问父类的方法。

