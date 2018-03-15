---
title: iOS网络层详情和优化 # 这是标题
date: 2018-03-15 10:41:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
- 网络层
---
# HTTP
## HTTP方法
HTTP属于应用层。具有以下方法：
* GET 最常见
* HEAD 服务器只返回头部。比如可用于了解资源情况，看看某个对象是否存在，测试资源是否被修改了。
* PUT 向服务器写入文档,是全部更新。
* PATCH 局部更新。比如我们有一个UserInfo，里面有userId、userName、等10个字段。只编辑部分字段进行提交的时候
* POST 写服务器提交数据，通常是表单
* TRACE 允许客户端在最终将请求发送给服务器时，看看请求变成了什么样。因为有可能被防火墙、代理、网关等修改
* OPTIONS 请求服务器告知其支持的各种功能。比如服务器支持哪些方法；对某些特殊资源支持哪些方法等。
* DELETE 请服务器上传请求URL所指定的资源

## 状态码
* 100-199 信息性状态码，。
* 200-299 成功状态码。200是最常见的
* 300-399 重定向。302、304比较常见。比如判断服务器图片是否修改了使用304
* 400-499 客户端错误。
客户端向服务器发送了一些无法处理的东西，比如错误的请求报文，请求一个不存在的URL404
* 400-599 服务器内部错误。
# Api请求过程
当我们向服务器发送一个请求的时候，做了以下事情：

1.DNS Lookup

2.TCP Handshake

3.TLS或SSL Handshake

4.TCP/HTTP Request/Response

![image.png](https://upload-images.jianshu.io/upload_images/6644906-69ddcaded5aadf03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## DNS Lookup
DNS(Domain Name System)域名解析系统，计算机之间的通信不认识域名，只能认识IP，所以DNS就是网络请求的第一步，通过域名获取IP
## TCP Handshake
HTTP属于应用层协议，它不关心网络t通信的具体细节，全部交给TCP/IP。TCP（Transmission Control Protocol 传输控制协议）它完成第四层传输层所指定的功能。
### TCP三次握手连接
第一次握手：建立连接时，客户端发送syn包（syn=j）到服务器，并进入SYN_SENT状态，等待服务器确认；SYN：同步序列编号（Synchronize Sequence Numbers）。

第二次握手：服务器收到syn包，必须确认客户的SYN（ack=j+1），同时自己也发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态；

第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK(ack=k+1），此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手。
![三次握手](http://img.blog.csdn.net/20160914100821925?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
> <font color="red">第三次握手，主机A发送一次确认是为了防止：如果客户端迟迟没有收到服务器返回的确认报文，这时他会放弃连接，重新启动一条连接请求；但问题是：服务器不知客户端没收到，所以他会收到两个连接请求，白白浪费了一条连接开销。</font>

## SSL握手
SSL运行在TCP/IP层之上、应用层之下，为应用层提供加密数据通道

HTTPS比较消耗性能，主要体现在SSL握手消耗的时间。它消耗的时间是TCP握手的时间的3倍甚至更多。

![SSL四次通信握手图](http://www.ruanyifeng.com/blogimg/asset/201402/bg2014020502.png)
>"握手阶段"的所有通信都是明文的。整个握手阶段全部结束。接下来，客户端与服务器进入加密通信，就完全是使用普通的HTTP协议，只不过用"会话密钥"加密内容。
# HTTP和HTTPS时间消耗
>HTTP耗时 = TCP握手
>
>HTTPs耗时 = TCP握手 + SSL握手

# keep-alive
客户端与服务器保持连接的状态，比如长连接

    HTTP/1.1 200 OK
    Connection: Keep-Alive
    Content-Encoding: gzip
    Content-Type: text/html; charset=utf-8
    Date: Thu, 11 Aug 2016 15:23:13 GMT
    Keep-Alive: timeout=5, max=1000
    Last-Modified: Mon, 25 Jul 2016 04:32:39 GMT
    Server: Apache
* max:表示服务器还希望为多少个事物保持此连接的活跃状态
* timeout:服务器能够保持多久活跃状态
# 网络优化
* 优化DNS解析和缓存

APP内置Server IP列表，该列表可以在App启动服务中下发更新。App启动后的首次网络服务会从Server IP列表中取一个IP地址进行TCP连接，同时DNS解析会并行进行，DNS成功后，会返回最适合用户网络的Server IP，那么这个Server IP会被加入到Server IP列表中被优先使用。

Server IP列表有权重机制的，DNS解析返回的IP很明显具有最高的权重，每次从Server IP列表中取IP会取权重最高的IP。列表中IP权重也是动态更新的，根据连接或者服务的成功失败来动态调整，这样即使DNS解析失败，用户在使用一段时间后也会选取到适合的Server IP。

* 网络质量检测（根据网络质量来改变策略）

根据用户是在2G/3G/4G/Wi-Fi的网络环境来设置不同的超时参数，以及网络服务的并发数量。比如在环境比较差的情况下:

  1. 将并发数设置为一个改成串行
  2. 动态设置超时时间
  3. throttle 进行节流

AFNetworking中的throttle方法
```
- (void)throttleBandwidthWithPacketSize:(NSUInteger)numberOfBytes
                                  delay:(NSTimeInterval)delay;
```
  4. 管道化连接
  如果服务器不支持管线化的话，那么响应就会乱序。所以服务器要支持。
  AFNetworking中通过`HTTPShouldUsePipelining`属性来设置，默认为NO。
* 减少数据传输量

选择合适的数据格式进行传输，比如使用Protocol Buffer数等，使用WebP图片格式
* 提供网络服务重发机制

当第一次网络请求失败的时候，自动尝试再次重发
* 使用HTTP缓存

# cookie
cookie分为两类：会话cookie和持久cookie。区别通过Discard参数或者有没有设置Expires或Max-Age参数
