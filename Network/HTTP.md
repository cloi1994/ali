在详细探究HTTP与HTTPS之前，先理清一下HTTP的基本概念：

> HTTP是客户端浏览器或其他程序与Web服务器之间的应用层通信协议。在Internet上的Web服务器上存放的都是超文本信息，客户机需要通过HTTP协议传输所要访问的超文本信息。

> HTTPS（全称：Hyper Text Transfer Protocol over Secure Socket Layer），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。

在这里，我们需要先提出几个问题：
 *为什么需要使用HTTPS来进行通信？HTTPS在安全上做了哪些事情？*

### HTTP的缺点

尽管HTTP已经具有相当优秀的方面，但是其依然具有不足之处。

- ##### 通信内容为明文，即未加密，内容可能会被窃听。

**窃听可能发生在互联网通信中的各个环节。**

![image-20190225220110242](/Users/ck/Library/Application Support/typora-user-images/image-20190225220110242.png)

因此，需要对通信进行加密以防止窃听。
 一种加密方式是将通信加密。HTTP没有加密机制，需要配合SSL（安全套接层）或TLS（安全传输层协议）来对通信进行加密。配合SSL使用得HTTP则称为HTTPS。另一种是通过对通信报文的具体内容进行加密。
 虽然我们可以对通信的内容进行加密，但其仅仅是达到让攻击者难以破解报文的目的，但是加密后的报文本身还是能够被截获。目前获取报文信息的软件也有很多，如Sniffer和Wireshark等。

- ##### 通信双方的身份没有进行验证，可能出现伪装身份的情况。

**所有人都可以对服务器发起请求**

![image-20190225220123458](/Users/ck/Library/Application Support/typora-user-images/image-20190225220123458.png)

可以看出，对于客户端来说，无法确定这台Web服务器是否是“真的”服务器，可能通过了伪装。对于服务器来说，也无法确定自己返回的报文是否被真正的客户端接收到。
 此外，服务器的全盘接收的缺点也会被利用来进行DOS攻击。
 因此，以客户端为例，客户端在与服务器通信之前需要确定服务器的身份，该身份即是一份证书，该证书有值得信赖的第三方颁发，客户端确认身份后才进行通信。

![image-20190225220134809](/Users/ck/Library/Application Support/typora-user-images/image-20190225220134809.png)

- ##### 接受的报文完整性无法确定，可能中途被改动。

我们知道，服务器接收到请求后，会进行响应。但服务器和客户端都无法知道报文中途的传输是否出现了问题。很有可能在传输时被其他攻击者进行了篡改，报文完整性遭到破坏。



![image-20190225220159616](/Users/ck/Library/Application Support/typora-user-images/image-20190225220159616.png)

虽然HTTP提供了确认报文完整性的方法（MD5,SHA-1），但是也无法完全保证报文的完整性。因为MD
 5本身也可能被攻击者改写。
 在SSL中，提供了认证和加密处理等功能。通过配合SSL可以达到保证报文完整性的目的。

### 关于HTTPS

鉴于HTTP的缺点，HTTPS在HTTP的基础上增加了：

- 通信加密
- 证书认证
- 完整性保护

在访问使用HTTPS通信的Web网站时，就可以看到一些不同之处：

![image-20190225220244946](/Users/ck/Library/Application Support/typora-user-images/image-20190225220244946.png)



那么，SSL是如何配合HTTP来达到安全通信的？
 首先，需要理清的是HTTPS并非是一个新的协议。HTTP的通信接口部分采用了SSL协议来实现。

![image-20190225220236978](/Users/ck/Library/Application Support/typora-user-images/image-20190225220236978.png)

可以看出，SSL是独立于HTTP的协议，它同样也可用于其他协议的加密，如SMTP等。

#### 普遍的加密方式

-  **共享密钥加密（对称密钥）**
   顾名思义，就是客户端和服务器都拥有一把相同的钥匙，对报文的加密和解密用的都是这把钥匙，而且密钥也需要在通信的过程中发给对方，对方才能通过这把钥匙来解密。因此，一旦在通信过程中，这把密钥被攻击者获取，报文加密便失去了意义。

![image-20190225220254332](/Users/ck/Library/Application Support/typora-user-images/image-20190225220254332.png)



图片来源网络

-  **公开密钥加密**
   共享密钥带来了一个问题就是，如何能够安全地把密钥发送给对方。而公开密钥则较好地解决了这个问题。
   公开密钥加密使用得是非对称的密钥。一把是公有密钥，一把是私有密钥。公有密钥是对通信双方公开的，任何人都可以获取，而私有的则不公开。发送方使用这把公有密钥对报文进行加密，接收方则使用私有的密钥进行解密。仅仅通过密文和公有密钥是很难破解到密文。
   使用公开密钥带来安全的同时，也隐藏着一些问题，就是如何保证公开的这把公有密钥的真实性？这个问题伴随也是证书机构。通过证书来保证公有密钥的真实性。



![image-20190225220307943](/Users/ck/Library/Application Support/typora-user-images/image-20190225220307943.png)

图片来源网络

#### HTTPS采用混合加密机制

由于公有密钥的机制相对复杂，导致其处理速度相对较慢。于是HTTPS利用了两者的优势，采用了混合加密的机制。我们知道，共享（对称）密钥未能解决的问题是**如何能够安全地把密钥发送给对方**。只要解决了这个问题就可以进行安全地通信。于是，HTTPS首先是通过**公有密钥**来对**共享密钥**进行加密传输。当**共享密钥安全地传输给对方**后，双方则使用共享密钥的方式来加密报文，以此来提高传输的效率。

### HTTPS的握手机制

![image-20190225220322353](/Users/ck/Library/Application Support/typora-user-images/image-20190225220322353.png)



图片来源网络

 步骤1：向服务器发起请求。

 步骤2-3：取出公有密钥及证书并发送给客户端。

 步骤4：客户端判断公有密钥是否有效，无效则显示警告。有效则生成一个随机数串，并以此生成客户端的共享密钥。

 步骤5：用步骤3得到的公有密钥对该随机数串加密，发送到服务器。

 步骤6：服务器得到加密报文，用私有密钥解密报文，得到随机数串，并以此生成服务器端的共享密钥

。此时客户端和服务端拥有相同的共享密钥，可以用该共享密钥进行安全通信。

 步骤7-8：服务器对响应进行加密，客户端对报文进行解密。



### 选择HTTP还是HTTPS来搭建服务器

在比较之前，首先要了解HTTPS存在的问题才能进行权衡。

#### SSL会使通信的效率降低

- 通信速率降低
   HTTPS 除了TCP连接，发送请求，响应之外，还需要进行SSL通信。整体通信信息量增加。
- 加密过程消耗资源
   每个报文都需要进行加密和解密的运算处理。比起HTTP会消耗更多的服务器资源。
- 证书开销
   如果想要通过HTTPS进行通信，就必须向认证机构购买证书。

基于以上三点，如果通信中传输的是非敏感的信息，则会较多地选择HTTP协议。而当通信过程中会涉及个人隐私或其他安全信息时，则会选择用HTTPS。当然，访问量也是考虑的因素之一，如果访问量很大，而每个报文都进行加密解密，也会给服务器带来很大的负担。



### 在浏览器中输入url地址 ->> 显示主页的过程

![image-20190226133212289](/Users/ck/Library/Application Support/typora-user-images/image-20190226133212289.png)