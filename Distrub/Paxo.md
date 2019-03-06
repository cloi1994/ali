## 什么是paxos协议？

Paxos用于解决分布式系统中一致性问题。分布式一致性算法（Consensus Algorithm）是一个分布式计算领域的基础性问题，其最基本的功能是为了在多个进程之间对某个（某些）值达成一致（强一致）；简单来说就是确定一个值，一旦被写入就不可改变。paxos用来实现多节点写入来完成一件事情，例如mysql主从也是一种方案，但这种方案有个致命的缺陷，如果主库挂了会直接影响业务，导致业务不可写，从而影响整个系统的高可用性。paxos协议是只是一个协议，不是具体的一套解决方案。目的是解决多节点写入问题。



## paxos协议用来解决的问题可以用一句话来简化：

将所有节点都写入同一个值，且被写入后不再更改。

## paxos的几个基本概念

#### 一、两个操作：

Proposal Value：提议的值；
Proposal Number：提议编号，可理解为提议版本号，要求不能冲突；

#### 二、三个角色：

Proposer：提议发起者。Proposer 可以有多个，Proposer 提出议案（value）。所谓 value，可以是任何操作，比如“设置某个变量的值为value”。不同的 Proposer 可以提出不同的 value，例如某个Proposer 提议“将变量 X 设置为 1”，另一个 Proposer 提议“将变量 X 设置为 2”，但对同一轮 Paxos过程，最多只有一个 value 被批准。
Acceptor：提议接受者；Acceptor 有 N 个，Proposer 提出的 value 必须获得超过半数(N/2+1)的 Acceptor批准后才能通过。Acceptor 之间完全对等独立。
Learner：提议学习者。上面提到只要超过半数accpetor通过即可获得通过，那么learner角色的目的就是把通过的确定性取值同步给其他未确定的Acceptor。
️三、协议过程

#### 一句话说明是：

proposer将发起提案（value）给所有accpetor，超过半数accpetor获得批准后，proposer将提案写入accpetor内，最终所有accpetor获得一致性的确定性取值，且后续不允许再修改。
协议分为两大阶段，每个阶段又分为A/B两小步骤：

#### 准备阶段（占坑阶段）

##### 第一阶段A：

Proposer选择一个提议编号n，向所有的Acceptor广播Prepare（n）请求。

##### 第一阶段B：

Acceptor接收到Prepare（n）请求，若提议编号n比之前接收的Prepare请求都要大，则承诺将不会接收提议编号比n小的提议，并且带上之前Accept的提议中编号小于n的最大的提议，否则不予理会。

#### 接受阶段（提交阶段）

##### 第二阶段A：

整个协议最为关键的点：Proposer得到了Acceptor响应
如果未超过半数accpetor响应，直接转为提议失败；
如果超过多数Acceptor的承诺，又分为不同情况：
如果所有Acceptor都未接收过值（都为null），那么向所有的Acceptor发起自己的值和提议编号n，记住，一定是所有Acceptor都没接受过值；
如果有部分Acceptor接收过值，那么从所有接受过的值中选择对应的提议编号最大的作为提议的值，提议编号仍然为n。但此时Proposer就不能提议自己的值，只能信任Acceptor通过的值，维护一但获得确定性取值就不能更改原则；

##### 第二阶段B：

Acceptor接收到提议后，如果该提议版本号不等于自身保存记录的版本号（第一阶段记录的），不接受该请求，相等则写入本地。

#### 整个paxos协议过程看似复杂难懂，但只要把握和理解这两点就基本理解了paxos的精髓：

1. 理解第一阶段accpetor的处理流程：如果本地已经写入了，不再接受和同意后面的所有请求，并返回本地写入的值；如果本地未写入，则本地记录该请求的版本号，并不再接受其他版本号的请求，简单来说只信任最后一次提交的版本号的请求，使其他版本号写入失效；

2. 理解第二阶段proposer的处理流程：未超过半数accpetor响应，提议失败；超过半数的accpetor值都为空才提交自身要写入的值，否则选择非空值里版本号最大的值提交，最大的区别在于是提交的值是自身的还是使用以前提交的。



#### 协议过程举例：

看这个最简单的例子：1个processor，3个Acceptor，无learner。

目标：proposer向3个aceptort 将name变量写为v1。

第一阶段A：proposer发起prepare（name，n1）,n1是递增提议版本号，发送给3个Acceptor，说，我现在要写name这个变量，我的版本号是n1
第一阶段B：Acceptor收到proposer的消息，比对自己内部保存的内容，发现之前name变量（null，null）没有被写入且未收到过提议，都返回给proposer，并在内部记录name这个变量，已经有proposer申请提议了，提议版本号是n1;
第二阶段A：proposer收到3个Acceptor的响应，响应内容都是：name变量现在还没有写入，你可以来写。proposer确认获得超过半数以上Acceptor同意，发起第二阶段写入操作：accept（v1,n1），告诉Acceptor我现在要把name变量协议v1,我的版本号是刚刚获得通过的n1;
第二阶段B：accpetor收到accept（v1,n1），比对自身的版本号是一致的，保存成功，并响应accepted（v1,n1）；
结果阶段：proposer收到3个accepted响应都成功，超过半数响应成功，到此name变量被确定为v1。





##### 情况一：Proposer提议正常，未超过accpetor失败情况

问题：还是上面的例子，如果第二阶段B，只有2个accpetor响应接收提议成功，另外1个没有响应怎么处理呢？

处理：proposer发现只有2个成功，已经超过半数，那么还是认为提议成功，并把消息传递给learner，由learner角色将确定的提议通知给所有accpetor，最终使最后未响应的accpetor也同步更新，通过learner角色使所有Acceptor达到最终一致性。



##### 情况二：Proposer提议正常，但超过accpetor失败情况

问题：假设有2个accpetor失败，又该如何处理呢？

处理：由于未达到超过半数同意条件，proposer要么直接提示失败，要么递增版本号重新发起提议，如果重新发起提议对于第一次写入成功的accpetor不会修改，另外两个accpetor会重新接受提议，达到最终成功。



##### 情况三：proposer1和proposer2串行执行

proposer1和最开始情况一样，把name设置为v1，并接受提议。

proposer1提议结束后，proposer2发起提议流程：

第一阶段A：proposer1发起prepare（name，n2）

第一阶段B：Acceptor收到proposer的消息，发现内部name已经写入确定了，返回（name,v1,n1）

第二阶段A：proposer收到3个Acceptor的响应，发现超过半数都是v1，说明name已经确定为v1，接受这个值，不在发起提议操作。



##### 情况四：proposer1和proposer2交错执行

proposer1提议accpetor1成功，但写入accpetor2和accpetor3时，发现版本号已经小于accpetor内部记录的版本号（保存了proposer2的版本号），直接返回失败。

proposer2写入accpetor2和accpetor3成功，写入accpetor1失败，但最终还是超过半数写入v2成功，name变量最终确定为v2；

proposer1递增版本号再重试发现超过半数为v2，接受name变量为v2，也不再写入v1。name最终确定还是为v2



##### 情况五：proposer1和proposer2第一次都只写成功1个Acceptor怎么办

都只写成功一个，未超过半数，那么Proposer会递增版本号重新发起提议，这里需要分多种情况：

3个Acceptor都响应提议，发现Acceptor1{v1,n1} ,Acceptor2{v2,n2},Acceptor{null,null}，Processor选择最大的{v2,n2}发起第二阶段，成功后name值为v2;
2个Acceptor都响应提议，
如果是Acceptor1{v1,n1} ,Acceptor2{v2,n2}，那么选择最大的{v2,n2}发起第二阶段，成功后name值为v2;
如果是Acceptor1{v1,n1} ,Acceptor3{null,null}，那么选择最大的{v1,n1}发起第二阶段，成功后name值为v1;
如果是Acceptor2{v2,n2} ,Acceptor3{null,null}，那么选择最大的{v2,n2}发起第二阶段，成功后name值为v2;
只有1个Acceptor响应提议，未达到半数，放弃或者递增版本号重新发起提议

可以看到，都未达到半数时，最终值是不确定的！



## **活锁**

当一proposer提交的poposal被拒绝时，可能是因为acceptor promise了更大编号的proposal，因此proposer提高编号继续提交。 如果2个proposer都发现自己的编号过低转而提出更高编号的proposal，会导致死循环，也称为活锁。

 

## **Leader选举**

活锁的问题在理论上的确存在，Lamport给出的解决办法是选举出一个proposer作leader，所有的proposal都通过leader来提交，当Leader宕机时马上再选举其他的Leader。

Leader之所以能解决这个问题，是因为其可以控制提交的进度，比如果之前的proposal没有结果，之后的proposal就等一等，不着急提高编号再次提交，相当于把一个分布式问题转化为一个单点问题，而单点的健壮性是靠选举机制保证。