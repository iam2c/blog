# 0x01 原因
最近听到一个说法：
```
HTTPS的请求在使用代理时，会被截获拿到明文（HTTP请求消息和响应消息）。
```
于是我们讨论了一下，我的观点是：
```
在客户端没有安装相应证书或者忽略证书错误提示的情况下，代理时没法拿到明文的。
```
那到底是怎么样的呢？下面我们慢慢来说。

# 0x02 读前
```
由于个人能力有限，错漏在所难免，欢迎批评指正。
```

本文适合读者：

1. 对HTTPS感兴趣。
2. 对知识有很强的求知欲。
3. 对[RSA](https://baike.baidu.com/item/RSA算法/263310)、[散列算法](https://baike.baidu.com/item/Hash/390310)和[TCP](https://baike.baidu.com/item/TCP/33012)以及[HTTP](https://baike.baidu.com/item/http)有一定了解的.

阅读本文，你可能会学到：
1. HTTPS、TLS协议，这也是本文的重点。
2. 证书及证书的认证。
3. 代理上网相关知识。
4. wireshark分析TCP等协议

阅读本文，你可能不会学到：
1. 证书的生成。
2. RSA、散列算法、TCP和HTTP的具体细节。
3. 如何抓包、如何搭建本模拟环境。

# 0x03 HTTPS和TLS

下面这两段来自[百度百科·HTTPS](https://baike.baidu.com/item/https/285356)：

> HTTPS（全称：Hyper Text Transfer Protocol over Secure Socket Layer），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。 

> 最初，HTTPS是与SSL一起使用的；在SSL逐渐演变到TLS时，最新的HTTPS也由在2000年五月公布的RFC 2818正式确定下来。

下面这段也是来自[百度百科·SSL](https://baike.baidu.com/item/ssl)：
> SSL(Secure Sockets Layer 安全套接层),及其继任者传输层安全（Transport Layer Security，TLS）是为网络通信提供安全及数据完整性的一种安全协议。TLS与SSL在传输层对网络连接进行加密。

下面的来自[维基百科-Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security)
> TLS 1.0 was first defined in RFC 2246 in January 1999 as an upgrade of SSL Version 3.0, and written by Christopher Allen and Tim Dierks of Consensus Development. As stated in the RFC, "the differences between this protocol and SSL 3.0 are not dramatic, but they are significant enough to preclude interoperability between TLS 1.0 and SSL 3.0". TLS 1.0 does include a means by which a TLS implementation can downgrade the connection to SSL 3.0, thus weakening security.

> The PCI Council suggests that organizations migrate from TLS 1.0 to TLS 1.1 or higher before June 30, 2018

大概意思就是说：[TLS 1.0](https://tools.ietf.org/html/rfc2246)于为[SSL 3.0](https://tools.ietf.org/html/rfc6101)版的升级版，TLS 1.0可以降级为SSL 3.0。

引用了这么多，其实就是想说：现在HTTPS基本都是HTTP基于TLS协议实现的，目前最新的TLS标准版本为[TLS 1.2](https://tools.ietf.org/html/rfc5246)。

现在回到我们的问题，既然HTTPS是HTTP基于TLS的协议，那么要保证HTTP消息（明文）不为破解，那就要讨论TLS的安全性（太旧版本的SSL就不讨论了，这里讨论最新TLS 1.2）。

先来看看完整的TLS协议交互图（图0，来自[RFC 5246](https://tools.ietf.org/html/rfc5246)）：

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/0.png?raw=true)

<center>图0</center>

```
星号（*）表示可选的或者根据消息情况发送的
中括号（[ChangeCipherSpec]）表示是独立的TLS协议内容，目的是防止管道阻塞停顿，并不是TLS握手消息。
```

后面我们会针对TLS协议做实验分析。

# 0x04 实验分析
#### （一）准备
实践是检验真理的唯一标准。没有实验性的东西说起来有点乏味，所以我准备了一个模拟环境，结合例子分析上面的问题。
```
客户端（Client）：
    系统：CentOS 7
    IP：172.16.30.157
    hosts文件：www.grpc.local 172.16.30.106
    已安装的证书包含：给www.grpc.local颁发证书C1的CA的根证书C2
服务端（Server）：
    系统：CentOS 7
    IP：172.16.30.106
    域名：www.grpc.local
    Web服务器：nginx 1.10.1，监听端口：443，虚拟主机：www.grpc.local
    其他：域名www.grpc.local的证书C1，证书C1对应的私钥K1
代理（Proxy）：
    系统：Windows 7
    代理软件：Fiddler 4
    IP:172.16.10.216，监听端口：8888
```

因为资源有限，所以实验环境是模拟的，虽然是模拟的，但也不影响其过程和结论，和真实的没什么区别。

因为这个域名（ www.grpc.local ）不存在的，所以要客户端做一些初始化：

1. hosts文件添加域名（ www.grpc.local ）和IP的（172.16.30.106）的映射，保证在客户端访问www.grpc.local能够访问到服务器172.16.30.106。
2. 已安装有给www.grpc.local颁发证书C1的CA的根证书C2，C1和C2都是自签证书（也就是证书自己生成的）。C2模拟为权威机构的根证书，那么C1就是权威机构颁发的证书，这样客户端正常访问https://www.grpc.local。

不明白的可以先跳过，你就认为www.grpc.local是存在的，是一个正常的域名，客户端访问 https://www.grpc.local 就像访问百度等其他网站的一样，没有什么问题。

#### （二）客户端和服务器存在代理的通信分析
其实有代理存在的时候，我们来看看抓到的报文，看图1的wireshark抓包：

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/-1.png?raw=true)

<center>图1</center>

由图1可以看出，客户端连接的是代理，而客户端的请求通过代理转发和服务端通信。这类似于客户端->服务端（代理充当）和客户端（代理充当）->服务端的通信，相当于两步客户端->服务端的通信，那么下面我们先看看客户端和服务器的通信是怎么样的。

#### （三）客户端和服务器正常通信分析
先上图，看看客户端和服务端通信是怎么样的：

```
注意：下面只是对实验例子的说明，并不是所有HTTPS通信都这样，可以参看图0。
```
##### ● wireshark抓包

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/1.png?raw=true)

<center>图2</center>

##### ● 图解

1-3：TCP的三次握手，客户端和服务端建立连接。

4：客户端发送Client Hello，这一阶段属于握手阶段（Handshake Protocol）。

```txt
Version
    希望使用的协议版本，本例：TLS 1.2
Random
    随机数1，下称R1，后面生成对称加密的密钥时用到。
Session ID
    这个Session ID是继续重用之前的连接用的，如果没有则为空，本例子就是空的。
Cipher Suites
    支持的加密组件，如：TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384，更多可以见附录。
    TLS：TLS协议
    ECDHE_ECDSA：密钥交换算法（密钥交换过程在图2中的报文6-7中）
    AES_256_GCM_SHA384：对称加密算法，协商后密钥后，后面的数据都是通过对称加密算法加密，因为对称加密算法比非对称加密算法快很多。
Compression Methods
    支持的压缩算法，如果Session ID不为空，应该包含之前连接的压缩算法，本例为空。
Extensions
    扩展字段，都不一定有。
    server_name：域名，本例中：www.grpc.local
    signature_algorithms：支持的数字签名的散列/签名算法对。
        如果有，那在后面服务端发给客户端的证书（报文6中的Certificate）必须由该扩展中出现的散列/签名算法对进行签名。
```

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/2.png?raw=true)

<center>图3</center>

5：TCP报文ACK，回应客户端已收到上面（报文4之前）的报文。

6：这一报文中把多个Record Layer消息合并发送了，包含Server Hello、Certificate、Server Key Exchange和Server Hello Done。这一阶段属于握手阶段（Handshake Protocol），后面会对握手阶段发过的信息进行验证，防止被中间人篡改。有些服务器还需要客户端验证，需要客户端验证时，会在Server Hello Done前发送Certificate Request，本例中没有，我们也不展开讨论，其实跟客户端要求服务端发送Certificate差不多的，达到双向认证。


（1）先看看Server Hello，Server Hello和Client Hello有点像打电话时前面的打招呼，当然还包含了其他信息，比如交换随机数R1和R2，后面生成对称加密的密钥需要。

```txt
Random
    随机数2，下称R2，后面生成对称加密的密钥时用到。
Session ID
    这个Session ID跟客户端不一样了，这里不为空。
    如果报文4中的发送的Session ID不为空，且和该步Session ID一样，那么说明之前的连接可以直接重用，直接到报文7中的Encrypted Handshake Message，否则继续后面的步骤。
Cipher Suite
    选择的加密套件，本例为TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384，可以见报文4中Cipher Suites的解释。
Compression Method
    压缩算法。
```

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/3.png?raw=true)

<center>图4</center>

（2）接下来看Certificate，Certificate是为了证明服务器的身份，发证书过去给客户端，客户端验证证书的正确性，如果证书可信，那么继续后面的步骤。
```txt
Certificates
    证书列表（证书链），域名相应的证书（本例中C1）必须为第一个，然后紧跟的必须是上一个的颁发者的证书，本例没有。
Certificate
    证书的内容，必须为X.509v3格式的。内容，暂时先跳过，只要这就是C1证书的内容就行了，后面会详细讲。
signedCertificate
    签名过的证书内容，暂时先跳过，后面会详细讲。
algorithmIdentifier
    算法标识，本例sha256WithRSAEncryption，就是给signedCertificate内容算摘要的算法（SHA-256）和给摘要签名的算法（RSA），后面会详细讲。
encrypted
    用RSA的私钥K1给摘要加密后的密文，这里的私钥K1跟signedCertificate中的subjectPublicKeyInfo（公钥）成对的，不了解可以去了解RSA算法。
```
![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/4.png?raw=true)

<center>图5</center>

（3）再接下来看Server Key Exchange，Server Key Exchange是当Certificate提供的信息不足以生成对称加密的私钥时才发送的，一般当Server Hello中选择的加密套件密钥交换算法为DHE_DSS、DHE_RSA和DH_anon以及在[ECC加密套件文档](https://tools.ietf.org/html/rfc4492)有定义的加密套件时才发送，本例中ECDHE_RSA就是，所以要发送Server Key Exchange，如果是RSA就不需要发送。

* 当密钥交换算法为RSA时，那么客户端可以直接用Certificates中C1的公钥加密需要发送的随机数R3（这是后面的步骤图2报文7中Client Key Exchange），然后服务端用对应私钥K1解密，便得到随机数R3，后面报文7我们再讨论。RSA最为密钥交换算法没有[前向安全性](https://baike.baidu.com/item/前向安全性/6357798)，当私钥K1泄漏是会导致过去的会话密钥泄漏，更换私钥K1和证书C1也不能阻止攻击者算出过去的会话密钥。在TLS 1.3中被废弃了。
* 当密钥交换算法为ECDHE_RSA时（本例），服务端会发送如图6中的信息，每个会话生成一个随机的公钥（图6中的Pubkey）来完成一个秘钥协定，不使用一个确定性算法（本例中用的rsa_pkcs1_sha512），这意味着一条信息的泄露不会泄露其它消息。此算法具有前向安全性。

```txt
EC Diffie-Hellman（ECDH） Server Params
    ECDH算法的参数，这个算法我也没深入看过，有兴趣的自行去搜索。
Pubkey
    服务端的Diffie-Hellman算法公开值，下称DH_PV1，每个会话都不一样，跟后面的客户端公开值DH_PV2结合就能算出对称加密的密钥。
Signature Algorithm
    数字签名算法对（散列/签名）。
    本例是rsa_pkcs1_sha512，散列算法用的SHA512，对散列算法生成的摘要加密的签名算法为RSA。
    也就是用先对Pubkey进行sha512算出摘要，然后用K1对摘要加密得到Signature。这样同时也证明了服务端的身份，因为这是私钥K1加密的。
    算出摘要一个是防篡改，另外一个是RSA对内容长度有限制，而且内容过长耗CPU。
Signature
    看Signature Algorithm解释。
```

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/5.png?raw=true)

<center>图6</center>

（4）最后来看Server Hello Done，Server Hello Done表示服务器已完成密钥交换消息的发送，客户端可以继续其密钥交换阶段（后面的Client Key Exchange）。收到ServerHelloDone消息后，客户端应该检查服务器是否提供了有效的证书（如果发送了，本例是有的）。

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/6.png?raw=true)

<center>图7</center>

7：这一个报文跟报文6一样，把多个Record Layer消息合并发送了，包括握手阶段的Client Key Exchange和变更密码通知协议的（Change Cipher Spec Protocol）的Change Cipher Spec以及握手阶段的Encrypted Handshake Message。如果需要验证客户端身份（也就是服务端发送了Certificate Request），则在Client Key Exchange发送客户端证书，本例没有，就不展开讨论，有需要自行搜索。如果发送了Client Key Exchange，说明客户端信任了报文6中的证书，也是了验证了证书。

（1）先看看Client Key Exchange，这在报文6的（3）也提到了。
* 当密钥交换算法为RSA时，这一步就是用C1的公钥加密随机数R3，然后服务端用私钥K1解密出来，然后结合随机数R1和R2按照一定的算法得到对称加密算法的密钥。中间人没法拿到随机数R3（公钥加密，只能用私钥加密），就没法知道密钥。
* 当密钥交换算法为ECDHE_RSA时（本例），那么如果Pubkey它尚未包含在客户端的证书中（本例没有验证客户端证书），则需要发送Pubkey，见图7。客户端和服务端的两个Pubkey结合能算出对称加密的密钥。

```txt
Pubkey
    客户端的Diffie-Hellman算法公开值，下称DH_PV2，跟服务端的公开值DH_PV1结合就能算出对称加密的密钥。
```

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/7.png?raw=true)

<center>图8</center>

（2）接下来是Change Cipher Spec，Change Cipher Spec并不是TLS握手消息。客户端通知服务器在后续的通信中使用加密模式（对称加密），如图9。

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/8.png?raw=true)

<center>图9</center>

（3）最后来看Encrypted Handshake Message，属于握手阶段（Handshake Protocol）。Encrypted Handshake Message也就是图0中的Finished。该消息在Change Cipher Spec后发送，用来验证密钥交换和身份验证过程是否成功。客户端和服务端都会发送，服务端会在报文8中Encrypted Handshake Message发送。接收方必须验证内容（本报文中内容长度40）是否正确，如果正确则进入图0或图2中Application Data阶段。因为在这之前已经知道对称加密的密钥，下称SCK1，内容是使用SCK1和之前握手阶段（不包括本消息）的消息通过一定的算法（具体没深究）生成的一个摘要消息。如果之前握手数据被篡改过或者不知道SCK1，那么这里的验证阶段是通不过的。

![图片](https://raw.githubusercontent.com/iam2c/blog/master/assets/https_proxy/9.png?raw=true)

<center>图10</center>

8：该报文包括Change Cipher Spec和Encrypted Handshake Message（Finished）两个Record Layer消息。

（1）先看看Change Cipher Spec，Change Cipher Spec跟报文7中的Change Cipher Spec类似，是服务端通知客户端在后续的通信中使用加密模式（对称加密），图跟图9一样的，就不重复贴了。

（2）再看Encrypted Handshake Message，跟报文7的Encrypted Handshake Message差不多，详见报文7的Encrypted Handshake Message。图也是一样的，就不贴了。

到此为止，密钥协商成功，后面将会使用对称加密的密钥SCK1和相关的对称加密算法（见报文6的Server Hello中的Cipher Suite）。

9-10：这里的报文是使用对称加密算法加密的了。其实报文9就是请求报文（如GET /index.html HTTP/1.1），报文10就是相应报文（如HTTP/1.1 200 OK）。没有密钥SCK1是解不出来原来的请求和响应消息（明文）的。

##### ● 安全性分析

作为中间人只能篡改服务器返回的报文，包括握手阶段（Handshake Protocol）还有后面的对称加密阶段（Application Data），也可以窥探客户端和服务端的通信报文。下面我们就这几个问题来分析：

1. 篡改握手阶段（Handshake Protocol）的报文（报文4-8）

    * 握手阶段信息最终都会在Encrypted Handshake Message（Finished）消息里验证（Encrypted Handshake Message也分析了），所有握手阶段的发送和接收到报文都会参与摘要的计算，篡改过的内容没法通过验证的。
    * 当然你也会提出来，把服务端发送的数据篡改，到最后Finished阶段验证也会过啊。那问题是你能修改报文6的Certificate并且能通过报文7之前客户端的证书验证。当然中间人是没法修改的，这是证书体系保证的，证书的验证后面会讲。
    * 当密钥协商算法为ECDHE_RSA等其他需要Server Key Exchange消息的算法时，修改Server Key Exchange消息呢？这个你也别想了，因为Pubkey是通过散列算法算摘要并用证书C1的私钥加密过的，如果篡改了，客户将会无法解密，毕竟中间人不知道私钥K1，那么用证书C1的公钥解出来必将是错误的。


2. 篡改Application Data（报文9-10）

    * 对称加密算法保证了其安全性，不知道密钥SCK1是没法解出来的。
    * 每条信息都会带有消息序号和消息长度、消息内容等，这些内容和固定的字符串参与到摘要的计算中，防止修改和消息重放。

3. 窥探通信报文

    上面篡改握手的报文行不通，那么我看看总可以吧。答案也是不可以的。

    * 前面的握手阶段的就不说了，都是明文的，也不是HTTP请求和响应报文。不过可以看到server_name（见报文4解释）。
    * Application Data也别想了，没有密钥SCK1是没法解出来的。那也有人提问题了，握手阶段都是明文的，怎么就算出来密钥SCK1呢？别搞错了，R1和R2是明文的，R3可是加密的（详见报文7的Client Key Exchange解释），所以没法算出来密钥SCK1。

# 0x05 结论

# 0x06 附录一：加密套件Cipher Suites

# 0x07 附录二：证书和证书的验证

    
