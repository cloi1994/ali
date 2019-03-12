# ConcurrentHashMap 1.7

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。

- Segment是一种可重入锁ReentrantLock
- HashEntry则用于存储键值对数据。

一个ConcurrentHashMap里包含一个Segment数组，Segment的结构和HashMap类似，是一种数组和链表结构， 

一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素， 

每个Segment守护者一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时，必须首先获得它对应的Segment锁。

![s](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/image004.jpg)


### 和Hashmap分別
当ConcurrentMaps使用map.get(key)时返回为null,无法判断key是不存在还是值为空，non-concurrent还可以再调用map.contains(key)检查，但ConcurrentMaps可能再两次调用间已经发生改变。

## HashEntry 类

HashEntry 用来封装散列映射表中的键值对。

key，hash 和 next 域都被声明为 final 型，

value 域被声明为 volatile 型。

```java
static final class HashEntry<K,V> { 
       final K key;                       // 声明 key 为 final 型
       final int hash;                   // 声明 hash 值为 final 型 
       volatile V value;                 // 声明 value 为 volatile 型
       final HashEntry<K,V> next;      // 声明 next 为 final 型 
}
```



在 ConcurrentHashMap 中，在散列时如果产生“碰撞”，将采用“分离链接法”来处理“碰撞”：把“碰撞”的 HashEntry 对象链接成一个链表。由于 HashEntry 的 next 域为 final 型，所以新节点只能在链表的表头处插入。 下图是在一个空桶中依次插入 A，B，C 三个 HashEntry 对象后的结构图：

![a](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/image002.jpg)

注意：由于只能在表头插入，所以链表中节点的顺序和插入的顺序相反。

ConcurrentHashMap 默认创建 16 个 Segment 对象的数组。每个 Segment 的成员对象 table 包含若干个散列表的桶。每个桶是由 HashEntry 链接起来的一个链表。如果键能均匀散列，每个 Segment 大约守护整个散列表中桶总数的 1/16。


## 用分离锁实现多个线程间的并发写操作

在 ConcurrentHashMap 中，线程对映射表做读操作时，*一般情况下*不需要加锁就可以完成，对容器做结构性修改的操作才需要加锁。下面以 put 操作为例说明对 ConcurrentHashMap 做结构性修改的过程。

#### Put

1. Put方法首先定位到Segment，然后在Segment里进行插入操作。

2. 插入操作需要经历两个步骤，

3. if: 判断是否需要对Segment里的HashEntry数组进行扩容

4. 定位添加元素的位置然后放在HashEntry数组里

5. 如果key存在就直接替换这个结点的value。
   否则创建一个新的结点并添加到hash链的头部，这时一定要修改modCount和count的值
   同样修改count的值一定要放在最后一步

#### Get

ConcurrentHashMap的get操作是直接委托给Segment的get方法(不需要锁)

```java
V get(Object key, int hash) {  
     if (count != 0) { // read-volatile 当前桶的数据个数是否为0 
         HashEntry<K,V> e = getFirst(hash);  得到头节点
         while (e != null) {  
             if (e.hash == hash && key.equals(e.key)) {  
                 V v = e.value;  
                 if (v != null)  
                     return v;  
                 return readValueUnderLock(e); // recheck  
             }  
             e = e.next;  
         }  
     }  
     return null;  
 }
```

##### 不需要锁

1. 访问count volatile变量是否为0
2. 根据hash和key对hash链进行遍历找到要获取的结点，如果没有找到，直接返回null。
3. 如果找到了所求的结点，判断它的值如果非空就直接返回，否则在有锁的状态下再读一次。

PS1: 所有的修改操作在进行结构修改时都会在最后一步写count, 通过这种机制保证get操作能够得到几乎最新的结构更新。对于非结构更新，也就是结点值的改变，由于HashEntry的value变量是 volatile的，也能保证读取到最新的值。

PS2: 对hash链进行遍历不需要加锁的原因在于**链指针next是final的**。但是头指针却不是final的，这是通过getFirst(hash)方法返回，也就是存在 table数组中的值。这使得getFirst(hash)可能返回过时的头结点，例如，当执行get方法时，刚执行完getFirst(hash)之后，另一个线程执行了删除操作并更新头结点，这就导致get方法中返回的头结点不是最新的。这是可以允许，通过对count变量的协调机制，get能读取到几乎最新的数据，虽然可能不是最新的。要得到最新的数据，只有采用完全的同步。

PS3: 理论上结点的值不可能为空，这是因为 put的时候就进行了判断，如果为空就要抛NullPointerException。空值的唯一源头就是HashEntry中的默认值，因为 HashEntry中的value不是final的，非同步读取有可能读取到空值。仔细看下put操作的语句：tab[index] = new HashEntry<K,V>(key, hash, first, value)，在这条语句中，HashEntry构造函数中对value的赋值以及对tab[index]的赋值可能被重新排序，这就可能导致结点的值为空。这里当v为空时，可能是一个线程正在改变节点，而之前的get操作都未进行锁定，根据bernstein条件，读后写或写后读都会引起数据的不一致，所以这里要对这个e重新上锁再读一遍，以保证得到的是正确值。

#### Remove

1. 和 get 操作一样，首先根据hash找到具体的链表(如果不存在这个节点就直接返回null)
2. 遍历这个链表找到要删除的节点
3. 把待删除节点之后的所有节点保留在新链表中，把待删除节点之前的每个节点克隆到新链表中。

![a(https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/image007.jpg)

![c](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/image008.jpg)

​	假设写线程执行 remove 操作，要删除链表的 C 节点，另一个读线程同时正在遍历这个链表。

#### Resize

##### 是否需要扩容?

在插入元素前会先判断Segment里的HashEntry数组是否超过容量（threshold），如果超过阀值，数组进行扩容。值得一提的是，Segment的扩容判断比HashMap更恰当，因为HashMap是在插入元素后判断元素是否已经到达容量的，如果到达了就进行扩容，但是很有可能扩容之后没有新元素插入，这时HashMap就进行了一次无效的扩容。

##### 如何扩容?

扩容的时候首先会创建一个两倍于原容量的数组，然后将原数组里的元素进行再hash后插入到新的数组里。为了高效ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容。

#### Size

Segment里的全局变量count是一个volatile变量，

##### 直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢？

不是的，虽然相加时可以获取每个Segment的count的最新值，但是拿到之后count发生了变化，那么统计结果就不准了。所以最安全的做法，是在统计size的时候把所有Segment的put，remove和clean方法全部锁住，但是这种做法显然非常低效。

因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以ConcurrentHashMap的做法是先尝试2次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小。

##### 如何判断在统计的时候容器是否发生了变化呢？

使用modCount(修改次数)变量，在put , remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。
