## RPC 是什么？

RPC 的全称是 Remote Procedure Call 是一种进程间通信方式。它允许程序调用另一个地址空间（通常是共享网络的另一台机器上）的过程或函数，而不用程序员显式编码这个远程调用的细节。即程序员无论是调用本地的还是远程的，本质上编写的调用代码基本相同。

## RPC 起源

简单：RPC 概念的语义十分清晰和简单，这样建立分布式计算就更容易。
高效：过程调用看起来十分简单而且高效。
通用：在单机计算中过程往往是不同算法部分间最重要的通信机制。 

通俗一点说，就是一般程序员对于本地的过程调用很熟悉，那么我们把 RPC 作成和本地调用完全类似，那么就更容易被接受，使用起来毫无障碍。Nelson 的论文发表于 30 年前，其观点今天看来确实高瞻远瞩，今天我们使用的 RPC 框架基本就是按这个目标来实现的。

## RPC 结构

Nelson 的论文中指出实现 RPC 的程序包括 5 个部分：

这 5 个部分的关系如下图所示
![img](https://img-blog.csdn.net/20150108170924203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWluZGZsb2F0aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
这里 user 就是 client 端，当 user 想发起一个远程调用时，它实际是通过本地调用 user-stub。user-stub 负责将调用的接口、方法和参数通过约定的协议规范进行编码并通过本地的 RPCRuntime 实例传输到远端的实例。远端 RPCRuntime 实例收到请求后交给 server-stub 进行解码后发起本地端调用，调用结果再返回给 user 端。