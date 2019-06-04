[TOC]

# 网络编程之 Socket

* ``<https://www.cnblogs.com/jikexianfeng/p/5729168.html>``

## 网络中进程之间如何通信？

* 网络中进程之间如何通信？首要解决的问题是如何唯一标识一个进程，否则通信无从谈起！在本地可以通过进程PID来唯一标识一个进程，但是在网络中这是行不通的。其实TCP/IP协议族已经帮我们解决了这个问题，网络层的“**ip地址**”可以唯一标识网络中的主机，而传输层的“**协议+端口**”可以唯一标识主机中的应用程序（进程）。这样利用三元组（ip地址，协议，端口）就可以标识网络的进程了，网络中的进程通信就可以利用这个标志与其它进程进行交互。
* 使用TCP/IP协议的应用程序通常采用应用编程接口：UNIX  BSD的套接字（socket）和UNIX System V的TLI（已经被淘汰），来实现网络进程之间的通信。就目前而言，几乎所有的应用程序都是采用socket，而现在又是网络时代，网络中进程通信是无处不在，这就是为什么说“一切皆socket”。

## 什么是 Socket

* Socket 英文直译为“孔或插座”，也称为套接字。用于描述 IP 地址和端口号，是一种进程间的通信机制。你可以理解为 **IP 地址确定了网内的唯一计算机，而端口号则指定了将消息发送给哪一个应用程序**（大多应用程序启动时会主动绑定一个端口，如果不主动绑定，操作系统自动为其分配一个端口）。
* socket起源于Unix，而Unix/Linux基本哲学之一就是“一切皆文件”，都可以用“打开open –> 读写write/read –> 关闭close”模式来操作。Socket就是该模式的一个实现，socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭），这些函数我们在后面进行介绍。
* *Socket**是应用层与**TCP/IP**协议族通信的中间软件抽象层，它是一组接口。在设计模式中，**Socket**其实就是一个门面模式，它把复杂的**TCP/IP**协议族隐藏在**Socket**接口后面，对用户来说，一组简单的接口就是全部，让**Socket**去组织数据，以符合指定的协议。*
* ![](https://img-blog.csdn.net/20141211230815245?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamlhMjgxNDYwNTMw/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## **Socket** 程序一般应用模式及运行流程

* 服务器端会启动一个 Socket，开始监听端口，监听客户端的连接信息，我们称之为 Watch Socket。
* 客户端 Socket 连接服务器端的监听 Socket，一旦成功连接，服务器端会立刻创建一个新的 Socket 负责与客户端进行通信，之后，客户端将不再与 Watch Socket 通信，直接与这个新的 Socket 通信
* Watch Socket 继续监听可能会来自其他客户端的连接。



## Socket 的基本操作

* ```java
  //ocket函数对应于普通文件的打开操作。普通文件的打开操作返回一个文件描述字，而socket()用于创建一个socket描述符（socket descriptor），它唯一标识一个socket。这个socket描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。
  int socket(int domain, int type, int protocol);
  //当我们调用socket创建一个socket时，返回的socket描述字它存在于协议族（address family，AF_XXX）空间中，但没有一个具体的地址(还未绑定)。如果想要给它赋值一个地址，就必须调用bind()函数，否则就当调用connect()、listen()时系统会自动随机分配一个端口。
  ```

* ~~~java
  //bind()函数把一个地址族中的特定地址赋给socket
  int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
  //通常服务器在启动的时候都会绑定一个众所周知的地址（如ip地址+端口号），用于提供服务，客户就可以通过它来接连服务器；而客户端就不用指定，有系统自动分配一个端口号和自身的ip地址组合。这就是为什么通常服务器端在listen之前会调用bind()，而客户端就不会调用，而是在connect()时由系统随机生成一个。
  ~~~

* ~~~java
  //如果作为一个服务器，在调用socket()、bind()之后就会调用listen()来监听这个socket，如果客户端这时调用connect()发出连接请求，服务器端就会接收到这个请求。
  int listen(int sockfd, int backlog);
  //listen函数的第一个参数即为要监听的socket描述字，第二个参数为相应socket可以排队的最大连接个数。
  //socket()函数创建的socket默认是一个主动类型的，listen函数将socket变为被动类型的，等待客户的连接请求。
  ~~~

* ~~~java
  //connect函数的第一个参数即为客户端的socket描述字，第二参数为服务器的socket地址，第三个参数为socket地址的长度。客户端通过调用connect函数来建立与TCP服务器的连接。
  int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
  ~~~

* ~~~java
  //TCP服务器端依次调用socket()、bind()、listen()之后，就会监听指定的socket地址了。TCP客户端依次调用socket()、connect()之后就想TCP服务器发送了一个连接请求。TCP服务器监听到这个请求之后，就会调用accept()函数取接收请求，这样连接就建立好了。之后就可以开始网络I/O操作了，即类同于普通文件的读写I/O操作。
  int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
  //如果accpet成功，那么其返回值是由内核自动生成的一个全新的描述字，代表与返回客户的TCP连接。
  //：accept的第一个参数为服务器的socket描述字，是服务器开始调用socket()函数生成的，称为监听socket描述字；而accept函数返回的是已连接的socket描述字。一个服务器通常通常仅仅只创建一个监听socket描述字，它在该服务器的生命周期内一直存在。内核为每个由服务器进程接受的客户连接创建了一个已连接socket描述字，当服务器完成了对某个客户的服务，相应的已连接socket描述字就被关闭。
  ~~~

* 建立好Socket连接之后，可以调用网络I/O进行读写操作了，即实现了网咯中不同进程之间的通信！，即read()、write()等函数

* ~~~java
  //在服务器与客户端建立连接之后，会进行一些读写操作，完成了读写操作就要关闭相应的socket描述字，好比操作完打开的文件要调用fclose关闭打开的文件。
  int close(int fd);
  //close一个TCP socket的缺省行为时把该socket标记为以关闭，然后立即返回到调用进程。该描述字不能再由调用进程使用，也就是说不能再作为read或write的第一个参数。
  //注意：close操作只是使相应socket描述字的引用计数-1，只有当引用计数为0的时候，才会触发TCP客户端向服务器发送终止连接请求。
  ~~~



## socket中TCP的三次握手建立连接详解

* 最重要的部分其实是建立 tcp 连接
* ![](<http://images.cnblogs.com/cnblogs_com/skynet/201012/201012122157467258.png>)
* 当客户端调用connect时，触发了连接请求，向服务器发送了SYN J包，这时connect进入阻塞状态；服务器监听到连接请求，即收到SYN J包，调用accept函数接收请求向客户端发送SYN K ，ACK J+1，这时accept进入阻塞状态；客户端收到服务器的SYN K ，ACK J+1之后，这时connect返回，并对SYN K进行确认；服务器收到ACK K+1时，accept返回，至此三次握手完毕，连接建立。

## socket中TCP的四次握手释放连接详解

* ![](https://images.cnblogs.com/cnblogs_com/skynet/201012/201012122157494693.png)

* 图示过程如下：
  - 某个应用进程首先调用close主动关闭连接，这时TCP发送一个FIN M；
  - 另一端接收到FIN M之后，执行被动关闭，对这个FIN进行确认。它的接收也作为文件结束符传递给应用进程，因为FIN的接收意味着应用进程在相应的连接上再也接收不到额外数据；
  - 一段时间之后，接收到文件结束符的应用进程调用close关闭它的socket。这导致它的TCP也发送一个FIN N；
  - 接收到这个FIN的源发送端TCP对它进行确认;

## Http Server 的演进之路

* 使用 http 的一个最重要的部分是建立连接，这个连接是用 socket 建立的。

* 标准代码

  ~~~java
  listenfd = socket(...);
  bind(listenfd,服务器的IP和知名端口如80端口,..);
  listen(listenfd,...);
  while(true){
      connfd = accept(listenfd,...);//当客户端调用connect连接请求，listenfd接收到连接请求，则创建一个新的socket 连接
      receive(connfd,...);//然后开始接收客户端发送的数据（在这里有问题，因为网络读取是耗时的操作，这里会引起阻塞，以后的演进就是为了解决这个问题）
      send(connfd,...);//然后发送数据给客户端
  }
  ~~~

* 演进一代，使用 多进程解决，当接收到连接以后，对于这个新的socket ，不在主进程里而是新创新的进程接管，下面的操作就由子进程代理，这样主进程就不会阻塞在 receive 上，可以接收新的的连接请求。但是这样会导致操作系统不堪重负，当请求数量多了的时候，创建了成百上千的进程处理，因为每个进程都要耗费大量的系统资源，光是切换进程就是很麻烦的工作了。

* 演进一代，抛弃上一代的做法。阻塞的本质：因为浏览器还没有把数据发过来，操作系统自然没有数据给服务端，然后服务端又迫不及待的去 receive ，这样操作系统只能阻塞它，在单进程情况下，一阻塞，别的事情自然干不了。

  所以现在采用新的工作方式，服务端接收了客户端的连接以后，不马上去读，而是每次把一批 socket 的编号（其实是 fd_set 数据结构）告诉操作系统，然后再阻塞，然后操作系统拿到这一批socket 后在后台检查，如果发现socket可以读写了，就做个标记在上面，然后再唤醒服务端进程，服务端进程就检查这批socket，处理那些做了标记的socket，然后处理完了，然后再把一批 socket 给 操作系统然后再进入阻塞，以此反复。这种方式叫做 select 模型。

* 演进一代，根据上一代的做法，有一个问题难以解决：服务端进程需要把 socket 编号（其实是 fd_set 数据结构）不断复制给操作系统（应该是复制进内核空间里），这挺消耗资源，还有就是服务端进程从阻塞中恢复以后，需要遍历这些  socket 编号，看看有没有标志位需要处理，而实际情况是大部分 socket 并不活跃，1000多个socket 里可能就几十个可以处理了，但是服务端必须遍历全部 socket。为了解决上面这个问题，提出一种新的方式，叫做 epoll 模型，这个模型和 sclect 差不多，但是，有一点不同，在操作系统检查了一批 socket 后，唤醒服务端进程然后只返回那些可以 处理的 socket，这样，服务端只需要处理这些准备就绪的 socket 即可；