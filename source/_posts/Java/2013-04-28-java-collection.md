---
layout: post
title: Java基础 之 集合
categories: Java
description:
keywords: collection 集合
date: 2013-04-28
---
Java 中的广义集合分两大类，Collection 和 Map ;

#### Set : 成员不能重复

HashSet: 外部无序地遍历成员；覆盖了equals方法，注意修改hashCode方法。
TreeSet：外部有序地遍历成员；成员要求实现caparable接口，或者使用 Comparator构造TreeSet。
LinkedHashSet：外部按成员的插入顺序遍历成员。

#### List：提供基于索引对成员随机访问。

LinkedList可以实现 队列(queue)，栈（stack），双向队列（deque）。注意LinkedList没有同步方法。
ArrayList 实现了可变大小的数组.ArrayList缺省情况下自动增长原来一倍的50%,
Vector非常类似ArrayList，但是Vector是同步的，ArrayList是非同步的。Vector缺省情况下自动增长原来一倍的数组长度。

#### Map:  提供一组key-value的键值对

Hashtable：是同步的。不允许null为key或value。查找value是通过对key进行散列计算，所以确保key对象，要同时复写equals方法和hashCode方法，而不要只写其中一个。Hashtable原理：通过节点的关键码确定节点的位置，即给定的关键码K，通过一定的函数关系H(散列函数)，得到函数值H(K)，将此值解释为这个节点的存储位置。
HashMap:非同步的。允许null 为key 或value。
WeakHashMap：是一种改进的HashMap，它对key实行“弱引用”，如果一个key不再被外部所引用，那么该key可以被GC回收。
IdentityHashMap：它在比较键时，使用的是引用等价而不是值等价性 ，而HashMap使用的是值等价性


### 使用总结：

1. 如果涉及 堆栈，队列等操作，应该考虑使用List，对需要快速插入元素、删除元素的 使用LinkedList，如果需要快速随机访问元素，应该使用ArrayList
2. 如果程序在单线程的环境中，或者访问仅仅在一个线程中心进行，那就可以考虑使用非同步的类，执行效率高，如果在多线程可能同时操作一个类，应该使用 同步的类；
3. 要特别注意哈希表的使用，作为key的对象要正确覆写equals 和 hashcode 方法。
4. 尽量返回接口而非实际的类型，如返回List而非ArrayList，这样如果以后需要将ArrayList换成LinkedList时，客户端代码不用改变。
5. 如果一个HashSet  ,Hashtable  或 HashMap被序列化，那么请确认他们的内容没有直接或间接地引用他们自身。
6. 在对List进行循环的时候不要在循环体内对List元素进行remove操作；

### 探究实现原理：
#### ArrayList
**实现原理：**内部维护了一个数组 和 int 型 大小变量 size；

**动态扩容原理：**没有显示定义容量的时候，默认开辟10个数组大小空间；每次add的时候调用ensureCapacity方法，判断添加一个元素后是否超过现有容量，如果超过，那么新开辟之前容量的1.5倍，然后将之前的数据拷贝的新的数组当中.

**总结：**可以看出进行动态扩容实际是进行数组拷贝，这个就会花费一定的时间消耗，所以当我们的数据比较大或者能够与之数据大小的时候，做好提前做一次ensureCapacity，或定义时指定。

#### LinkedList
**实现原理:**底层采用双向循环列表实现。(可以使用LinkedList来实现队列和栈)

**Entry类数据模型：**一个数据element，两个指针，一个指向前一个节点，名为previous，一个指向下一个节点，名为next，形成双向列表，LinkedList的无参构造函数初始化的时候就实现了链表的循环。


```java
header.next = header.previous = header
```

#### HashSet 
**实现原理：**HashSet的底层采用HashMap来存放数据，在add的时候调用hashMap的put方法，对应hashmap的key是hashSet的值，value是一个常量对象PRESENT


```java
private static final Object PRESENT = new Object();
```

它的原理就是利用 hashMap的key是不可重复的这一特性来用key存储Set的元素值以达到Set的元素不可重复。

#### HashMap 
**实现原理:** 底层采用 数组链表 作为数据存储模型；Entry[] table；

**Entry类数据模型：**一个泛型Key，一个泛型value，一个指向下一个Entry的指针 next，一个 hash int型值。

put操作：

首先判断key是否为null，为null的话就会去table[0]链循环查找key为null的节点Entry，找到则替换value，无则添加；其次对key的hashcode进行hash得到数组下标 i ，遍历 i 链上的所有节点，判断是否有key重复的，有则替换value，并返回，无则继续；最后在 数组中添加 下标为 i 的 Entry链中第一个节点出添加这个节点。Entry[]的长度一定后，随着map里面数据的越来越长，这样同一个i的链就会很长，会不会影响性能？HashMap里面设置一个因素（也称为因子），随着map的size越来越大，Entry[]会以一定的规则加长长度。

get操作：

首先判断key是否为null，为null的话就去table[0]链循环查找key为null的节点Entry。不为null，就根据key的hashcode进行hash得到数据下标i ，遍历 下标为 i 的链，查找key元素，如果不存在就返回null。

#### Hashtable
**实现原理：**底层采用 数组链表 作为数据存储模型；Entry[] table；
HashTable和HashMap采用相同的存储机制，二者的实现基本一致.
在Hashtable中调用put方法时，如果key为null，直接抛出NullPointerException
Hashtable是线程安全的，内部的方法基本都是synchronized.

#### ConcurrentHashMap
**实现原理：**基于一个叫Segment数组的，其实和Entry类似。

ConcurrentHashMap基于concurrentLevel划分出了多个Segment来对key-value进行存储，从而避免每次锁定整个数组，在默认的情况下，允许16个线程并发无阻塞的操作集合对象，尽可能地减少并发时的阻塞现象。

#### WeakHashMap
**实现原理：**内部类Entry继承了WeakReference类，且在构造函数中构造了Key的弱引用，当进行put或者get操作时，都会调用一个函数叫expungeStaleEntries()用来判断，如果key存在弱引用，则进行垃圾回收。

WeakHashMap在不考了命中率的情况下，多用于缓存系统，就是说在系统内存紧张的时候可随时进行GC，但是如果内存不紧张则可以用来存放一些缓存数据。因为如果使用HashMap的话，它里面的值基本都是强引用，即使内存不足，它也不会进行GC。

### 补充：
1. 集合的排序：Comparator & Comparable
 当需要排序的集合或数组不是纯数字时，通常使用 Comparator 或 Comparable 实现对象排序或者自定义排序
一个类实现了Camparable接口则表明这个类的对象之间是可以相互比较的，这个类对象组成的集合就可以直接使用sort方法排序。
Comparator可以看成一种算法的实现，将算法和数据分离；
2. 集合工具类：Collections（一系列算法的集合，里面属性都是static）
    * 生成单元素集合：    Collections.singletonList、Collections.singletonMap、Collections.singleton
    * Checked集合：checkedCollection、checkedList、checkedMap、checkedSet、checkedSortedMap、checkedSortedSet
    * 同步集合：为一些非线程安全的集合类提供同步机制。
    * 查找替换：fill、frequency、indexOfSubList、lastIndexOfSubList、max、min、replaceAll
    * 集合排序：reverse、shuffle、sort、swap、rotate
3. hashCode()方法的作用：
hashCode() 是用来产生哈希码的，而哈希码是用来在散列存储结构中确定对象的存储地址的。对于Object类的 hashCode方法，会针对不同对象返回不同的整数，这一般是通过将该对象的内部地址转换为一个整数来实现的，当两个对象是一样的时（equals为true），他们的hashCode码也是相同的。String的hashCode返回值是基于String的内容的，内容相同，hashCode就相同。在集合中很多地方使用了hashCode，比如保证hashSet的元素不重复，保证hashMap的key不重复，提高hashMap存取数据的效率。

