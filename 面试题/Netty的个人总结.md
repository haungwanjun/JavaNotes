



1、介绍一下Netty

Netty是一个异步事件驱动的网络框架。快速开发高性能、高可靠性的网络服务器/客户端程序。

重点是NIO、快速、高性能。

2、Netty的线程模型

​		单线程模型

​		在 ServerBootstrap 调用方法 group 的时候，传递的参数是同一个线程组，且在构造线程 组的时候，构造参数为 1，

​		这种开发方式，就是一个单线程模型。 

​		多线程模型

​		在 ServerBootstrap 调用方法 group 的时候，传递的参数是两个不同的线程组。负责监听 的 acceptor 线程组，线程		数为 1，也就是构造参数为 1。负责处理客户端任务的线程组，线 程数大于 1，也就是构造参数大于 1。这种开发方		式，就是多线程模型。 

​		主从线程模型

​		在 ServerBootstrap 调用方法 group 的时候，传递的参数是两个不同的线程组。负责监听 的 acceptor 线程组，线程		数大于 1，也就是构造参数大于 1。负责处理客户端任务的线程组， 线程数大于 1，也就是构造参数大于 1。这种开		发方式，就是主从多线程模型。 

3、你觉得那个组件最好，最喜欢？

1、EventLoop

每一个channel 分配一个EventLoop，一个EventLoop可以同时处理多个Channel.

每一个Channel分配一个thread。并且所有的工作都由一个Thread完成。

处理每一个channel 的工作都是调用对应的ChannelPipeLine来处理I/O工作。

读事件：直接处理

写事件：封装成task,放入队列。等select()完成再执行。

NioEventLoop 继承于 SingleThreadEventLoop, 而 SingleThreadEventLoop 又继承于 SingleThreadEventExecutor. SingleThreadEventExecutor 是 Netty 中对本地线程的抽象, 它内部有一个 Thread thread 属性, 存储了一个本地 Java 线程。从名字来看，NioEventLoop是一个单线程的，所以，一个 NioEventLoop 其实和一个特定的线程绑定, 并且在其生命周期内, 绑定的线程都不会再改变。

而NioEventLoop在顶层也实现了ExecutorService接口，表示可以向NioEventLoop提交task，由NioEventLoop进行调度执行。实际上，NioEventLoop维护了一个任务队列，可以执行一些定时任务和优先级任务。NioEventLoop 肩负着两种任务, 第一个是作为 IO 线程, 执行与 Channel 相关的 IO 操作, 包括 调用 select 等待就绪的 IO 事件、读写数据与数据的处理等; 而第二个任务是作为任务队列, 执行 taskQueue 中的任务, 例如用户调用 eventLoop.schedule 提交的定时任务也是这个线程执行的。并且，netty的write操作也是通过task来实现的，这就是NioEventLoop设计精巧之处。

NioEventLoop是一个单线程的无限循环，启动时首先会判断是否在循环中，如果在，就直接向NioEventLoop的taskQueue添加task；如果不在，则先启动线程，开始循环，然后向taskQueue添加task。

run方法中的代码看起来复杂，但其实主要的就只有三步：

select操作：轮询注册到reactor线程对应的selector上的所有channel的IO事件

processSelectedKeys操作：处理轮询到的IO事件

runAllTasks操作：处理任务队列的task

2、PipeLine(ChannelHandler/ChannelHandlerContext)

每一个channel都会在创建的时候创建一个channelPipeLine.并且绑定，

ChannelGroup.数据结构是一个Set里面可以存储所有的channel。每一个channel可以添加到多个channelGroup中。

使用的时候可以把所有创建的channel添加进一个全局的Channel中。每一个channel再创建的时候，可以和userId参数绑定。相互之间通信就可以根据userId在channelGroup中去获取这个对象。能获取到就主动发送，（双向的，全双工通信）。如果返回null.表示不在线。入库。

4、你是怎么用的？



4.2 拆包粘包问题解决 
netty 使用 tcp/ip 协议传输数据。而 tcp/ip 协议是类似水流一样的数据传输方式。多次 访问的时候有可能出现数据粘包的问题，解决这种问题的方式如下： 
4.2.1 定长数据流 
客户端和服务器，提前协调好，每个消息长度固定。（如：长度 10）。如果客户端或服 务器写出的数据不足 10，则使用空白字符补足（如：使用空格）。 

4.2.2 特殊结束符 
客户端和服务器，协商定义一个特殊的分隔符号，分隔符号长度自定义。如：‘#’、‘$_$’、 ‘AA@’。在通讯的时候，只要没有发送分隔符号，则代表一条数据没有结束。 

4.2.3 协议 
相对最成熟的数据传递方式。有服务器的开发者提供一个固定格式的协议标准。客户端 和服务器发送数据和接受数据的时候，都依据协议制定和解析消息。 

 

4.3 序列化对象 
JBoss Marshalling 序列化 Java 是面向对象的开发语言。传递的数据如果是 Java 对象，应该是最方便且可靠。 

4.4 定时断线重连 
客户端断线重连机制。 客户端数量多，且需要传递的数据量级较大。可以周期性的发送数据的时候，使用。要 求对数据的即时性不高的时候，才可使用。 优点： 可以使用数据缓存。不是每条数据进行一次数据交互。可以定时回收资源，对 资源利用率高。相对来说，即时性可以通过其他方式保证。如： 120 秒自动断线。数据变 化 1000 次请求服务器一次。300 秒中自动发送不足 1000 次的变化数据。 

4.5 心跳监测 
使用定时发送消息的方式，实现硬件检测，达到心态检测的目的。 心跳监测是用于检测电脑硬件和软件信息的一种技术。如：CPU 使用率，磁盘使用率， 内存使用率，进程情况，线程情况等。 
4.5.1 sigar 
需要下载一个 zip 压缩包。内部包含若干 sigar 需要的操作系统文件。sigar 插件是通过 JVM 访问操作系统，读取计算机硬件的一个插件库。读取计算机硬件过程中，必须由操作系
客户端 WriteTimeoutHandler(3) 
服务器 
1 connect 
2 write 1s 
3 write 3s 
6 write 10s 
4 close 6s 
5 connect 10s 

 统提供硬件信息。硬件信息是通过操作系统提供的。zip 压缩包中是 sigar 编写的操作系统文 件，如：windows 中的动态链接库文件。 解压需要的操作系统文件，将操作系统文件赋值到${Java_home}/bin 目录中。 
4.6 HTTP 协议处理 
使用 Netty 服务开发。实现 HTTP 协议处理逻辑。 
5 流数据的传输处理 
在基于流的传输里比如 TCP/IP，接收到的数据会先被存储到一个 socket 接收缓冲里。不 幸的是，基于流的传输并不是一个数据包队列，而是一个字节队列。即使你发送了 2 个独立 的数据包，操作系统也不会作为 2 个消息处理而仅仅是作为一连串的字节而言。因此这是不 能保证你远程写入的数据就会准确地读取。所以一个接收方不管他是客户端还是服务端，都 应该把接收到的数据整理成一个或者多个更有意思并且能够让程序的业务逻辑更好理解的 数据。 在处理流数据粘包拆包时，可以使用下述处理方式： 使用定长数据处理，如：每个完整请求数据长度为 8 字节等。 （FixedLengthFrameDecoder）
 使用特殊分隔符的方式处理，如：每个完整请求数据末尾使用’\0’作为数据结束标记。 （DelimiterBasedFrameDecoder） 使用自定义协议方式处理，如：http 协议格式等。 使用 POJO 来替代传递的流数据，如：每个完整的请求数据都是一个 RequestMessage 对象，在 Java 语言中，使用 POJO 更符合语种特性，推荐使用。 