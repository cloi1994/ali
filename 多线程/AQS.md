# AQS

**挂起等待线程**

![img](http://static.oschina.net/uploads/space/2016/0403/111901_ASqi_1759553.png)

上面只是addWaiter的实现部分，那么节点入队列之后会继续发生什么呢？那就要看看acquireQueued是怎么实现的了，Thread2和Thread3已经被放入队列了，进入acquireQueued之后：

1. 对于Thread2来说，它的prev指向HEAD，因此会首先再尝试获取锁一次，如果失败，则会将HEAD的waitStatus值为SIGNAL，下次循环的时候再去尝试获取锁，如果还是失败，且这个时候prev节点的waitStatus已经是SIGNAL，则这个时候线程会被通过LockSupport挂起。
2. 对于Thread3来说，它的prev指向Thread2，因此直接看看Thread2对应的节点的waitStatus是否为SIGNAL，如果不是则将它设置为SIGNAL，再给自己一次去看看自己有没有资格获取锁，如果Thread2还是挡在前面，且它的waitStatus是SIGNAL，则将自己挂起。

如果Thread1死死的握住锁不放，那么Thread2和Thread3现在的状态就是挂起状态啦，而且HEAD，以及Thread的waitStatus都是SIGNAL，尽管他们在整个过程中曾经数次去尝试获取锁，但是都失败了，失败了不能死循环呀，所以就被挂起了。当前状态如下：

**锁释放-等待线程唤起**

首先，Thread1会修改AQS的state状态，加入之前是1，则变为0，注意这个时候对于非公平锁来说是个很好的插入机会，举个例子，如果锁是公平锁，这个时候来了Thread4，那么这个锁将会被Thread4抢去。。。

我们继续走常规路线来分析，当Thread1修改完状态了，判断队列是否为null，以及队头的waitStatus是否为0，如果waitStatus为0，说明队列无等待线程，按照我们的例子来说，队头的waitStatus为SIGNAL=-1，因此这个时候要通知队列的等待线程，可以来拿锁啦，这也是unparkSuccessor做的事情，unparkSuccessor主要做三件事情：

1. 将队头的waitStatus设置为0.
2. 通过从队列尾部向队列头部移动，找到最后一个waitStatus<=0的那个节点，也就是离队头最近的没有被cancelled的那个节点，队头这个时候指向这个节点。
3. 将这个节点唤醒，其实这个时候Thread1已经出队列了。

还记得线程在哪里挂起的么，上面说过了，在acquireQueued里面，这里我们也大概能理解AQS的这个队列为什么叫FIFO队列了，因此每次唤醒仅仅唤醒队头等待线程，让队头等待线程先出。