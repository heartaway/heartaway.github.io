---
layout: post
title: 透过序列化字节码看Java序列化
categories: Java
description:
keywords: serializable 序列化 字节码 
date: 2016-03-09
---

Java序列化的基础知识 请参考之前的文章 'Java基础 之 序列化与反序列化'

**序列化数据的存储结构：**

Java序列化后存储的信息包括：类元数据描述、类的属性、父类信息以及属性域的值。

编写一个测试类：


```java
public class SerializableTest implements Serializable {
    private int father;
    private static final long serialVersionUID = 1937803012639770720L;
    private class ObjectSaver  extends  SerializableTest implements Serializable{
        private static final long serialVersionUID = -1460368089309853877L;
        public String name;
        public int old;
        public ObjectSaver(String name, int old) {
            super.father = 1;
            this.name = name;
            this.old = old;
        }
    }
}
```

通过Junit进行序列化，生成序列化后的对象：


```java
@Test
public void testWriteObject() throws Exception{
    ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("/Users/xinyuan/tmp/ObjectSaver.obj"));
    objectOutputStream.writeObject("严明明");
    objectOutputStream.writeObject(new Date());
    objectOutputStream.writeObject(new ObjectSaver("测试类",30));
    objectOutputStream.close();
}
```

使用strings打开ObjectSaver.obj 文件，可以看到存储下来的可打印的字符信息如下：


```java
java.util.Datehj
Scxsr
7com.java.demo.serializable.SerializableTest$ObjectSaver
oldL
namet
Ljava/lang/String;L
this$0t
-Lcom/java/demo/serializable/SerializableTest;xr
+com.java.demo.serializable.SerializableTest
fatherxp
```

二进制内容为：


```java
aced 0005 7400 09e4 b8a5 e698 8ee6 988e
7372 000e 6a61 7661 2e75 7469 6c2e 4461
7465 686a 8101 4b59 7419 0300 0078 7077
0800 0001 5356 b659 b778 7372 0037 636f
6d2e 6a61 7661 2e64 656d 6f2e 7365 7269
616c 697a 6162 6c65 2e53 6572 6961 6c69
7a61 626c 6554 6573 7424 4f62 6a65 6374
5361 7665 72eb bbba f5cb 5ec7 4b02 0003
4900 036f 6c64 4c00 046e 616d 6574 0012
4c6a 6176 612f 6c61 6e67 2f53 7472 696e
673b 4c00 0674 6869 7324 3074 002d 4c63
6f6d 2f6a 6176 612f 6465 6d6f 2f73 6572
6961 6c69 7a61 626c 652f 5365 7269 616c
697a 6162 6c65 5465 7374 3b78 7200 2b63
6f6d 2e6a 6176 612e 6465 6d6f 2e73 6572
6961 6c69 7a61 626c 652e 5365 7269 616c
697a 6162 6c65 5465 7374 1ae4 7592 b513
4860 0200 0149 0006 6661 7468 6572 7870
0000 0001 0000 001e 7400 09e6 b58b e8af
95e7 b1bb 7371 007e 0006 0000 0000
```

##### 第一部分：

**aced** STREAM_MAGIC 流的幻数，用于标识序列化协议；

**0005**  STREAM_VERSION 标识序列化协议的版本号；

一些标识性字符可以参考类：ObjectStreamConstants
ObjectOutputStream.writeStreamHeader()方法：


```java
protected void writeStreamHeader() throws IOException {
bout.writeShort(STREAM_MAGIC);
bout.writeShort(STREAM_VERSION);
}
```

##### 第二部分：

```java
objectOutputStream.writeObject(“严明明”)
```

1. 74  标识TC_STRING
2. 00 09 标识第一个String的长度为9个字节
3. e4 b8a5 e698 8ee6 988e 表示String类型的值：严明明


```java
objectOutputStream.writeObject(new Date())
```

1. 7372 标识TC_OBJECT 和 TC_CLASSDESC
2. 000e  标识类名称长度为14个字节
3. 6a61 7661 2e75 7469 6c2e 4461 7465 表示类型值：java.util.Date
4. 686a 8101 4b59 7419  标识uid对象序列化ID的类型为long型，占用8个字节
5. 03 这一个字节可能有十种值标识：参考java.io.ObjectStreamConstants#SC_*
6. 0000 标识类属性个数，因为是Date类型，所以没有自定义属性
7. 78 标识域类型TC_ENDBLOCKDATA ，因为属性个数为0，所以对象数据结束
8. 70 再没有父类的标识
9. 77  对象数据块开始，TC_BLOCKDATA
10. 08 标识数据长度为8个字节
11. 00 0001 5356 b659 b7 标识new Date()的对象时间戳long型，占用8个字节
12. 78 TC_ENDBLOCKDATA 对象数据库结束标识


```java
objectOutputStream.writeObject(new ObjectSaver(“测试类”,30))
```

1. 7372 0037 表示TC_OBJECT 和 TC_CLASSDESC ，且类名称长度为55个字节
2. 636f
     ​    6d2e 6a61 7661 2e64 656d 6f2e 7365 7269
     ​    616c 697a 6162 6c65 2e53 6572 6961 6c69
     ​    7a61 626c 6554 6573 7424 4f62 6a65 6374
     ​    5361 7665 72
     ​    表示类名值：com.java.demo.serializable.SerializableTest$ObjectSaver
3. eb bbba f5cb 5ec7 4b 标识ObjectSaver对象序列化ID的类型为long型，占用8个字节 ，值为-1460368089309853877L，因为uid为静态属性，所以属于类元信息一部分
4. 02 这一个字节可能有十种值标识，其中02标识SC_SERIALIZABLE，此类继承了Serializable接口
5. 0003 标识类属性个数为 3 个，包含this；
6. 49 域类型，49转十进制为73，73在ASC码中对应的是 I ，因为old为int类型
7. 00 03 标识属性名称长度为3个字节
8. 6f 6c64 标识属性名称为 old
9. 4c 域类型，4c转十进制为76，76在ASC码中对应的是 L，因为name为String对象
10. 00 04 标识属性名称长度为4个字节,name字符占四个字符
11. 6e 616d 65 标识属性名称的值 name
12. 74 标识TC_STRING一个新的字符串
13. 0012 域类型长度为18个字节
14. 4c6a 6176 612f 6c61 6e67 2f53 7472 696e
    ​       673b
    ​       对象类型签名 Ljava/lang/String;  包含封号
15. 4c 域类型，4c转十进制为76，76在ASC码中对应的是 L
16. 00 06 标识属性名称长度为6个字节
17. 74 6869 7324 30  标识字符串 this$0
18. 74 标识TC_STRING一个新的字符串
19. 002d 域类型长度为45个字节
20. 4c63
    ​       6f6d 2f6a 6176 612f 6465 6d6f 2f73 6572
    ​       6961 6c69 7a61 626c 652f 5365 7269 616c
    ​       697a 6162 6c65 5465 7374 3b
    ​       Lcom/java/demo/serializable/SerializableTest;
21. 78  标识域类型TC_ENDBLOCKDATA
22. `上面为之类ObjectSaver的描述信息和元信息，下面为父类SerializableTest的描述信息和元信息`
23. 72 标识TC_CLASSDESC
24. 002b 标识类名称长度为43个字节
25. 63
    ​       6f6d 2e6a 6176 612e 6465 6d6f 2e73 6572
    ​       6961 6c69 7a61 626c 652e 5365 7269 616c
    ​       697a 6162 6c65 5465 7374
    ​       标识类字符串：com.java.demo.serializable.SerializableTest
26. 1ae4 7592 b513 4860  标识ObjectSaver对象序列化ID的类型为long型，占用8个字节 ，值为-1937803012639770720L
27.  02 这一个字节可能有十种值标识，其中02标识SC_SERIALIZABLE，此类继承了Serializable接口
28. 00 01 标识类属性个数为1个
29. 49 域类型，49转十进制为73，73在ASC码中对应的是 I ，因为old为int类型
30. 0006 标识属性名称长度为6个字节
31. 6661 7468 6572 标识father六个字符；
32. 7870 TC_ENDBLOCKDATA 对象数据库结束标识，且没有父类

##### 第三部分：
接下来是对象属性域的值部分，按照从父类到子类的顺序写入域的值

1. 0000 0001 标识的十进制为1，对应父类father的值为1
2. 0000 001e 标识十进制为30，标识之类old的值为30
3. 74 标识TC_STRING一个新的字符串
4. 0009 标识占用9个字节
5. e6b5 8be8 af95 e7b1 bb 标识字符“测试类”
6. 73 71  标识TC_OBJECT  TC_REFERENCE
7. 007e 0006 0000 0000

#### 总结 Java序列化算法的基本步骤
* 输出序列化的头部信息，包括序列化协议的幻数和版本；
* 基本类型按照一字节的类型标识、两字节类型长度、N个字节值
* 基本对象类型，7372标识OBJECT和CLASSDESC，两字节类型长度，8字节uid，一些辅助信息
* 复杂对象类型，第一步按照由子类到父类的顺序，递归的输出类的描述信息，知道不再有父类为止；类描述信息按照类元数据，类属性信息的顺序写入序列化流中；第二步按照由父类到之类的顺序，递归的输出对象域对象域的实际数据值；而对象的属性信息是按照基本类型到java对象类型的顺序写入序列化流中，其中java对象类型的属性会从第一步重新开始递归的输出，知道不再存在java对象类型的属性。

