[TOC]

# 了解 http 和 https

* `<https://mp.weixin.qq.com/s?__biz=MzAxMTg2MjA2OA==&mid=2649844526&idx=1&sn=bc93498d873952719e92fedb70fc82e1&chksm=83bf6275b4c8eb63984de2b322f317ec31b4d9ff4ac5794430e562ec0e9780c52ea520a23308&mpshare=1&scene=1&srcid=&pass_ticket=VVCYV56tw3aTCBLrkeXVv40HeDbkrrs14nA2RY%2BQ5eJxCJa52DfTbh1B9nqeqwnP#rd>`

## 网络层结构

* 网络结构有两种主流的分层方式：OSI七层模型（理论上的模型，没有成熟的产品）和TCP/IP四层模型（现在的主国际标准）。

  **OSI七层模型和TCP/IP四层模型**

  * OSI是指Open System Interconnect，意为开放式系统互联。

  * TCP/IP是指传输控制协议/网间协议，是目前世界上应用最广的协议。

  

  ![img](https://mmbiz.qpic.cn/mmbiz_png/MOu2ZNAwZwOuyliaNZdicviaozMPibFuhZX3w4kfDtmMnxnntswJknnaUoQEZ3Jjm8xMajlQ8TbrRf0AXKOphonXmw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

  **两种模型区别**

  1. OSI采用七层模型，TCP/IP是四层模型

  2. TCP/IP网络接口层没有真正的定义，只是概念性的描述。OSI把它分为2层，每一层功能详尽。

  3. 在协议开发之前，就有了OSI模型，所以OSI模型具有共通性，而TCP/IP是基于协议建立的模型，不适用于非TCP/IP的网络。

     

## http协议

* Http是基于TCP/IP协议的应用程序协议，不包括数据包的传输，主要规定了客户端和服务器的通信格式，默认使用80端口。

* ### Http请求和响应格式

  ~~~java
  //Request格式
  GET /barite/account/stock/groups HTTP/1.1
  QUARTZ-SESSION: MC4xMDQ0NjA3NTI0Mzc0MjAyNg.VPXuA8rxTghcZlRCfiAwZlAIdCA
  DEVICE-TYPE: ANDROID//设备类型
  API-VERSION: 15
  Host: shitouji.bluestonehk.com//指定服务器域名
  Connection: Keep-Alive//keep-alive表示要求服务器不要关闭TCP连接，close表示明确要求关闭连接，默认值是keep-alive
  Accept-Encoding: gzip//说明自己可以接收的压缩方式
  User-Agent: okhttp/3.10.0//用户代理，是服务器能识别客户端的操作系统（Android、IOS、WEB）及相关的信息。作用是帮助服务器区分客户端，并且针对不同客户端让用户看到不同数据，做不同操作。
  ~~~

  ~~~java
  //Response格式
  HTTP/1.1 200 OK
  Server: nginx/1.6.3
  Date: Mon, 15 Oct 2018 03:30:28 GMT
  Content-Type: application/json;charset=UTF-8//服务器告诉客户端数据的格式，常见的值有text/plain，image/jpeg，image/png，video/mp4，application/json，application/zip。这些数据类型总称为MIME TYPE。
  Pragma: no-cache
  Cache-Control: no-cache
  Expires: Thu, 01 Jan 1970 00:00:00 GMT
  Content-Encoding: gzip//Content-Encoding：服务器数据压缩方式
  Transfer-Encoding: chunked//chunked表示采用分块传输编码，有该字段则无需使用Content-Length字段
  Proxy-Connection: Keep-alive
  
  {"errno":0,"dialogInfo":null,"body":{"list":[{"flag":2,"group_id":1557,"group_name":"港股","count":1},{"flag":3,"group_id":1558,"group_name":"美股","count":7},{"flag":1,"group_id":1556,"group_name":"全部","count":8}]},"message":"success"}
  ~~~

* Content-Length：声明数据的长度，请求和回应头部都可以使用该字段。

* Keep-Alive模式

  * 这个键值对的作用是让HTTP保持连接状态，因为HTTP 协议采用“请求-应答”模式，当使用普通模式，即非 Keep-Alive 模式时，每个请求/应答客户和服务器都要新建一个连接，完成之后立即断开连接（HTTP 协议为无连接的协议）；当使用 Keep-Alive 模式时，Keep-Alive 功能使客户端到服务器端的连接持续有效。

  * Keep-Alive模式下如何知道某一次数据传输结束

    如果不是Keep-Alive模式，HTTP协议中客户端发送一个请求，服务器响应其请求，返回数据。服务器通常在发送回所请求的数据之后就关闭连接。这样客户端读数据时会返回EOF（-1），就知道数据已经接收完全了。
     但是如果开启了 Keep-Alive模式，那么客户端如何知道某一次的响应结束了呢？

    * 如果是静态的响应数据，可以通过判断响应头部中的Content-Length 字段，判断数据达到这个大小就知道数据传输结束了。
    * 但是返回的数据是动态变化的，服务器不能第一时间知道数据长度，这样就没有 Content-Length 关键字了。这种情况下，服务器是分块传输数据的，`Transfer-Encoding：chunk`，这时候就要根据传输的数据块chunk来判断，数据传输结束的时候，最后的一个数据块chunk的长度是0。

* TCP的 keep-Alive

  * HTTP的Keep-Alive与TCP的Keep Alive，有些不同，两者意图不一样。前者主要是 TCP连接复用，避免简历过多的TCP连接。而TCP的Keep Alive的意图是在于保持TCP连接的存活，就是发送心跳包。隔一段时间给连接对端发送一个探测包，如果收到对方回应的 ACK，则认为连接还是存活的，在超过一定重试次数之后还是没有收到对方的回应，则丢弃该 TCP 连接。

* 短连接

  * 所谓短连接，即连接只保持在数据传输过程，请求发起，连接建立，数据返回，连接关闭。它适用于一些实时数据请求，配合轮询来进行新旧数据的更替。

* 长连接

  * 长连接便是在连接发起后，在请求关闭连接前客户端与服务端都保持连接，实质是保持这个通信管道，之后便可以对其进行复用。它适用于涉及消息推送，请求频繁的场景（直播，流媒体）。连接建立后，在该连接下的所有请求都可以重用这个长连接管道，避免了频繁了连接请求，提升了效率。

## https 协议/SSL 协议

*  Https协议是以安全为目标的Http通道，简单来说就是Http的安全版。主要是在Http下加入SSL层（现在主流的是SLL/TLS），SSL是Https协议的安全基础。Https默认端口号为443。

* http存在的风险
  1.  窃听风险：Http采用明文传输数据，第三方可以获知通信内容
  2.  篡改风险：第三方可以修改通信内容
  3.  冒充风险：第三方可以冒充他人身份进行通信
* SSL/TLS协议就是为了解决这些风险而设计，希望达到：
  1. 所有信息加密传输，三方窃听通信内容
  2. 具有校验机制，内容一旦被篡改，通信双发立刻会发现
  3. 配备身份证书，防止身份被冒充

* ### **SSL原理及运行过程**

  * Secure Sockets Layer 安全套接层

  * SSL/TLS协议基本思路是采用公钥加密法（最有名的是RSA加密算法）。大概流程是，客户端向服务器索要公钥，然后用公钥加密信息，服务器收到密文，用自己的私钥解密。
  * 为了防止公钥被篡改，把公钥放在数字证书中，证书可信则公钥可信。公钥加密计算量很大，为了提高效率，服务端和客户端都生成对话秘钥，用它加密信息，而对话秘钥是对称加密，速度非常快。而公钥用来解密对话秘钥。

##　加密算法

* 加密算法分类
* 加密算法分为对称加密、非对称加密和Hash加密算法。
  * 对称加密：甲方和乙方使用同一种加密规则对信息加解密
  * 非对称加密：乙方生成两把秘钥（公钥和私钥）。公钥是公开的，任何人都可以获取，私钥是保密的，只存在于乙方手中。甲方获取公钥，然后用公钥加密信息，乙方得到密文后，用私钥解密。
  * Hash加密：Hash算法是一种单向密码体制，即只有加密过程，没有解密过程

* 对称加密算法加解密效率高，速度快，适合大数据量加解密。常见的堆成加密算法有DES、AES、RC5、Blowfish、IDEA
* 非对称加密算法复杂，加解密速度慢，但安全性高，一般与对称加密结合使用（对称加密通信内容，非对称加密对称秘钥）。常见的非对称加密算法有RSA、DH、DSA、ECC

* Hash算法特性是：输入值一样，经过哈希函数得到相同的散列值，但并非散列值相同则输入值也相同。常见的Hash加密算法有MD5、SHA-1、SHA-X系列

## Http和Https的区别

1. https协议需要到CA申请证书，大多数情况下需要一定费用
2. Http是超文本传输协议，信息采用明文传输，Https则是具有安全性SSL加密传输协议
3. Http和Https端口号不一样，Http是80端口，Https是443端口
4. Http连接是无状态的，而Https采用Http+SSL构建可进行加密传输、身份认证的网络协议，更安全。
5. Http协议建立连接的过程比Https协议快。因为Https除了Tcp三次握手，还要经过SSL握手。连接建立之后数据传输速度，二者无明显区别。