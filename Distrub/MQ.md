# RabbitMQ基础知识

### 背景

在项目中，将一些无需即时返回且耗时的操作提取出来，进行了[异步处理](https://baike.baidu.com/item/%E5%BC%82%E6%AD%A5%E5%A4%84%E7%90%86)，而这种异步处理的方式大大的节省了服务器的[请求响应时间](https://baike.baidu.com/item/%E8%AF%B7%E6%B1%82%E5%93%8D%E5%BA%94%E6%97%B6%E9%97%B4)，从而提高了系统的吞吐量。

削峰，并发量大的时候，所有的请求直接怼到数据库，造成数据库连接异常

### 基础概念

![image-20190129155735537](/Users/ck/Library/Application Support/typora-user-images/image-20190129155735537.png)

#### 　**Queue**

![image-20190129155800488](/Users/ck/Library/Application Support/typora-user-images/image-20190129155800488.png)



  多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。

但是会存在一个一个问题就是如果每个消息的处理时间不同，就会导致某些消费者一直在忙碌中，而有的消费者处理完了消息后一直处于空闲状态，因为前面已经提及到了Queue会平分这些消息给相应的消费者。这里我们就可以使用prefetchCount来限制每次发送给消费者消息的个数。

 prefetchCount=1是指每次从Queue中发送一条消息来。等消费者处理完这条消息后Queue会再发送一条消息给消费者。

####     **Exchange**

  在绑定（Binding）Exchange和Queue的同时，一般会指定一个Binding Key，生产者将消息发送给Exchange的时候，一般会产生一个Routing Key，当Routing Key和Binding Key对应上的时候，消息就会发送到对应的Queue中去。 

- ##### **topic**

​     模糊匹配，可以通过通配符满足一部分规则就可以传送。它的约定是：

1. routing key为一个句点号“. ”分隔的字符串（我们将被句点号“. ”分隔开的每一段独立的字符串称为一个单词），如“stock.usd.nyse”、“nyse.vmw”、“quick.orange.rabbit”
2. binding key与routing key一样也是句点号“. ”分隔的字符串
3. binding key中可以存在两种特殊字符“*”与“#”，用于做模糊匹配，其中“*”用于匹配一个单词，“#”用于匹配多个单词（可以是零个）

| 类型名称 | 类型描述                                                     |
| -------- | ------------------------------------------------------------ |
| fanout   | 把所有发送到该Exchange的消息路由到所有与它绑定的Queue中      |
| direct   | Routing Key==Binding Key                                     |
| topic    | 模糊匹配                                                     |
| headers  | Exchange不依赖于routing key与binding key的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配。 |



#### 问题

1. 如果自动创建队列，那么是谁负责创建队列呢？是生产者？还是消费者？ 

​       如果队列不存在，当然消费者不会收到任何的消息。但是如果队列不存在，那么生产者发送的消息就会丢失。

​       ＝》消费者和生产者都可以创建队列

2. 那么如果创建一个已经存在的队列呢？

   没有任何的影响 ，也就是说第二次创建如果参数和第一次不一样，那么该操作虽然成功，但是队列属性并不会改变。

   队列对于负载均衡的处理是完美的。对于多个消费者来说，RabbitMQ使用轮询的方式均衡的发送给不同的消费者。

#### RabbitMQ的消息确认机制

​      默认情况下，如果消息已经被某个消费者正确的接收到了，那么该消息就会被从队列中移除。当然也可以让同一个消息发送到很多的消费者。

​      如果一个队列没有消费者，那么，如果这个队列有数据到达，那么这个数据会**被缓存**，不会被丢弃。当有消费者时，这个数据会被立即发送到这个消费者，这个数据被消费者正确收到时，这个数据就被从队列中删除。

​     那么什么是正确收到呢？通过ack。**每个消息都要被acknowledged**（确认，ack）。我们可以显示的在程序中去ack，也可以自动的ack。

如果有数据没有被ack：

​     RabbitMQ Server会把这个信息发送到下一个消费者。

​     如果这个app有bug，忘记了ack，那么RabbitMQServer不会再发送数据给它，因为Server认为**这个消费者处理能力有限。**

​    而且ack的机制可以起到限流的作用（Benefitto throttling）：在消费者处理完成数据后发送ack，甚至在额外的延时后发送ack，将有效的均衡消费者的负载。

### ConnectionFactory、Connection、Channel

Connection是RabbitMQ的socket链接，它封装了socket协议相关部分逻辑。ConnectionFactory为Connection的制造工厂。

Channel是我们与RabbitMQ打交道的最重要的一个接口，我们大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等。

Connection就是建立一个TCP连接，生产者和消费者的都是通过TCP的连接到RabbitMQ Server中的，这个后续会再程序中体现出来。

Channel虚拟连接，建立在上面TCP连接的基础上，数据流动都是通过Channel来进行的。为什么不是直接建立在TCP的基础上进行数据流动呢？如果建立在TCP的基础上进行数据流动，建立和关闭TCP连接有代价。频繁的建立关闭TCP连接对于系统的性能有很大的影响，而且TCP的连接数也有限制，这也限制了系统处理高并发的能力。但是，在TCP连接中建立Channel是没有上述代价的。



### 消息持久化

RabbitMQ消息持久化，即数据写到磁盘上。需要exchange，queue，消息（投递时指定delivery_mode）都持久化。如果exchange和queue都是持久化的，那么它们之间的binding也是持久化的。如果exchange和queue两者之间有一个持久化，一个非持久化，就不允许建立绑定。

注： 将消息标记为持久化并不能完全保证消息不会丢失。

1. RabbitMQ接受消息但并未保存时，仍有一个短暂时间
2. RabbitMQ不会为每条消息执行fsync(2)——它可能只是保存到缓存而不是真正写入了磁盘。如果需要更高级别，需要使用发布者确认机制（publisher confirms）。

### 丢数据

##### 1.生产者丢数据

生产者的消息没有投递到MQ中怎么办？从生产者弄丢数据这个角度来看，RabbitMQ提供transaction和confirm模式来确保生产者不丢消息。

**transaction**

发送消息前，开启事物(channel.txSelect())，然后发送消息，如果发送过程中出现什么异常，事物就会回滚(channel.txRollback())，如果发送成功则提交事物(channel.txCommit())。然而缺点就是吞吐量下降了。因此，按照博主的经验，生产上用**confirm**模式的居多。

**confirm**

一旦channel进入confirm模式，所有在该信道上面发布的消息都将会被指派一个唯一的ID(从1开始)，一旦消息被投递到所有匹配的队列之后，rabbitMQ就会发送一个Ack给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了.如果rabiitMQ没能处理该消息，则会发送一个Nack消息给你，你可以进行重试操作。

##### 2.消息队列丢数据

处理消息队列丢数据的情况，一般是开启**持久化磁盘的配置**。这个持久化配置可以和confirm机制配合使用，你可以在消息持久化磁盘后，再给生产者发送一个Ack信号。这样，如果消息持久化磁盘之前，rabbitMQ阵亡了，那么生产者收不到Ack信号，生产者会自动重发。

那么如何持久化呢，这里顺便说一下吧，其实也很容易，就下面两步

①、将queue的持久化标识durable设置为true,则代表是一个持久的队列

②、发送消息的时候将deliveryMode=2

这样设置以后，rabbitMQ就算挂了，**重启后也能恢复数据**。在消息还没有持久化到硬盘时，可能服务已经死掉，这种情况可以通过引入mirrored-queue即镜像队列，但也不能保证消息百分百不丢失（整个集群都挂掉）

##### 3.消费者丢数据

1. 消息处理正常，没有抛出异常，这时我们需要手动应答消息

```
 channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
```

2. 当出现异常时，我们需要把这个消息回滚到消息队列，有两种方式

```
//ack返回false，并重新回到队列，api里面解释得很清楚 (Group)
channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, true);
//拒绝消息 individual 
channel.basicReject(message.getMessageProperties().getDeliveryTag(), true);
```

启用手动确认模式可以解决这个问题

①自动确认模式，消费者挂掉，待ack的消息回归到队列中。**消费者抛出异常，消息会不断的被重发**，直到处理成功。不会丢失消息，即便服务挂掉，没有处理完成的消息会重回队列，但是异常会让消息不断重试。

②手动确认模式，如果消费者来不及处理就死掉时，没有响应ack时会**重复发送一条信息给其他消费者**；

消费者会向rabbitmq server发送Basic.Reject，表示消息拒绝接受，由于Spring默认requeue-rejected配置为true，消息会重新入队，然后rabbitmq server重新投递，造成了程序一直异常的情况

如果监听程序处理异常了，且未对异常进行捕获，会一直重复接收消息，然后一直抛异常；如果对异常进行了捕获，但是没有在finally里ack，也会一直重复发送消息(重试机制)。

③不确认模式，acknowledge="none" 不使用确认机制，只要消息发送完成会立即在队列移除，无论客户端异常还是断开，只要发送完就移除，不会重发。

### 如何保证消息的顺序性？

针对这个问题，通过某种算法，将需要保持先后顺序的消息放到同一个消息队列中。然后只用一个消费者去消费该队列。同一个queue里的消息一定是顺序消息的。

入队有序就行，出队以后的顺序交给消费者自己去保证，没有固定套路。

例如B消息的业务应该保证在A消息后业务后执行，那么我们保证A消息先进queueA，B消息后进queueB就可以了。

### 重复消费

1insert操作。那就容易了，给这个消息做一个唯一主键，那么就算出现重复消费的情况，就会导致主键冲突，避免数据库出现脏数据。

2.redis的set的操作，那就容易了，不用解决，因为你无论set几次结果都是一样的，set操作本来就算幂等操作。

3.如果上面两种情况还不行，准备一个第三方介质,来做消费记录。以redis为例，给消息分配一个全局id，只要消费过该消息，将<id,message>以K-V形式写入redis。那消费者开始消费前，先去redis中查询有没消费记录即可

