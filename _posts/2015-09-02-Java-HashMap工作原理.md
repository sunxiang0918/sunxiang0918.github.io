---
title: <转>Java HashMap工作原理
date: 2015-09-02 23:00:38
tags:
- JAVA
comments: true
---

Java中HashMap的工作原理其实以前就看过很多次了,也仔细的分析过它的源码.但是一直没有写一篇文章来记录一下,这篇文章写的非常的清楚,非常的透彻.是我见过讲的最详细的.值得保存下来.

# Java HashMap工作原理 

大部分Java开发者都在使用Map，特别是HashMap。HashMap是一种简单但强大的方式去存储和获取数据。但有多少开发者知道HashMap内部如何工作呢？几天前，我阅读了java.util.HashMap的大量源代码（包括Java 7 和Java 8），来深入理解这个基础的数据结构。在这篇文章中，我会解释java.util.HashMap的实现，描述Java 8实现中添加的新特性，并讨论性能、内存以及使用HashMap时的一些已知问题。

## 内部存储

Java HashMap类实现了Map<K, V>接口。这个接口中的主要方法包括：

* V put(K key, V value)
* V get(Object key)
* V remove(Object key)
* Boolean containsKey(Object key)

HashMap使用了一个内部类Entry<K, V>来存储数据。这个内部类是一个简单的键值对，并带有额外两个数据：

* 一个指向其他入口（译者注：引用对象）的引用，这样HashMap可以存储类似链接列表这样的对象。
* 一个用来代表键的哈希值，存储这个值可以避免HashMap在每次需要时都重新生成键所对应的哈希值。

<!--more-->

下面是Entry<K, V>在Java 7下的一部分代码：

```java
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
…
}
```

HashMap将数据存储到多个单向Entry链表中（有时也被称为桶bucket或者容器orbins）。所有的列表都被注册到一个Entry数组中（Entry<K, V>[]数组），这个内部数组的默认长度是16。

下面这幅图描述了一个HashMap实例的内部存储，它包含一个nullable对象组成的数组。每个对象都连接到另外一个对象，这样就构成了一个链表。

![](/img/2015/09/02/1.jpg) 

所有具有相同哈希值的键都会被放到同一个链表（桶）中。具有不同哈希值的键最终可能会在相同的桶中。

当用户调用 put(K key， V value) 或者 get(Object key) 时，程序会计算对象应该在的桶的索引。然后，程序会迭代遍历对应的列表，来寻找具有相同键的Entry对象（使用键的equals()方法）。

对于调用get()的情况，程序会返回值所对应的Entry对象（如果Entry对象存在）。

对于调用put(K key, V value)的情况，如果Entry对象已经存在，那么程序会将值替换为新值，否则，程序会在单向链表的表头创建一个新的Entry（从参数中的键和值）。

桶（链表）的索引，是通过map的3个步骤生成的：

* 首先获取键的**散列码**。
* 程序**重复**散列码，来阻止针对键的糟糕的哈希函数，因为这有可能会将所有的数据都放到内部数组的相同的索引（桶）上。
* 程序拿到重复后的散列码，并对其使用数组长度（最小是1）的**位掩码（bit-mask）**。这个操作可以保证索引不会大于数组的大小。你可以将其看做是一个经过计算的优化取模函数。

下面是生成索引的源代码：

```java
// the "rehash" function in JAVA 7 that takes the hashcode of the key
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
// the "rehash" function in JAVA 8 that directly takes the key
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
// the function that returns the index from the rehashed hash
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

为了更有效地工作，内部数组的大小必须是2的幂值。让我们看一下为什么：

假设数组的长度是17，那么掩码的值就是16（数组长度-1）。16的二进制表示是0…010000，这样对于任何值H来说，“H & 16”的结果就是16或者0。这意味着长度为17的数组只能应用到两个桶上：一个是0，另外一个是16，这样不是很有效率。但是如果你将数组的长度设置为2的幂值，例如16，那么按位索引的工作变成“H & 15”。15的二进制表示是0…001111，索引公式输出的值可以从0到15，这样长度为16的数组就可以被充分使用了。例如：

* 如果H = 952，它的二进制表示是0..01110111000，对应的索引是0…01000 = 8
* 如果H = 1576，它的二进制表示是0..011000101000，对应的索引是0…01000 = 8
* 如果H = 12356146，它的二进制表示是0..0101111001000101000110010，对应的索引是0…00010 = 2
* 如果H = 59843，它的二进制表示是0..01110100111000011，它对应的索引是0…00011 = 3

这种机制对于开发者来说是透明的：如果他选择一个长度为37的HashMap，Map会自动选择下一个大于37的2的幂值（64）作为内部数组的长度。

## 自动调整大小

在获取索引后，get()、put()或者remove()方法会访问对应的链表，来查看针对指定键的Entry对象是否已经存在。在不做修改的情况下，这个机制可能会导致性能问题，因为这个方法需要迭代整个列表来查看Entry对象是否存在。假设内部数组的长度采用默认值16，而你需要存储2，000,000条记录。在最好的情况下，每个链表会有125,000个Entry对象（2,000,000/16）。get()、remove()和put()方法在每一次执行时，都需要进行125,000次迭代。为了避免这种情况，HashMap可以增加内部数组的长度，从而保证链表中只保留很少的Entry对象。

当你创建一个HashMap时，你可以通过以下构造函数指定一个初始长度，以及一个loadFactor：

```java
public HashMap(int initialCapacity, float loadFactor)
```

如果你不指定参数，那么默认的initialCapacity的值是16， loadFactor的默认值是0.75。initialCapacity代表内部数组的链表的长度。

当你每次使用put(…)方法向Map中添加一个新的键值对时，该方法会检查是否需要增加内部数组的长度。为了实现这一点，Map存储了2个数据：

* Map的大小：它代表HashMap中记录的条数。我们在向HashMap中插入或者删除值时更新它。
* 阀值：它等于内部数组的长度*loadFactor，在每次调整内部数组的长度时，该阀值也会同时更新。

在添加新的Entry对象之前，put(…)方法会检查当前Map的大小是否大于阀值。如果大于阀值，它会创建一个新的数组，数组长度是当前内部数组的两倍。因为新数组的大小已经发生改变，所以索引函数（就是返回“键的哈希值 & (数组长度-1)”的位运算结果）也随之改变。调整数组的大小会创建两个新的桶（链表），并且将所有现存Entry对象重新分配到桶上。调整数组大小的目标在于降低链表的大小，从而降低put()、remove()和get()方法的执行时间。对于具有相同哈希值的键所对应的所有Entry对象来说，它们会在调整大小后分配到相同的桶中。但是，如果两个Entry对象的键的哈希值不一样，但它们之前在同一个桶上，那么在调整以后，并不能保证它们依然在同一个桶上。

![](/img/2015/09/02/2.jpg) 

这幅图片描述了调整前和调整后的内部数组的情况。在调整数组长度之前，为了得到Entry对象E，Map需要迭代遍历一个包含5个元素的链表。在调整数组长度之后，同样的get()方法则只需要遍历一个包含2个元素的链表，这样get()方法在调整数组长度后的运行速度提高了2倍。

## 线程安全

如果你已经非常熟悉HashMap，那么你肯定知道它不是线程安全的，但是为什么呢？例如假设你有一个Writer线程，它只会向Map中插入已经存在的数据，一个Reader线程，它会从Map中读取数据，那么它为什么不工作呢？

因为在自动调整大小的机制下，如果线程试着去添加或者获取一个对象，Map可能会使用旧的索引值，这样就不会找到Entry对象所在的新桶。

在最糟糕的情况下，当2个线程同时插入数据，而2次put()调用会同时出发数组自动调整大小。既然两个线程在同时修改链表，那么Map有可能在一个链表的内部循环中退出。如果你试着去获取一个带有内部循环的列表中的数据，那么get()方法永远不会结束。

**HashTable**提供了一个线程安全的实现，可以阻止上述情况发生。但是，既然所有的同步的CRUD操作都非常慢。例如，如果线程1调用get(key1)，然后线程2调用get(key2)，线程2调用get(key3)，那么在指定时间，只能有1个线程可以得到它的值，但是3个线程都可以同时访问这些数据。

从Java 5开始，我们就拥有一个更好的、保证线程安全的HashMap实现：**ConcurrentHashMap**。对于ConcurrentMap来说，只有桶是同步的，这样如果多个线程不使用同一个桶或者调整内部数组的大小，它们可以同时调用get()、remove()或者put()方法。**在一个多线程应用程序中，这种方式是更好的选择**。

## 键的不变性

为什么将字符串和整数作为HashMap的键是一种很好的实现？主要是因为它们是不可变的！如果你选择自己创建一个类作为键，但不能保证这个类是不可变的，那么你可能会在HashMap内部丢失数据。

我们来看下面的用例：

* 你有一个键，它的内部值是“1”。
* 你向HashMap中插入一个对象，它的键就是“1”。
* HashMap从键（即“1”）的散列码中生成哈希值。
* Map在新创建的记录中存储这个哈希值。
* 你改动键的内部值，将其变为“2”。
* 键的哈希值发生了改变，但是HashMap并不知道这一点（因为存储的是旧的哈希值）。
* 你试着通过修改后的键获取相应的对象。
* Map会计算新的键（即“2”）的哈希值，从而找到Entry对象所在的链表（桶）。
	* 情况1： 既然你已经修改了键，Map会试着在错误的桶中寻找Entry对象，没有找到。
	* 情况2： 你很幸运，修改后的键生成的桶和旧键生成的桶是同一个。Map这时会在链表中进行遍历，已找到具有相同键的Entry对象。但是为了寻找键，Map首先会通过调用equals()方法来**比较键的哈希值**。因为修改后的键会生成不同的哈希值（旧的哈希值被存储在记录中），那么Map没有办法在链表中找到对应的Entry对象。
	* 
下面是一个Java示例，我们向Map中插入两个键值对，然后我修改第一个键，并试着去获取这两个对象。你会发现从Map中返回的只有第二个对象，第一个对象已经“丢失”在HashMap中：

```java
public class MutableKeyTest {
 
    public static void main(String[] args) {
 
        class MyKey {
            Integer i;
 
            public void setI(Integer i) {
                this.i = i;
            }
 
            public MyKey(Integer i) {
                this.i = i;
            }
 
            @Override
            public int hashCode() {
                return i;
            }
 
            @Override
            public boolean equals(Object obj) {
                if (obj instanceof MyKey) {
                    return i.equals(((MyKey) obj).i);
                } else
                    return false;
            }
 
        }
 
        Map<MyKey, String> myMap = new HashMap<>();
        MyKey key1 = new MyKey(1);
        MyKey key2 = new MyKey(2);
 
        myMap.put(key1, "test " + 1);
        myMap.put(key2, "test " + 2);
 
        // modifying key1
        key1.setI(3);
 
        String test1 = myMap.get(key1);
        String test2 = myMap.get(key2);
 
        System.out.println("test1= " + test1 + " test2=" + test2);
 
    }
 
}
```

上述代码的输出是“test1=null test2=test 2”。如我们期望的那样，Map没有能力获取经过修改的键 1所对应的字符串1。

## Java 8 中的改进

在Java 8中，HashMap中的内部实现进行了很多修改。的确如此，Java 7使用了1000行代码来实现，而Java 8中使用了2000行代码。我在前面描述的大部分内容在Java 8中依然是对的，除了使用链表来保存Entry对象。在Java 8中，我们仍然使用数组，但它会被保存在Node中，Node中包含了和之前Entry对象一样的信息，并且也会使用链表：

下面是在Java 8中Node实现的一部分代码：

```java
static class Node<K,V> implements Map.Entry<K,V> {
     final int hash;
     final K key;
     V value;
     Node<K,V> next;
```

那么和Java 7相比，到底有什么大的区别呢？好吧，Node可以被扩展成TreeNode。TreeNode是一个红黑树的数据结构，它可以存储更多的信息，这样我们可以在O(log(n))的复杂度下添加、删除或者获取一个元素。下面的示例描述了TreeNode保存的所有信息：

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    final int hash; // inherited from Node<K,V>
    final K key; // inherited from Node<K,V>
    V value; // inherited from Node<K,V>
    Node<K,V> next; // inherited from Node<K,V>
    Entry<K,V> before, after;// inherited from LinkedHashMap.Entry<K,V>
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;
    boolean red;
```

红黑树是自平衡的二叉搜索树。它的内部机制可以保证它的长度总是log(n)，不管我们是添加还是删除节点。使用这种类型的树，最主要的好处是针对内部表中许多数据都具有相同索引（桶）的情况，这时对树进行搜索的复杂度是O(log(n))，而对于链表来说，执行相同的操作，复杂度是O(n)。

如你所见，我们在树中确实存储了比链表更多的数据。根据继承原则，**内部表中可以包含Node**（链表）或者**TreeNode（红黑树）**。Oracle决定根据下面的规则来使用这两种数据结构：

- 对于内部表中的指定索引（桶），如果node的数目多于8个，那么链表就会被转换成红黑树。

- 对于内部表中的指定索引（桶），如果node的数目小于6个，那么红黑树就会被转换成链表。

![](/img/2015/09/02/3.jpg) 

这张图片描述了在Java 8 HashMap中的内部数组，它既包含树（桶0），也包含链表（桶1，2和3）。桶0是一个树结构是因为它包含的节点大于8个。

## 内存开销

### JAVA 7

使用HashMap会消耗一些内存。在Java 7中，HashMap将键值对封装成Entry对象，一个Entry对象包含以下信息：

* 指向下一个记录的引用
* 一个预先计算的哈希值（整数）
* 一个指向键的引用
* 一个指向值的引用

此外，Java 7中的HashMap使用了Entry对象的内部数组。假设一个Java 7 HashMap包含N个元素，它的内部数组的容量是CAPACITY，那么额外的内存消耗大约是：

`sizeOf(integer)* N + sizeOf(reference)* (3*N+C)`

其中：

* 整数的大小是4个字节
* 引用的大小依赖于JVM、操作系统以及处理器，但通常都是4个字节。

这就意味着内存总开销通常是16 * N + 4 * CAPACITY字节。

注意：在Map自动调整大小后，CAPACITY的值是下一个大于N的最小的2的幂值。

注意：从Java 7开始，HashMap采用了延迟加载的机制。这意味着即使你为HashMap指定了大小，在我们第一次使用put()方法之前，记录使用的内部数组（耗费4*CAPACITY字节）也不会在内存中分配空间。

### JAVA 8

在Java 8实现中，计算内存使用情况变得复杂一些，因为Node可能会和Entry存储相同的数据，或者在此基础上再增加6个引用和一个Boolean属性（指定是否是TreeNode）。

如果所有的节点都只是Node，那么Java 8 HashMap消耗的内存和Java 7 HashMap消耗的内存是一样的。

如果所有的节点都是TreeNode，那么Java 8 HashMap消耗的内存就变成：

`N * sizeOf(integer) + N * sizeOf(boolean) + sizeOf(reference)* (9*N+CAPACITY )`

在大部分标准JVM中，上述公式的结果是44 * N + 4 * CAPACITY 字节。

## 性能问题

### 非对称HashMap vs 均衡HashMap

在最好的情况下，get()和put()方法都只有O(1)的复杂度。但是，如果你不去关心键的哈希函数，那么你的put()和get()方法可能会执行非常慢。put()和get()方法的高效执行，取决于数据被分配到内部数组（桶）的不同的索引上。如果键的哈希函数设计不合理，你会得到一个非对称的分区（不管内部数据的是多大）。所有的put()和get()方法会使用最大的链表，这样就会执行很慢，因为它需要迭代链表中的全部记录。在最坏的情况下（如果大部分数据都在同一个桶上），那么你的时间复杂度就会变为O(n)。

下面是一个可视化的示例。第一张图描述了一个非对称HashMap，第二张图描述了一个均衡HashMap。

![](/img/2015/09/02/4.jpg) 

在这个非对称HashMap中，在桶0上运行get()和put()方法会很花费时间。获取记录K需要花费6次迭代。

![](/img/2015/09/02/5.jpg) 

在这个均衡HashMap中，获取记录K只需要花费3次迭代。这两个HashMap存储了相同数量的数据，并且内部数组的大小一样。唯一的区别是键的哈希函数，这个函数用来将记录分布到不同的桶上。

下面是一个使用Java编写的极端示例，在这个示例中，我使用哈希函数将所有的数据放到相同的链表（桶），然后我添加了2,000,000条数据。

```java
public class Test {
 
    public static void main(String[] args) {
 
        class MyKey {
            Integer i;
            public MyKey(Integer i){
                this.i =i;
            }
 
            @Override
            public int hashCode() {
                return 1;
            }
 
            @Override
            public boolean equals(Object obj) {
            …
            }
 
        }
        Date begin = new Date();
        Map <MyKey,String> myMap= new HashMap<>(2_500_000,1);
        for (int i=0;i<2_000_000;i++){
            myMap.put( new MyKey(i), "test "+i);
        }
 
        Date end = new Date();
        System.out.println("Duration (ms) "+ (end.getTime()-begin.getTime()));
    }
}
```

我的机器配置是core i5-2500k @ 3.6G，在java 8u40下需要花费超过**45分钟**的时间来运行（我在45分钟后停止了进程）。如果我运行同样的代码， 但是我使用如下的hash函数：

```java
    @Override
    public int hashCode() {
        int key = 2097152-1;
        return key+2097152*i;
	}
```

运行它需要花费**46秒**，和之前比，这种方式好很多了！新的hash函数比旧的hash函数在处理哈希分区时更合理，因此调用put()方法会更快一些。如果你现在运行相同的代码，但是使用下面的hash函数，它提供了更好的哈希分区：

```java
@Override
public int hashCode() {
return i;
}
```

现在只需要花费**2秒**！

我希望你能够意识到哈希函数有多重要。如果在Java 7上面运行同样的测试，第一个和第二个的情况会更糟（因为Java 7中的put()方法复杂度是O(n)，而Java 8中的复杂度是O(log(n))。

在使用HashMap时，你需要针对键找到一种哈希函数，可以**将键扩散到最可能的桶上**。为此，你需要**避免哈希冲突**。String对象是一个非常好的键，因为它有很好的哈希函数。Integer也很好，因为它的哈希值就是它自身的值。

### 调整大小的开销

如果你需要存储大量数据，你应该在创建HashMap时指定一个初始的容量，这个容量应该接近你期望的大小。

如果你不这样做，Map会使用默认的大小，即16，factorLoad的值是0.75。前11次调用put()方法会非常快，但是第12次（16*0.75）调用时会创建一个新的长度为32的内部数组（以及对应的链表/树），第13次到第22次调用put()方法会很快，但是第23次（32*0.75）调用时会重新创建（再一次）一个新的内部数组，数组的长度翻倍。然后内部调整大小的操作会在第48次、96次、192次…..调用put()方法时触发。如果数据量不大，重建内部数组的操作会很快，但是数据量很大时，花费的时间可能会从秒级到分钟级。通过初始化时指定Map期望的大小，你可以**避免调整大小操作带来的消耗**。

但这里也有一个**缺点**：如果你将数组设置的非常大，例如2^28，但你只是用了数组中的2^26个桶，那么你将会浪费大量的内存（在这个示例中大约是2^30字节）。

## 结论

对于简单的用例，你没有必要知道HashMap是如何工作的，因为你不会看到O(1)、O(n)以及O(log(n))之间的区别。但是如果能够理解这一经常使用的数据结构背后的机制，总是有好处的。另外，对于Java开发者职位来说，这是一道典型的面试问题。

对于大数据量的情况，了解HashMap如何工作以及理解键的哈希函数的重要性就变得非常重要。

我希望这篇文章可以帮助你对HashMap的实现有一个深入的理解。

---
原文链接： [coding-geek](http://coding-geek.com/how-does-a-hashmap-work-in-java/) 翻译： [ImportNew.com](http://www.importnew.com/) - [Wing](http://www.importnew.com/author/wing011203)
译文链接： [http://www.importnew.com/16599.html](http://www.importnew.com/16599.html)

