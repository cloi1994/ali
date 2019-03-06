### 原理

Java语言提供了一种稍弱的同步机制，即volatile变量，用来确保将变量的更新操作通知到其他线程。当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。

　　在访问volatile变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此volatile变量是一种比sychronized关键字更轻量级的同步机制。

![img](https://images2015.cnblogs.com/blog/731716/201607/731716-20160708224602686-2141387366.png)

　　当对非 volatile 变量进行读写的时候，每个线程先从内存拷贝变量到CPU缓存中。如果计算机有多个CPU，每个线程可能在不同的CPU上被处理，这意味着每个线程可以拷贝到不同的 CPU cache 中。

　　而声明变量是 volatile 的，JVM 保证了每次读变量都从内存中读，跳过 CPU cache 这一步。

### 特性

　volatile具备两种特性，第一就是保证共享变量对所有线程的可见性。将一个共享变量声明为volatile后，会有以下效应：

　　　　1.当写一个volatile变量时，JVM会把该线程对应的本地内存中的变量强制刷新到主内存中去；

　　　　2.这个写会操作会导致其他线程中的缓存无效。

### 重排序

volatile禁止指令重排序也有一些规则，简单列举一下：

　　1.当第二个操作是volatile写时，无论第一个操作是什么，都不能进行重排序

　　2.当地一个操作是volatile读时，不管第二个操作是什么，都不能进行重排序

　　3.当第一个操作是volatile写时，第二个操作是volatile读时，不能进行重排序**

### 原子性

java的内存模型保证了`valotile`修饰的`long`、`double`变量的读写是原子性的。

例如你让一个volatile的integer自增（i++），其实要分成3步：1）读取volatile变量值到local； 2）增加变量的值；3）把local的值写回，让其它的线程可见。这3步的jvm指令为：



```java
mov    0xc(%r10),%r8d ; Load
inc    %r8d           ; Increment
mov    %r8d,0xc(%r10) ; Store
lock addl $0x0,(%rsp) ; StoreLoad Barrier
```

从Load到store到内存屏障，一共4步，其中最后一步jvm让这个最新的变量的值在所有线程可见，也就是最后一步让所有的CPU内核都获得了最新的值，但**中间的几步（从Load到Store）**是不安全的，中间如果其他的CPU修改了值将会丢失。



### CAS算法

CAS：Compare and Swap，即比较再交换。是无锁操作，也有人说他是乐观锁，也许是相对于独占锁来说，原子性操作（通过操作系统指令来实现）。

重入锁：java.util.concurrent.ReentrantLock和AutomicInteger就是使用CAS算法实现的。它是通过算法的

来实现数据操作的互斥性。

CAS有3个操作数：内存值V（可能被thread修改）、旧的预期值A（thread修改后打算回写时看的主存变量值）、要修改的新值B（当前thread操作后的值）。

当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

该操作是一个原子操作，被广泛的应用在Java的底层实现中。在Java中，CAS主要是由sun.misc.Unsafe这个类通过JNI调用CPU底层指令实现。

![img](https://img-blog.csdn.net/20170805090100139?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2hlbmdxaWFuZmVuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**ps：失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次发起尝试。重复CAS算法**



#### ABA Problem

thread1在进行cas前的这段时间内，也许会发生一些事情：thread2将进行了两次CAS操作，先+1，再-1.

thread1在cas时发现内存值还是1，于是进行+1操作，内存值变为2，但是其实thread1后来读取到的1的现场已经发生了变化。

#### CAS开销

CAS特点：速度快。

原因：

1. CAS（比较并交换）是CPU指令级的操作，只有一步原子操作，所以非常快。
2. 且CAS避免了请求操作系统来裁定锁的问题，不用麻烦操作系统，直接在CPU内部就搞定了


CAS的开销在于Cache miss：也就是对某一个变量执行 CAS 操作的 CPU 并不是最后一个操作该变量的那个CPU，而是系统中的其他CPU（假如8核CPU），所以对应的缓存线并不存在于当前对变量执行CAS操作的那个 CPU 的高速缓存中。所以需要将当前变量读取到当前操作CPU的寄存器（高速缓存中）。