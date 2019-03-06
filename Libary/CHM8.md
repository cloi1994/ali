# ConcurrentHashMap 1.8



#### JDK1.8分析

1.8的实现已经抛弃了Segment分段锁机制，利用CAS+Synchronized来保证并发更新的安全，底层采用数组+链表+红黑树的存储结构。



```java
class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    ... 省略部分代码
}
```

#### 重要概念

在开始之前，有些重要的概念需要介绍一下：

1.  **table**：默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。
2.  **nextTable**：默认为null，扩容时新生成的数组，其大小为原数组的两倍。

-  **sizeCtl** ：默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。
- **-1 **代表table正在初始化
- **-N **表示有N-1个线程正在进行扩容操作
- 其余情况：
   1、如果table未初始化，表示table需要初始化的大小。
   2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍，居然用这个公式算0.75（n - (n >>> 2)）。
-  **Node**：保存key，value及key的hash值的数据结构。



#### 实例初始化

实例化ConcurrentHashMap时带参数时，会根据参数调整table的大小，假设参数为100，最终会调整成256，确保table的大小总是2的幂次方

**注意**，ConcurrentHashMap在构造函数中只会初始化sizeCtl值，并不会直接初始化table，而是延缓到第一次put操作。

#### Put

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        ...省略部分代码
    }
    addCount(1L, binCount);
    return null;
}
```

假设table已经初始化完成，put操作采用CAS+synchronized实现并发插入或更新操作，具体实现如下。

1. 判断存储的key、value是否为空，若为空，则抛出异常，否则，进入步骤②

2. 计算key的hash值，随后进入无限循环，该无限循环可以确保成功插入数据，若table表为空或者长度为0，则初始化table表，否则，进入步骤③

3. 根据key的hash值取出table表中的结点元素，若取出的结点为空（该桶为空），则使用CAS将key、value、hash值生成的结点放入桶中。否则，进入步骤④

4.  若该结点的的hash值为MOVED，则对该桶中的结点进行转移，否则，进入步骤⑤

5. 对桶中的第一个结点（即table表中的结点）进行加锁

   - 如果f.hash >= 0，说明f是链表结构的头结点，遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点。

   - 如果f是TreeBin类型节点，说明f是红黑树根节点，则在树结构上遍历元素，更新或增加节点。

   - 如果链表中节点数binCount >= TREEIFY_THRESHOLD(默认是8)，则把链表转化为红黑树结构

6. 增加binCount的值



#### Get

1. 判断table是否为空，如果为空，直接返回null。
2. 计算key的hash值，并获取指定table中指定位置的Node节点，通过遍历链表或则树结构找到对应的节点，返回value值。

#### Resize

当table容量不足的时候，即table的元素数量达到容量阈值sizeCtl，需要对table进行扩容。
整个扩容分为两部分：

* 构建一个nextTable，大小为table的两倍。

* 原table中的节点拷贝过来。对于容量为n的原table中Hash桶i中的数据，一部分存储到新table的相同位置i，另一部分存储到i+n的hash桶中。这样就完成了扩容操作，减少了每个Hash桶中的数据，提高了Map的性能

  ![image-20190123220859061](/Users/ck/Library/Application Support/typora-user-images/image-20190123220859061.png)

1. 首先根据运算得到需要遍历的次数i，然后利用tabAt方法获得i位置的元素f，初始化一个forwardNode实例fwd。

2. 如果f == null，则在table中的i位置放入fwd，这个过程是采用Unsafe.compareAndSwapObjectf方法实现的，很巧妙的实现了节点的并发移动。

3. 如果f是链表的头节点，就构造一个反序链表，把他们分别放在nextTable的i和i+n的位置上，移动完成，采用Unsafe.putObjectVolatile方法给table原位置赋值fwd。

4. 如果f是TreeBin节点，也做一个反序处理，并判断是否需要untreeify，把处理的结果分别放在nextTable的i和i+n的位置上，移动完成，同样采用Unsafe.putObjectVolatile方法给table原位置赋值fwd。

##### 多线程

transfer是可以多线程进行的。通过给每个线程分配迁移(将table中的hash桶迁移到新的nextTab中)的hash桶，每个线程最少迁移16个hash桶，用transferIndex来同步。一个map扩容，要迁移的hash桶是原来的table长度。分配时从后面的分配起。从下图看出，每个线程分配的hash桶脚标范围是transferIndex-stride（线程迁移的hash桶个数）到transferIndex-1。分配完一个线程后，通过cas将transferIndex设置为transferIndex-stride

![img](https://ss2.baidu.com/6ONYsjip0QIZ8tyhnq/it/u=3979561241,1885406207&fm=173&app=25&f=JPEG?w=640&h=860&s=7AAC346211DF65C81C7DD4CF0000A0B1)

#### Size

JDK1.8中使用一个volatile类型的变量baseCount记录元素的个数，当插入[新数据](https://www.baidu.com/s?wd=%E6%96%B0%E6%95%B0%E6%8D%AE&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)put()或则删除数据remove()时，会通过addCount()方法更新baseCount:

所以counterCells存储的都是value为1的CounterCell对象，而这些对象是因为在CAS更新baseCounter值时，由于高并发而导致失败，最终将值保存到CounterCell中，放到counterCells里。这也就是为什么sumCount()中需要遍历counterCells数组，sum累加CounterCell.value值了。

JDK 8 推荐使用mappingCount 方法，因为这个方法的返回值是 long 类型，不会因为 size 方法是 int 类型限制最大值。

