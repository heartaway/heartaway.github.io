---
layout: post
title: Java基础 之 序列化与反序列化
categories: Java
description:
keywords: serialize deserialize 序列化 反序列化
date: 2013-04-29
---
#### 为什么需要对象序列化：
解决Java对象在网络上传输和Java对象持久化的问题。序列化将对象转换为二进制流，然后在网络上传输，当抵打目的后在反序列化为Java对象。
#### 什么是Java对象序列化
　　Java平台允许我们在内存中创建可复用的Java对象，但一般情况下，只有当JVM处于运行时，这些对象才可能存在，即，这些对象的生命周期不会比JVM的生命周期更长。但在现实应用中，就可能要求在JVM停止运行之后能够保存（持久化）指定的对象，并在将来重新读取被保存的对象。Java对象序列化就能够帮助我们实现该功能。
　　
　　使用Java对象序列化，在保存对象时，会把其状态保存为一组字节，在未来，再将这些字节组装成对象。必须注意地是，对象序列化保存的是对象的”状态”，即它的成员变量。由此可知，对象序列化不会关注类中的静态变量。
　　除了在持久化对象时会用到对象序列化之外，当使用RMI（远程方法调用），或在网络中传递对象时，都会用到对象序列化。Java序列化API为处理对象序列化提供了一个标准机制，该API简单易用，在本文的后续章节中将会陆续讲到。
　　
#### Java对象的序列化和反序列化实践
当两个进程在进行远程通信时，彼此可以发送各种类型的数据。无论是何种类型的数据，都会以二进制序列的形式在网络上传送。发送方需要把这个 Java 对象转换为字节序列，才能在网络上传送；接收方则需要把字节序列再恢复为 Java 对象。

把 Java 对象转换为字节序列的过程称为对象的序列化。
把字节序列恢复为 Java 对象的过程称为对象的反序列化。

##### 对象的序列化主要有两种用途：
1. 把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中；
2. 在网络上传送对象的字节序列。

只有实现了 Serializable 和 Externalizable 接口的类的对象才能被序列化。 Externalizable 接口继承自 Serializable 接口，实现Externalizable 接口的类完全由自身来控制序列化的行为，而仅实现Serializable 接口的类可以采用默认的序列化方式 。

##### 对象序列化包括如下步骤：
1. 创建一个对象输出流，它可以包装一个其他类型的目标输出流，如文件输出流；
2. 通过对象输出流的 writeObject() 方法写对象。
##### 对象反序列化的步骤如下：
1. 创建一个对象输入流，它可以包装一个其他类型的源输入流，如文件输入流；
2. 通过对象输入流的 readObject() 方法读取对象。
下面让我们来看一个对应的例子，类的内容如下：


```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.util.Date;

public class ObjectSaver {
    public static void main(String[] args) throws Exception {
        ObjectOutputStream out = new ObjectOutputStream
            (new FileOutputStream("D:""objectFile.obj"));
        // 序列化对象
        Customer customer = new Customer(" 阿蜜果 ", 24);
        out.writeObject(" 你好 !");
        out.writeObject(new Date());
        out.writeObject(customer);
        out.writeInt(123); // 写入基本类型数据
        out.close();
        // 反序列化对象
        ObjectInputStream in = new ObjectInputStream
            (new FileInputStream("D:""objectFile.obj"));
        System.out.println("obj1=" + (String)in.readObject());
        System.out.println("obj2=" + (Date)in.readObject());
        Customer obj3 = (Customer)in.readObject();
        System.out.println("obj3=" + obj3);
        int obj4 = in.readInt();
        System.out.println("obj4=" + obj4);
        in.close();
    }
}

class Customer implements Serializable {
    private String name;
    private int age;

    public Customer(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String toString() {
        return "name=" + name + ", age=" + age;
    }
}
```

输出结果如下：


```
obj1= 你好 !
obj2=Sat Sep 15 22:02:21 CST 2007
obj3=name= 阿蜜果 , age=24
obj4=123
```

因此例比较简单，在此不再详述。


##### 实现 Serializable 接口
ObjectOutputStream 只能对 Serializable 接口的类的对象进行序列化。默认情况下，ObjectOutputStream 按照默认方式序列化，这种序列化方式仅仅对对象的非 transient 的实例变量进行序列化，而不会序列化对象的 transient 的实例变量，也不会序列化静态变量。

当 ObjectOutputStream 按照默认方式反序列化时，具有如下特点：

1. 如果在内存中对象所属的类还没有被加载，那么会先加载并初始化这个类。如果在 classpath 中不存在相应的类文件，那么会抛出 ClassNotFoundException ；
2. 在反序列化时不会调用类的任何构造方法。
如果用户希望控制类的序列化方式，可以在可序列化类中提供以下形式的 writeObject() 和 readObject() 方法。


```java
private void writeObject(java.io.ObjectOutputStream out) throws IOException
private void readObject(java.io.ObjectInputStream in) throws IOException, ClassNotFoundException;
```

当 ObjectOutputStream 对一个 Customer 对象进行序列化时，如果该对象具有 writeObject() 方法，那么就会执行这一方法，否则就按默认方式序列化。在该对象的 writeObjectt() 方法中，可以先调用ObjectOutputStream 的 defaultWriteObject() 方法，使得对象输出流先执行默认的序列化操作。同理可得出反序列化的情况，不过这次是 defaultReadObject() 方法。

有些对象中包含一些敏感信息，这些信息不宜对外公开。如果按照默认方式对它们序列化，那么它们的序列化数据在网络上传输时，可能会被不法份子窃取。

对于这类信息，可以对它们进行加密后再序列化，在反序列化时则需要解密，再恢复为原来的信息。

默认的序列化方式会序列化整个对象图，这需要递归遍历对象图。如果对象图很复杂，递归遍历操作需要消耗很多的空间和时间，它的内部数据结构为双向列表。

在应用时，如果对某些成员变量都改为 transient 类型，将节省空间和时间，提高序列化的性能。

##### 实现 Externalizable 接口

Externalizable 接口继承自 Serializable 接口，如果一个类实现了Externalizable 接口，那么将完全由这个类控制自身的序列化行为。

Externalizable 接口声明了两个方法：


```java
public void writeExternal(ObjectOutput out) throws IOException
public void readExternal(ObjectInput in) throws IOException , ClassNotFoundException
```

前者负责序列化操作，后者负责反序列化操作。

在对实现了 Externalizable 接口的类的对象进行反序列化时，会先调用类的不带参数的构造方法，这是有别于默认反序列方式的。

如果把类的不带参数的构造方法删除，或者把该构造方法的访问权限设置为 private 、默认或 protected 级别，
会抛出 java.io.InvalidException: no valid constructor 异常。

##### 可序列化类的不同版本的序列化兼容性
凡是实现 Serializable 接口的类都有一个表示序列化版本标识符的静态变量：
private static final long serialVersionUID;
以上 serialVersionUID 的取值是 Java 运行时环境根据类的内部细节自动生成的。如果对类的 源代码 作了修改，再重新编译，新生成的类文件的serialVersionUID 的取值有可能也会发生变化。

类的 serialVersionUID 的默认值完全依赖于 Java 编译器的实现，对于同一个类，用不同的 Java 编译器编译，有可能会导致不同的serialVersionUID ，也有可能相同。为了提高哦啊 serialVersionUID 的独立性和确定性，强烈建议在一个可序列化类中显示的定义serialVersionUID ，为它赋予明确的值。显式地定义 serialVersionUID 有

##### 两种用途：
1. 在某些场合，希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有相同的 serialVersionUID ；
2. 在某些场合，不希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有不同的 serialVersionUID 。

##### “transient”——“瞬态
###### 适用场景：
1. 不打算序列化某字段的值，节省空间
2. 传递序列化流的时候，不传递该值等
一个实现了serializable的单例类，必须有一个readResolve方法，用以返回他的唯一实例。

有可能由于对于一个实现了Serializable的类进行了扩展，或者由于实现了一个扩展自Serializable的接口，使得我们在无意中实现了Serializable。

### 复杂对象序列化

* 当父类继承Serializable接口，所有子类都可以被序列化
* 子类实现了Serializable接口，父类没有，父类中的属性不能序列化（不报错，数据会丢失），但是子类中属性人能正确序列化
* 如果序列化的属性是对象，这个对象也必须实现Serializable接口，否则会报错
* 在反序列化时，如果对象的属性有修改或删减，修改的部分属性会丢失，但不会报错
* 在反序列化时，如果serialVersionUID被修改，那么反序列化时会失败
* 如果一个父类没有实现Serializable接口，他的内部类如果不是static的，即使实现了序列化接口，也会序列失败。因为非静态
内部类会保存一个指向父类的类型this变量，而序列化类的所有属性必须实现序列化接口，所以要将内部类设置成静态类
* List或者Map容器中包含的泛型类型也必须实现Serializable接口，否则也会报java.io.NotSerializableException


