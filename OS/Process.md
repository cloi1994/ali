#### 进程与线程

进程是一个可运行的程序，程序一旦运行就是进程。

进程是系统资源分配的独立实体，每个进程都拥有独立的地址空间。所以进程无法访问另一个进程的变量和数据结构，如果想进程之间通信，要IPC（管道，信号量，共享内存，消息队列）

线程是进程的一个实体，是进程的一条执行路径(引擎)。一个进程可以拥有多个线程，每个线程共享地址空间但有独立的栈空间。如果多个线程可以读写同一块地址空间，可以用 堆、全局变量、静态变量来进行通信。



#### 切換

##### 安全

进程更加安全，进程间独立，一个进程挂了不会影响到别的进程。进程创建时间比较长，远远长于线程。进程相对于线程最大的区别是有内核保证的隔离：数据和错误隔离。

##### 消耗:

进程上下文切换时间高于线程上下文切换。线程有较轻的上下文切换开销，不用切换地址空间，不用清空TLB。

进程一旦去切换上下文，此时CPU硬件不知道哪些TLB表项是无效的,处理器中所有已经缓存的内存地址一瞬间都作废了(flush)。这将导致内存的访问在一段时间内相当的低效。但是在线程的切换中，不会出现这个问题。



![Image result for process life cycle](https://i.ytimg.com/vi/DacMRZRc4Bo/maxresdefault.jpg)

## 进程通信



### **共享内存**

在通信的进程之间存在一块可直接访问的共享空间，通过对这片共享空间进行写/读操作实现进程之间的信息交换。在对共享空间进行写/读操作时，需要使用同步互斥工具（如 P操作、V操作），对共享空间的写/读进行控制。

共享存储又分为两种：低级方式的共享是基于数据结构的共享；高级方式则是基于存储区的共享。操作系统只负责为通信进程提供可共享使用的存储空间和同步互斥工具，而数据交换则由用户自己安排读/写指令完成。

需要注意的是，**用户进程空间一般都是独立的，要想让两个用户进程共享空间必须通过特殊的系统调用**实现，而进程内的线程是自然共享进程空间的。

### **消息传递**(Message Queue)

在消息传递系统中，进程间的数据交换是以格式化的消息(Message)为单位的。若通信的进程之间不存在可直接访问的共享空间，则必须利用操作系统提供的消息传递方法实现进程通信。进程通过系统提供的发送消息和接收消息两个原语进行数据交换。

1. 直接通信方式：发送进程直接把消息发送给接收进程，并将它挂在接收进程的消息缓冲队列上，接收进程从消息缓冲队列中取得消息。
2. 间接通信方式：发送进程把消息发送到某个中间实体中，接收进程从中间实体中取得消息。这种中间实体一般称为信箱，这种通信方式又称为信箱通信方式。该通信方式广泛应用于计算机网络中，相应的通信系统称为电子邮件系统。

### **管道通信**(父子)

管道通信是消息传递的一种特殊方式。所谓“管道”，是指用于连接一个读进程和一个写进程以实现它们之间通信的一个**共享文件**，又名pipe文件。

向管道（共享文件）提供输入的发送进程（即写进程），以字符流形式将大量的数据送入（写）管道；

而接收管道输出的接收进程（即读进程），则从管道中接收（读）数据。

为了协调双方的通信，管道机制必须提供以下三方面的协调能力：互斥、同步和确定对方的存在。



#### TCB

- 线程ID：为每个新线程分配唯一ID（tid）
 - 堆栈指针：指向进程中线程的堆栈
 - 程序计数器：指向线程的当前程序指令
 - 线程状态（运行，准备，等待，启动，完成）
 - 线程的寄存器值
 - 指向线程所在进程的（PCB）的指针

#### PCB

- 进程状态：可以是new、ready、running、waiting或 blocked等。

- [程序计数器](https://zh.wikipedia.org/wiki/%E7%A8%8B%E5%BC%8F%E8%A8%88%E6%95%B8%E5%99%A8)：接着要运行的指令地址。

- [CPU](https://zh.wikipedia.org/wiki/CPU)[寄存器](https://zh.wikipedia.org/wiki/%E6%9A%AB%E5%AD%98%E5%99%A8)：如[累加器](https://zh.wikipedia.org/wiki/%E7%B4%AF%E5%8A%A0%E5%99%A8)、[变址寄存器](https://zh.wikipedia.org/w/index.php?title=%E7%B4%A2%E5%BC%95%E6%9A%AB%E5%AD%98%E5%99%A8&action=edit&redlink=1)、[堆栈指针](https://zh.wikipedia.org/wiki/%E5%A0%86%E7%96%8A)以及一般用途寄存器、状况代码等，主要用途在于中断时暂时存储数据，以便稍后继续利用；其数量及类别因计算机体系结构有所差异。

- CPU[排班法](https://zh.wikipedia.org/w/index.php?title=%E6%8E%92%E7%8F%AD%E6%B3%95&action=edit&redlink=1)：优先级、排班队列等指针以及其他参数。

- [存储器](https://zh.wikipedia.org/wiki/%E8%A8%98%E6%86%B6%E9%AB%94)管理：如[标签页表](https://zh.wikipedia.org/wiki/%E5%88%86%E9%A0%81%E8%A1%A8)等。

- 会计信息：如CPU与实际时间之使用数量、时限、账号、工作或进程号码。

- 输入输出状态：配置进程使用[I/O](https://zh.wikipedia.org/wiki/I/O)设备，如[磁带机](https://zh.wikipedia.org/wiki/%E7%A3%81%E5%B8%B6)。

  

**直接匹配**

![image-20190127201749247](/Users/ck/Library/Application Support/typora-user-images/image-20190127201749247.png)

##### **全相连 - full associative**

(Tag)(Offset)

- no index field in the address anymore, the entire address must be used as the tag, increasing the total cache size.
-  Data could be anywhere in the cache, so we must check the tag of every cache block. That’s a lot of comparators!

**组相连 - set-associative**

* The cache is divided into groups of blocks, called sets.

* Each memory address maps to exactly one set in the cache, but data may be placed in any block within that set.

![image-20190127202534758](/Users/ck/Library/Application Support/typora-user-images/image-20190127202534758.png)

![image-20190127202546872](/Users/ck/Library/Application Support/typora-user-images/image-20190127202546872.png)

直接映射方式的优点是：硬件简单，成本低。缺点是;每个主存块只有一个固定的行位置可存放

全相联的缺点是：比较器电路难于设计和实现。只适合小容量cache采用。优点是存放位置的灵活性命中率好。缺点: 数据可以位于缓存中的任何位置，因此我们必须检查每个缓存块的标记

组相联映射方式融合了前两者的优点。尽量避免了它们的缺点。普遍采用这种方式