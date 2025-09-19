---
title: 'HashMap 原理'
publishDate: 2025-02-09
updatedDate: 2025-02-24
description: '3D imagery has the power to bring cinematic visions to life and help accurately plan tomorrow’s cityscapes. Here, 3D expert Ricardo Ortiz explains how it works.'
tags:
  - Java
  - 数据结构
language: '简体中文'
heroImage: { src: './thumbnail.jpg', color: '#D58388' }
---

全文如果没有特殊说明，采用的均是**Java 1.8**

## HashMap的简介

HashMap 是 Java 开发中最常用到的数据结构之一，它是一个散列表，它存储的内容是键值对(key-value)映射。HashMap 是无序的，即不会记录插入的顺序。

HashMap 实现了 Map 接口，根据键的 HashCode 值存储数据，具有很快的访问速度，最多允许一条记录的键为 null，在多线程下操作不安全。

HashMap 继承于AbstractMap，实现了 Map、Cloneable、java.io.Serializable 接口。

![HashMap的继承结构图](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/image-1eee71d3773d4761a3f87a17214f1b57.png)



## HashMap的数据结构

HashMap底层采用 **数组+链表/红黑树** 实现的。每个放进去的(key-value) 元素都被包装成了一个 **Node节点**。Node节点有前后指针

插入的节点会优先放在数组对应下标处，如果该下标已经有值就会在该位置生成一个单向链表或红黑树

![HashMap Java](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/HashMapRep-b703dd62ae0c45148718dd98d2b56f6f.png)



## hash值计算原理

Java中的HashMap使用的hash算法如下。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这个算法的原理就是用hascode值的高16位和低16位做了一个异或运算，来混淆hashcode低位的特征

因为hashcode的值是32位的int类型，取值范围位为 **2147483647 ~ -2147483648**，整数部分有21亿多个，所以一般来说只要hash算法比较分散是很难发生hash冲突的。

但我们不能在项目中创建这么大的数组，要知道HashMap的默认数组长度也只有16。所以我们必须对这个hash值进行压缩，让他能放入到一个短一点的数组中。这里可以通过对长度取模求出余数来作为访问数组的下标（例如 hash % 16）

HashMap中是用了一个**与运算**来实现相同的效果，因为二进制运算比算术运算性能更好

```java
(n - 1) & hash  //n是数组的长度
```

我们可以通过一个简单的式子来了解这个运算。假设现在有一个hash值是43，HashMap数组长度为 16

43的二进制是 10101，数组长度减一的二进制码为 1111, 将它们做与运算

![与运算](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6%20(3)-cee3b443aa11452190c8a84560e179fb.png)


可以看到，43的高位全部被清零，只留下了低位的4位作为**余数**，这也是为什么HashMap数组长度是2的幂次方，只有这样才能将长度减一（n-1）作为一个“低位掩码”。

然后我们可以猜测出一个结论：**之所以Hashmap长度要设计成2的幂次方，就是因为可以使用二进制运算提升性能**



正是因为 hashcode 的 hash 值无法全部使用，只能使用低位，位数变少后hash冲突的概率会大大增加。

所以 HashMap 中使用了 hash 值前16位，刚好是32位 int 类型的一半与后16位做了一个异或操作，增加低位的随机性，同时也变相保留了高位的部分特征。这样可以更好的避免hash冲突



## HashMap如何解决hash冲突

先说结论：**HashMap解决hash冲突使用了链表和红黑树的数据结构**



HashMap在初始化的时候只是一个数组结构，后续元素会被插入到数组中。如果发生了hash冲突则会在冲突处下标 Node 节点后接上一个链表（链表满足一定条件会转换成红黑树）。源码如下

```java
for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
    if (e.hash == hash &&
        ((k = e.key) == key || (key != null && key.equals(k))))
        break;
    p = e;
}
```

这段代码主要逻辑如下：

去遍历当前下标处的链表，看有没有key相等的元素，如果有就直接替换，如果没有就添加到链表尾部。

**添加完成后**判断当前链表长度有没有**超过 TREEIFY_THRESHOLD = 8**，超过了会调用**treeifyBin**方法转换成红黑树

在执行移除操作的时候，如果删除的是红黑树的节点，会判断当前节点是否太小（判断是否只有一边有值）。如果太小的话当前的红黑树就成了单向链表了，此时会被退化成链表

```java
if (root == null || root.right == null ||
    (rl = root.left) == null || rl.left == null) {
    tab[index] = first.untreeify(map);  // too small
    return;
}
```



## HashMap扩容原理

当 HashMap 数组被使用的下标位置比例超过一定的阈值后，就会触发扩容机制。这个阈值定义为了一个常量**DEFAULT_LOAD_FACTOR = 0.75F**

例如 HashMap 初始化默认长度为16，在每次插入数据后都会做一个判断。如果已经使用的下标数量**超过12(16 x 0.75)时**就会触发 **resize** 方法扩容。resize 方法代码较长，挑选一些比较重要的片段

```java
if (oldCap > 0) {
    if (oldCap >= MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return oldTab;
    }
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
        //容量扩大一倍
        newThr = oldThr << 1; // double threshold
}
```

这段代码的主要逻辑是，在数组长度大于0的时候（小于0则进行初始化），如果数组长度已经超过了定义的最大长度，则使用 int 的最大值。如果没有超过则将数组容量扩大一倍（但是也不能超过最大长度）。因为数组长度本来就是2的幂次方，所以直接左移一位就代表扩大一倍



因为元素放在哪个数组下标处是通过hash值对数组长度取模计算出来的，所以扩容后我们需要把所有元素都重新迁移。如果是所有的元素都重新计算 hash 值然后移动位置效率是极低的。

在JDk8中，使用了一个巧妙的设计大大提升了迁移的性能：

**一个节点在迁移的时候，要么保持在原位置不动，要么就迁移到 (原位置+原长度) 的地方**，做法原理如下

![未命名文件](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6-d70acf51352449659913fe3793dde10d.png)

扩容前长度为16，进行取余运算后只保留了后四位作为余数。而因为HashMap数组长度是2的幂次方，所以扩容后对长度取模会保留下来五位，而低位的四位代表原来的位置。

所以只需要看余数最高位是0还是1，如果是0，则余数不变所以还是在原来的位置。如果为1，则相当于在原来余数的基础上增加了一个原来数组的长度。**（多出来的高位可以看成原来的四位 + 10000，而原来的数组长度就是10000）**

**因此在扩容的时候就不需要重新计算元素的hash值了，只需要判断一下高位是0还是1就行了。**

核心扩容代码逻辑（resize 方法）

1. 首先是进行了一个判断，如果对应下标的元素next指针为 null 的话证明该节点上没有发生hash冲突，直接重新计算hash值迁移就可以了
2. 如果是红黑树则执行转换流程（这里还有一点疑问，以后补充）
3. 最后如果是链表的话，则使用我们上面说的方法，具体公式为： `(e.hash & oldCap) == 0` 计算新的hash值迁移

```java
for (int j = 0; j < oldCap; ++j) {
    Node<K,V> e;
    if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)
            newTab[e.hash & (newCap - 1)] = e;
        else if (e instanceof TreeNode)
            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
        else { // preserve order
            Node<K,V> loHead = null, loTail = null;
            Node<K,V> hiHead = null, hiTail = null;
            Node<K,V> next;
            do {
                next = e.next;
                if ((e.hash & oldCap) == 0) {
					//省略一些代码
                }
            } while ((e = next) != null);
            if (loTail != null) {
                loTail.next = null;
                //在原来的地方不变
                newTab[j] = loHead;
            }
            if (hiTail != null) {
                hiTail.next = null;
                //位置变为原来的位置+旧数组的长度
                newTab[j + oldCap] = hiHead;
            }
        }
    }
}
```

在扩容后如果红黑树的节点数量小于等于常量  **UNTREEIFY_THRESHOLD = 6** ，则红黑树会退化成链表



## 关于Map线程安全的讨论

首先肯定一点，HashMap是线程不安全的。如果多个线程同时对其进行操作，是无法保证数据准确性的。因为HashMap有很多的成员变量，且插入删除等操作都不是原子性的。

举个例子，并发插入数据的时候如果两个元素落在同一个位置上，但是因为插入操作不是原子性的。所以两个元素可能都会判断为当前下标为 null，然后插入。这样在物理上后插入的值一定会覆盖掉先插入的值，从而造成数据丢失。如下图所示，两个元素并发插入到下标为2的位置，且都判断该下标没有元素。最后B或者A的值就会被覆盖掉

![并发插入数据](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6%20(2)-6547d5c005ad47738c732f58a0677fef.png)

那如果我们要使用线程安全的Map怎么办？


### 使用 HashTable

这是 HashTable 的继承图。从下图可以看到，HashTable 也是实现了 Map 接口的，所以 HashMap 有的操作它也都支持。

![image](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/image-00d5540e6e354c8996118d5166507426.png)



HashTable 之所以是线程安全的原因也很简单粗暴，它的所有方法上都添加了 **synchronized**  关键字来实现线程同步。下面是几个常见的方法

![image](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/image-e284291fd88649a68845a7544066a96b.png)

![image](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/image-b6b9e6ae8494420da86fdf2ef7b1b7f7.png)

![image](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/image-ed6e16ff762a4e66ade4a4ddd3477806.png)

这样做的好处在于简单粗暴的就解决了线程安全问题，但是缺点也很明显，锁的粒度太大导致性能过低。

### ConCurrentHashMap

ConcurrentHashMap 的继承图。从图上看到它也继承了Map。所以各种 api 都是和HashMap相同的

![image](https://lk-blog.oss-cn-shenzhen.aliyuncs.com/2021/11/image-6e326849135e4076b08c90a9a8f7998f.png)

ConcurrentHashMap 相较于 HashTable 性能更高的原因就在于：**前者的锁粒度更小。**

其实在并发编程中，优化性能最关键的原因就是减少锁的竞争，因为锁竞争会引起线程上下文的切换，而线程切换的时候需要保存和恢复上下文信息，是比较耗时的。最好的方法就是无锁编程，但是实现起来很困难。第二种方法就是最大程度减锁的粒度，只在必要的时候进行线程同步

ConcurrentHashMap 就是使用 **自旋 + CAS** 的方法减少了锁的粒度。插入流程如下

1. 计算出 hash 值

   ```java
   int hash = spread(key.hashCode());
   ```

2. 进入自旋并判断是否需要初始化

   ```java
   if (tab == null || (n = tab.length) == 0)
       tab = initTable();
   ```

3. 如果没有发生 hash 冲突，会使用**CAS**的方式将元素尝试插入到指定下标处，如果失败则代表发生 hash 冲突已经有线程先一步插入了，开始下一轮自旋。

   ```java
   else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
       if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
           break;                   // no lock when adding to empty bin
   }
   ```

4. 下一轮自旋的时候由于已经发生了 hash 冲突，所以会走到处理冲突的分支中。

   ```java
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            if (fh >= 0) {
                //...... 省略插入链表尾部代码
            }
            else if (f instanceof TreeBin) {
                //.. 省略插入红黑树代码
            }
        }
    }
   ```

   主要逻辑是用 **synchronized** 锁住当前数组下标处的元素（也就是链表的头节点），然后判断当前节点（`fh >= 0`）如果没有扩容且没有树化，就插入到链表中，反之则插入到红黑树中。

可以看到，ConcurrentHashMap 在插入元素的时候优先选择自旋+CAS不会主动引起上下文切换，发生冲突了也才只锁住冲突下标处的头节点，而其它节点依然可以继续进行操作，锁的粒度比 HashTable 要细很多，这也是其性能更高的原因。但是它的代码也更加复杂。



## 关于 Key 能否为 null 的问题

在 Map 的实现类中，有一个很有意思的现象，**HashMap 是允许插入 value为 null 的**，这个可以自己测试一下。



但是其余的 Map 实现类，例如 HashTable 如果 value 是空会直接报空指针的（其实key如果为null也会报错）

```java
public synchronized V put(K key, V value) {    
    // Make sure the value is not null    
    if (value == null) {        
        throw new NullPointerException();    
    }        
    // Makes sure the key is not already in the hashtable.    
    Entry<?,?> tab[] = table;    
    //如果key为null，这里也会报空指针异常
    int hash = key.hashCode();    
    //省略其余代码...
}
```



ConcurrentHashMap 则是 key 和 value 都不能为 null

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
}
```



这样做的原因是：HashTable 和 ConcurrentHashMap 都是支持并发的，如果支持 value 为 null 的话就可能会有一个问题：使用 get 方法取出一个 null 值后你无法判断该值是原本就为 null ，还是因为没有这个 key 才取出的null。这时候就需要用 **containsKey** 方法去判断是否包含key，伪代码如下

```java
Map<Integer,Integer> map = new ConcurrentHashMap<>();
map.put(2,null);
Integer num = map.get(2);
if (map.containsKey(2)){
    //对num进行一些处理...
}
```

这样这个操作就不是原子操作了，等判断完以后 key=2 这个元素可能已经被删除或者修改了，造成线程不安全。

而 HashMap 本身就是在单线程环境下用的，所以多加个判断也无所谓

这里又可以得出一个结论：**HashTable 和 ConCurreentHashMap 的 value 不能为 null 是为了避免取值为 null 的歧义，而 HashMap 不存在并发问题，所以可以存 null**

## 总结

1. HashMap 的数组长度之所以要设计成2的幂次方是因为方便进行一系列巧妙的二进制操作提升性能
2. HashMap 解决 hash 冲突的原因是使用**链表/红黑树**数据结构，在扩容或者移除元素的时候，红黑树也有可能退出成链表
3. HashMap 是线程不安全的，并发环境下推荐使用 ConcurrentHashMap，相比较与 HashTable 前者锁的粒度更低，性能更好
4. 线程安全的 Map 其 value 一般都不能为 null，这是为了避免产生歧义