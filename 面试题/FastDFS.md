**什么是FastDFS？**

FastDFS是一个开源的轻量级分布式文件系统。它解决了大数据量存储和负载均衡等问题。特别适合以中小文件（建议范围：4KB < file_size <500MB）为载体的在线服务，如相册网站、视频网站等等。在UC基于FastDFS开发向用户提供了：网盘，社区，广告和应用下载等业务的存储服务。

**FastDFS架构：**

FastDFS服务端有三个角色：跟踪服务器（tracker server）、存储服务器（storage server）和客户端（client）。

- **tracker server**：跟踪服务器，主要做调度工作，起负载均衡的作用。在内存中记录集群中所有存储组和存储服务器的状态信息，是客户端和数据服务器交互的枢纽。相比GFS中的master更为精简，不记录文件索引信息，占用的内存量很少。
- **storage server**：存储服务器（又称：存储节点或数据服务器），文件和文件属性（meta data）都保存到存储服务器上。Storage server直接利用OS的文件系统调用管理文件。
- **client**：客户端，作为业务请求的发起方，通过专有接口，使用TCP/IP协议与跟踪器服务器或存储节点进行数据交互。

[![img](http://static.oschina.net/uploads/img/201504/18112619_UG0r.png)](http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS架构图1.png)

Tracker Server：跟踪服务器，主要做调度工作，在访问上起负载均衡的作用。
Storage Server：存储服务器（又称数据服务器）。

**ps：这样的架构具有以下特点：1.轻量级（相比GFS简化了master角色，不再管理meta数据信息）。2.对等结构。3.分组方式。**

**FastDFS协议：**

FastDFS角色间是基于TCP/IP协议进行通信，协议包格式为：header + body。具体结构如图：

[![img](http://static.oschina.net/uploads/img/201504/18112620_aP3V.png)](http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS协议体结构图1.png)

FastDFS各节点间都是通过tcp/ip的方式来进行通信的。
协议包由两部分组成：header和body

**上传机制：**

[![img](http://static.oschina.net/uploads/img/201504/18112620_a4nw.png)](http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS上传流程.png)

**同步时间管理：**

当一个文件上传成功后，客户端马上发起对该文件下载请求（或删除请求）时，tracker是如何选定一个适用的存储服务器呢？

其实每个存储服务器都需要定时将自身的信息上报给tracker，这些信息就包括了本地同步时间（即，同步到的最新文件的时间戳）。而tracker根据各个存储服务器的上报情况，就能够知道刚刚上传的文件，在该存储组中是否已完成了同步。同步信息上报如下图：

[![img](http://static.oschina.net/uploads/img/201504/18112620_pNtX.png)](http://tech.uc.cn/wp-content/uploads/2012/08/管理同步时间.png)

**下载机制：**

[![img](http://static.oschina.net/uploads/img/201504/18112620_7Pt1.png)](http://tech.uc.cn/wp-content/uploads/2012/08/FastDFS下载流程.png)

**精巧的FID：**

说到下载就不得不提文件索引（又称：FID）的精巧设计了。文件索引结构如下图，是客户端上传文件后存储服务器返回给客户端，用于以后访问该文件的索引信息。文件索引信息包括：**组名，虚拟磁盘路径，数据两级目录，文件名**。

[![img](http://static.oschina.net/uploads/img/201504/18112620_6oyp.png)](http://tech.uc.cn/wp-content/uploads/2012/08/精巧的FID.png)

ps：

- **组名：**文件上传后所在的存储组名称，在文件上传成功后有存储服务器返回，需要客户端自行保存。一个组下可以有多个storage，我感觉组就是为管理storage的
- **虚拟磁盘路径：**存储服务器配置的虚拟路径，与磁盘选项store_path*对应。
- **数据两级目录：**存储服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。
- **文件名：与文件上传时不同。**是由存储服务器根据特定信息生成，文件名包含：源存储服务器IP地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。

**快速定位文件：**

知道FastDFS FID的组成后，我们来看看FastDFS是如何通过这个精巧的FID定位到需要访问的文件。

1. 通过组名tracker能够很快的定位到客户端需要访问的存储服务器组，并将选择合适的存储服务器提供客户端访问；
2. 存储服务器根据“文件存储虚拟磁盘路径”和“数据文件两级目录”可以很快定位到文件所在目录，并根据文件名找到客户端需要访问的文件。

 

[![img](http://static.oschina.net/uploads/img/201504/18112620_KnoT.png)](http://tech.uc.cn/wp-content/uploads/2012/08/文件智能定位1.png)

本次分享的主要内容包含：FastDFS各角色的任务分工/协作，文件索引的原理设计以及文件上传/下载操作的流程。通过此次学习我们对FastDFS有了初步的了解，如：

- FastDFS只有三个角色；且跟踪服务器和存储服务器均不存在单点。
- 跟踪服务器被动的接收存储服务器汇报，对存储服务器进行分组管理；并为客户端选定适用的存储服务器。同一存储服务器可以同时向多台跟踪服务器汇报状态信息。
- 存储服务器组内所有存储服务器是对等关系，存储的数据一一对应且相同；所有的存储服务器均是同时在线服务，极大的提高的服务器的使用率，分担了数据访问压力。