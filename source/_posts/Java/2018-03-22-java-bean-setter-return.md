---
layout: post
title: POJO 对象setter 方法是否合适return "this"
categories: Java
description:
keywords: setter
date: 2018-03-22
---


通常的POJO对象setter方法return 为void


```java
    public void setStatus(String status) {
        this.status = status;
    }
```

但是面对对象属性填充时，一堆的set方法让代码看起来很臃肿，部分同学采用类build模式，对setter方法进行改造，改造后就可以使用链式处理简化属性设置；

```java
    public Employee setName(String name) {
        this.name = name;
        return this;
    }
```

流式编码风格如：

```java
new Employee().setName("Xin yuan").setHeight(178);
```

有人认为这让pojo方法设置变得更加便捷，有其使用之处，但是也有人认为`return this`打破了[Java Bean](https://docstore.mik.ua/orelly/java-ent/jnut/ch06_02.htm)的规约，破坏了每个函数单一职责的原则，也可能会破坏一些工具库的使用，或组织JVM做一些优化。而且在IDE中getter&setter的自动生成内容中并没有“return this”。综其所述，setter方法“return this”并不建议。

当然还有其它选择方案：

**方法一：**

采用java内置语法：

```java
new Employee()
{{
    setName("Jack Sparrow");
    setId(1);
    setFoo("bacon!");
}});
```

**方法二：**

采用更加复杂的内部类builder模式：

```java
    public Employee setName(String name) {
        this.name = name;
        return this;
    }
    
    public static class Builder {
        private String name;
          public Builder name(Strig name) {
            this.name = name;
            return this;
        }
    }
```
造者模式(Builder Pattern)：将一个**复杂对象**的**构建**与它的**表示**分离，使得同样的构建过程可以创建不同的表示。但是一般POJO 类都比较简单，非复杂对象，所以采用builder模式并不会让代码变得简洁，反而会显得更加臃肿。


**方法三：**

属性新增withXxx方法：

```java
    public Employee setName(String name) {
        this.name = name;
        return this;
    }
    
    public Employee withName(String name) {
        setName(name);
        return this;
    }
    //使用方式
    new Employee().withName("Xin yuan").withHeight(178);
    
```

### 结论：
不建议在POJO： Java Bean的setter方法中添加‘return this’，我们遵循Java规范，如果期望简化Java Bean的属性设置，可以采用with或build方法。


**参考：**

[https://en.wikipedia.org/wiki/Plain_old_Java_object](https://en.wikipedia.org/wiki/Plain_old_Java_object)

[https://stackoverflow.com/questions/1345001/is-it-bad-practice-to-make-a-setter-return-this](https://stackoverflow.com/questions/1345001/is-it-bad-practice-to-make-a-setter-return-this)

