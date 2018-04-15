# 0x01 为啥玩起了gRPC
前些天，Nginx发布了1.13.10，[增加了新的特性：支持gRPC](https://www.nginx.com/blog/nginx-1-13-10-grpc/)。于是跟同事了解了一下，发现gRPC也是用到我们现有项目中用到的[Protocol Buffer](https://github.com/google/protobuf)，而且协议也差不多。现有项目中用到[Swoole](https://github.com/swoole/swoole-src/)+Protocol Buffer，于是想着能不能把Swoole当Server端，提供gRPC服务。

这里先说说gRPC，gRPC是由Google主导开发的RPC框架，使用HTTP/2协议并用ProtoBuf作为序列化工具。其客户端提供Objective-C、Java接口，服务器侧则有Java、Golang、C++等接口，从而为移动端（iOS/Androi）到服务器端通讯提供了一种解决方案。（这段是抄来，[点击这里查看原文](https://blog.csdn.net/omnispace/article/details/79562630)）。也可以[点击这里查看官方文档](https://grpc.io/docs/)。

# 0x02 表象及初诊
说干就干，和同事Z一起研究。下载了gRPC的源码，参考了一些[官方的教程](https://grpc.io/docs/tutorials/basic/php.html)，参考了前人的一些经验，同事Z写了一个Swoole的Server，跑了一些route_guide的demo，发现没有问题（暂时没用HTTPS）。但是Z把demo的循环跑完155次后，第156次没跑完就没有反应了，经过反复试验，总是155次。

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/gRPC_SW/20180415174712.png)

route_guide_client的部分代码如下：

```php
/**
 * Run all of the demos in order.
 */
function main()
{
    runGetFeature();
    runListFeatures();
    runRecordRoute();
    runRouteChat();
}

if (empty($argv[1])) {
    echo 'Usage: php -d extension=grpc.so route_guide_client.php ' .
        "<path to route_guide_db.json>\n";
    exit(1);
}

$i = 0;
$max = 160;
while (++$i <= $max) {
    echo '+++++++++++++++++ ROUND ' , $i, ' +++++++++++++++++', PHP_EOL;
    main();
}
```

到底是哪边出了问题呢？

使用`strace -f -p`追踪进程情况，发现Server端和Client端都在等。

下图是Server端的：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/gRPC_SW/workder-20180415175505.png)

下图是Client端的：

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/gRPC_SW/client-20180415175755.png)

使用`ss -atn|grep 50052`查看连接，显示正常，连接没关闭
![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/gRPC_SW/ss-20180415180320.png)

再使用`tcpdump -ilo tcp port 50052`抓包分析，发现连接确实没有关闭，TCP发包也正常。

# 0x03 分析
解决问题常见手段：控制变量法，使用官方提供的demo中其他语言的Server，于是用了[Python](https://grpc.io/docs/tutorials/basic/python.html)和[Node](https://grpc.io/docs/tutorials/basic/node.html)的Server来提供服务都没有问题，不会在155次后“停止”了。

于是就怀疑是我们用Swoole实现的Server出问题了，问题是哪里出问题了？Swoole底层还是我们基于Swoole写的Server出问题了呢？如果是我们Server代码的问题，那按理说前面155次应该都不能正常服务了，但是结果相反。Z提出会不会是Swoole底层出问题里，比如某个变量在跑了这么多次后越界了。

说一下，我们那时候用的还是比较老的版本1.8.4的，于是后面升级成1.10.x，但是问题依旧，所以应该不是Swoole版本的问题。

查看了抓的数据包，发现最后一个总是DATA帧，长度总是11个字节。

![image](https://raw.githubusercontent.com/iam2c/blog/master/assets/gRPC_SW/tcpdump-20180415181340.png)

于是把之前发送过的数据的长度加起来，每次循环22+5+43+220(22\*10)+132=422字节，155次就是65410字节，加上后面那次不完整的22+5+43+22\*2+11，刚好65535字节。这个数字对于程序员来说肯定不少见，没有每次都必现的巧合，于是怀疑是HTTP2流量控制（flow control）的问题，查看了[HTTP2的文档规范](https://tools.ietf.org/html/rfc7540)，在[section-5.2](https://tools.ietf.org/html/rfc7540#section-5.2)和[section-6.9](https://tools.ietf.org/html/rfc7540#section-6.9)中有相关描述：
```
6.9.2.  Initial Flow-Control Window Size

   When an HTTP/2 connection is first established, new streams are
   created with an initial flow-control window size of 65,535 octets.
   The connection flow-control window is also 65,535 octets.  Both
   endpoints can adjust the initial window size for new streams by
   including a value for SETTINGS_INITIAL_WINDOW_SIZE in the SETTINGS
   frame that forms part of the connection preface.  The connection
   flow-control window can only be changed using WINDOW_UPDATE frames.
        
   Prior to receiving a SETTINGS frame that sets a value for
   SETTINGS_INITIAL_WINDOW_SIZE, an endpoint can only use the default
   initial window size when sending flow-controlled frames.  Similarly,
   the connection flow-control window is set to the default initial
   window size until a WINDOW_UPDATE frame is received.
    
   In addition to changing the flow-control window for streams that are
   not yet active, a SETTINGS frame can alter the initial flow-control
   window size for streams with active flow-control windows (that is,
   streams in the "open" or "half-closed (remote)" state).  When the
   value of SETTINGS_INITIAL_WINDOW_SIZE changes, a receiver MUST adjust
   the size of all stream flow-control windows that it maintains by the
   difference between the new value and the old value.
        
   A change to SETTINGS_INITIAL_WINDOW_SIZE can cause the available
   space in a flow-control window to become negative.  A sender MUST
   track the negative flow-control window and MUST NOT send new flow-
   controlled frames until it receives WINDOW_UPDATE frames that cause
   the flow-control window to become positive.
    
   For example, if the client sends 60 KB immediately on connection
   establishment and the server sets the initial window size to be 16
   KB, the client will recalculate the available flow-control window to 
   be -44 KB on receipt of the SETTINGS frame.  The client retains a
   negative flow-control window until WINDOW_UPDATE frames restore the
   window to being positive, after which the client can resume sending.
   
```

查看翻译，大致意思：
```
当HTTP/2连接首次被建立时，新创建流的初始流量控制窗口大小为65535字节。连接的流量控制窗口也是65535字节。
两端都可以通过组成连接序幕的 SETTING 里的SETTINGS_INITIAL_WINDOW_SIZE来调整新流的初始流量控制窗口的大小。
连接的流量控制窗口只能通过WINDOW_UPDATE帧来改变。
       
在收到设置了 SETTINGS_INITIAL_WINDOW_SIZE 的 SETTIGNS帧之前，当一端发送受流量控制影响的帧时，只能使用默认的初始窗口大小。
相似地，直到收到了WINDOW_UPDATE 帧之前，连接的流量控制窗口都设置为默认的初始窗口大小。
    
除了改变还未激活的流的流量控制窗口，SETTIGNS 帧还可以用激活的流量控制窗口改变流(即，处于 打开(open) 或者 半关闭(远端)(half-closed (remote)) 状态的流)的初始流量控制窗口的大小。
当 SETTINGS_INITIAL_WINDOW_SIZE 的值变化了，接收端必须调整它所维护的所有流的流量控制窗口的值。
    
改变 SETTINGS_INITIAL_WINDOW_SIZE 可以引发流量控制窗口的可用空间变成负值。
发送端必须追踪负的流量控制窗口，并且直到它收到了使流量控制窗口变成正值的WINDOW_UPDATE帧，才能发送新的受流量控制影响的帧。
    
例如，如果连接一建立客户端就立即发送60KB的数据，而服务端却将初始窗口大小设置为16KB，那么客户端一收到 SETTINGS 帧，就会将可用的流量控制窗口重新计算为-44KB。
客户端保持负的流量控制窗口，直到WINDOW_UPDATE帧将窗口值恢复为正值，这之后，客户端才可以继续发送数据。
```

看了这段文档，首先让我看到的是65535这个数字，上面的意思说得很明白，初始流量控制窗口大小为65535字节，可以通过SETTINGS_INITIAL_WINDOW_SIZE改变，而且只能通过WINDOW_UPDATE帧来改变。

查看了Server和Client整个过程的数据包，发现Server虽然有发SETTINGS_INITIAL_WINDOW_SIZE，但是没有发过WINDOW_UPDATE帧（其实这个过程我通过调试很久才发现的，中间过程太长，我就不说了），所以Client没有接受Server的SETTINGS_INITIAL_WINDOW_SIZE值，也就是说还是65535个字节，这就符合上面为啥是65535个字节。

那到底为啥导致两边都在等呢？
```
Server端：
    
因为Swoole底层在收到一个完整的HTTP2消息的时候才会交给上层（也就是我们的Server），在这之前都在阻塞，直到收到一个完整的HTTP2消息，所以上面的epoll_wait返回为0，也就是超时。
    
Client端：
    
这个好理解，因为Client端维护的还是默认的65535个窗口大小，当发送到最后一个包的时候，发现窗口大小不够，只有11个字节，所以只发送了11个字节，不是一个完整的HTTP2消息，在等Server发送的WINDOW_UPDATE帧来更新窗口大小。
```

于是在某个PHP群里面问了一下各位老大Swoole对HTTP2的支持如何，但是没有人回应。好吧，只能自己去看源码了。看了一下源码，发现Swoole对HTTP2的规范实现很少，比如上面的收到数据要发送WINDOW_UPDATE帧告知对方已收到让其更新窗口大小，又如没有维护对方的窗口大小的值，不说是针对具体的流(stream)，连接(connection)的都没有完全实现（只有实现了收到WINDOW_UPDATE帧增加相应大小，但没有发送后减少相应的值）。

```c
/**
 * Http2
 */
int swoole_http2_onFrame(swoole_http_client *client, swEventData *req)
{
    // ...
    if (type == SW_HTTP2_TYPE_HEADERS)
    {
        // ...
    }
    else if (type == SW_HTTP2_TYPE_DATA)
    {
        // ...
    }
    else if (type == SW_HTTP2_TYPE_WINDOW_UPDATE)
    {
        client->window_size = *(int *) (buf + SW_HTTP2_FRAME_HEADER_SIZE);
    }
    // ...
}
```

找到问题点，但是得验证啊，于是我就按照文档规范，发送WINDOW_UPDATE帧，但是不知道在哪里加好，后面就觉得在swoole_http2_do_response()加比较好，因为可以知道收到的数据长度。太多的规范都没有实现，只能先暂时解决当前的问题，只控制整个连接(connection)的窗口大小。下面的是我加的代码：

```c
int swoole_http2_do_response(http_context *ctx, swString *body)
{
    // ...

    if (ctx->request.post_buffer && ctx->request.post_buffer->length > 0) {
        uint32_t size = (uint32_t) ctx->request.post_buffer->length;
        char value[SW_HTTP2_WINDOW_UPDATE_SIZE];
        value[0] = size >> 24;
        value[1] = size >> 16;
        value[2] = size >> 8;
        value[3] = size;
        // 最后一个参数是0，控制的是连接的窗口大小。
        swHttp2_set_frame_header(frame_header, SW_HTTP2_TYPE_WINDOW_UPDATE, SW_HTTP2_WINDOW_UPDATE_SIZE, 0, 0);
        swString_append_ptr(swoole_http_buffer, frame_header, 9);
        swString_append_ptr(swoole_http_buffer, value, SW_HTTP2_WINDOW_UPDATE_SIZE);
    }
    // ...
}
```
测试了一些，没有问题，发送了WINDOW_UPDATE帧了，Client的窗口大小也更新了，循环遍历几百遍都没有问题。

于是给Swoole提了个[PR](https://github.com/swoole/swoole-src/pull/1564)，目前还没有收到回应。

# 0x04 后记
上面的分析过程省略了很多中间细节，不过不影响阅读。Server端的代码没有贴出来，这是我同事Z写的，在此感谢他提供了这段代码。

查找和解决这个问题中，让我学到很多知识，这可能就是兴趣吧。

# 0x05 TODO
还有一些未解决的问题的：

1、Swoole对HTTP2的规范的更多实现。

2、即使发了WINDOW_UPDATE帧，客户端也没有像规范所说的，使用SETTINGS_INITIAL_WINDOW_SIZE值，还是使用65535。