---
title: Tomcat的IO模型与性能调优
date: 2025-10-13 23:30:00 +08:00
categories:
  - Java Web
  - 中间件
  - Tomcat
tags:
  - Tomcat IO模型
  - Linux IO模型
  - Reactor线程模型
  - Tomcat性能调优
  - NIO
  - APR
  - Tomcat线程池参数
  - JConsole监控
---

# 1. Tomcat的I/O模型

## 1.1 Linux I/O模型

### I/O要解决什么问题？

I/O本质上是在解决在计算机内存与外部设备之间拷贝数据的过程

程序通过CPU向外部设备发出读指令，数据从外部设备拷贝至内存需要一段时间，这段时间CPU就没事情做了，程序此时有两种选择：

1. 让出CPU资源，CPU执行其他任务

2. 继续使用CPU轮询数据是否拷贝完成

采取的具体策略就是不同I/O模型要解决的问题

以网络数据读取为例分析，会涉及两个对象，一个是调用I/O操作的用户线程，另一个是操作系统内核。一个进程的地址空间分为用户空间和内核空间，基于安全上的考虑，用户程序只能访问用户空间，内核程序可以访问整个进程空间，只有内核可以直接访问各种硬件资源，比如磁盘和网卡。

<img width="1101" height="613" alt="image" src="https://github.com/user-attachments/assets/28b24729-5077-45e1-98c4-7145582e439f" />


当用户线程发起I/O调用后，网络数据读取操作会经历两个步骤：

* 数据准备阶段：用户线程等待内核将数据从网卡拷贝到内核空间

* 数据拷贝阶段：内核将数据从内核空间拷贝到用户空间（用户进程的缓冲区）

<img width="1080" height="574" alt="image" src="https://github.com/user-attachments/assets/4b6fcf47-be70-46ef-bdef-9a9d32236d3c" />


### Linux的I/O模型分类

* 同步阻塞I/O(bloking I/O)

* 同步非阻塞I/O(non-bloking I/O)

* I/O多路复用（multiplexing I/O）

* 信号驱动式I/O（signal-driven I/O）

* 异步I/O(asynchronous I/O)

其中信号驱动式I/O在实际中并不常用:

<img width="1232" height="906" alt="image" src="https://github.com/user-attachments/assets/8ec4c31e-38ac-4b5f-97c8-e267edf7e5b2" />


* 阻塞或非阻塞I/O是指应用程序在发起I/O操作时，是立即返回还是等待

* 同步或异步是指应用程序在与内核通信时，数据从内核空间到应用空间的拷贝，是由内核主动发起还是应用程序来触发。

BIO的缺点：每个线程只能建立一个连接，所以必须使用线程池来管理多个客户端连接，客户端连接能力有限

NIO的缺点：每次都需要轮询所有的客户连接，确认是否有读取事件，造成CPU的空转，浪费CPU资源

多路复用I/O:事件驱动模型的I/O，每当有读取事件，或者写入事件发布时，会获取事件对应的连接，执行对应的事件，极大利用了CPU资源

| BIO(JioEndpoint)  | 同步阻塞式I/O，即Tomcat使用传统java.io进行操作。该模式下每个请求都会创建一个线程，对性能开销大，不适合高并发场景。优点是稳定，适合连接数小且固定架构。Tomcat8.5.x开始移除BIO                                                     |   |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | - |
| NIO(NioEndPoint)  | 同步非阻塞式I/O，jdk1.4之后实现的新IO。该模式基于多路复用选择器监测连接状态再同步通知线程处理，从而达到非阻塞的目的。比传统BIO能更好的支持并发性能。Tomcat8.0之后默认采用该模式。NIO模式使用连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯，编程比较复杂 |   |
| AIO（Nio2EndPoint） | 异步非阻塞式IO，jdk1.7后支持，与NIO不同在于不需要多路复用选择器，而是请求处理线程执行完成之后进行回调通知，继续执行后续操作。Tomcat8之后支持。一般适用于连接数较多且连接时间较长的应用。                                                     |   |
| APR（APrEndPoint）  | （全称是Apache Portable Runtime/Apache可移植运行库），是ApacheHTTP服务器的支持库。APrEndPoint是通过JNI调用APR本地库而实现非阻塞I/O的。使用需要编译安装APR库                                             |   |

## 1.2 Tomcat I/O模型如何选型

I/O调优实际上是连接器类型的选择，一般情况下默认都是NIO，在绝大多数情况下都是够用的，除非你的WEB应用用到了TLS加密传输，而且对性能要求极高，这个时候可以考虑APR，因为APR通过OoenSSL来处理TLS握手和加解密。OpenSSL本身用C语言实现，它还对TLS通信做了优化，所以性能比Java要高。如果你的Tomcat跑在Windows上，并且HTTP请求的数据量比较大，可以考虑NIO2,这是因为Windows从操作系统层面实现了真正意义的异步I/O，如果传输的数据量比较大，异步I/O的效果就能显现出来。如果你的Tomcat跑在Linux平台上，建议使用NIO。因为在Linux上，JavaNIO和JavaNIO2底层都是通过epoll来实现的，但是JavaNIO更加简单高效。指定IO模型只需要修改server.xml的protocol配置

## 1.3 网络编程模型Reactor线程模型

Reactor模型是网络服务器端用来处理高并发网络IO请求的一种编程模型。

该模型主要有三类处理事件：即连接事件、写事件、读事件；

三个关键角色：即reactor、acceptor、handler。

accpetor负责连接事件，handler负责读写事件，reactor负责事件监听和事件分发

### 单Reactor单线程（redis用的就是这个）：

<img width="955" height="630" alt="image" src="https://github.com/user-attachments/assets/dfa466fd-53dc-47ad-85d8-fc34e73d5df0" />


### 单Reactor多线程模型

<img width="904" height="869" alt="image" src="https://github.com/user-attachments/assets/4b9e3e71-ff29-4191-a895-cee39c158cfb" />


### 主从Reactor多线程

<img width="935" height="970" alt="image" src="https://github.com/user-attachments/assets/e104c179-2062-4457-895e-e795a298d189" />


## 1.4 Tomcat NIO实现

在Tomcat中，Endpoint组件的主要工作就是处理IO，而NIOEndpoint利用Java NIO API实现了多路复用I/O模型。Tomcat的NioEndPoint是基于主从Reactor多线程模型设计的

有一个额外的线程单独处理连接事件，线程池负责处理事件

学习主从Reactor多线程模型这种实现可以去看下thrift实现

# 2. Tomcat调优

## 2.1 如何监控Tomcat的性能

tomcat的关键指标有吞吐量、响应时间、错误数、线程池、CPU以及JVM内存。前三个指标是我们最关心的业务指标，Tomcat作为服务器，就是要能够又快又好地处理请求，因此吞吐量要大、响应时间要短，并且错误数要少。后面三个指标都是跟系统资源有关的，当某个资源出现瓶颈就会影响前面的业务指标，比如线程池种的线程数量不足会影响吞吐量和响应时间；但是线程数太多会耗费大量CPU，也会影响吞吐量，当内存不足时，会频繁触发GC,耗费CPU资源，最后也会反映在业务指标上来

#### 通过JConsole监控Tomcat

1）开启JMX的远程监听端口

我们可以在Tomcat的bin目录下新建一个名为setenv.sh的文件，然后输入下面内容

2\)重启Tomcat,这样JMX的监听端口8011就开启了，可以通过JConsole来连接这个端口

这样就可以监控这些核心业务参数了



## 2.2 常用调优参数



### 线程池参数
``` XML
<Executor name="tomcatThreadPool" 
          namePrefix="catalina-exec-"  <!-- 线程名前缀 -->
          maxThreads="500"            <!-- 最大线程数 -->
          minSpareThreads="50"        <!-- 最小空闲线程 -->
          maxIdleTime="60000"         <!-- 线程空闲超时时间（毫秒） -->
          maxQueueSize="100"          <!-- 任务队列大小 -->
          prestartminSpareThreads="true"/> <!-- 启动时初始化最小空闲线程 -->

<!-- 连接器引用线程池 -->
<Connector executor="tomcatThreadPool"
           port="8080"
           protocol="org.apache.coyote.http11.Http11Nio2Protocol"
           .../>
```
