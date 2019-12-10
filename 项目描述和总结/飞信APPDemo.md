项目描述：

这是一个基于Netty框架的类似于微信一样的聊天App，我的目的是学习聊天服务器的开发。学习了一般聊天服务器使用的是webSocket协议，他是一个长连接。一旦建立间接之后可以一直通信，不同于Http的短链接。但是因为资源的原因，我们可以设置心跳机制。让它闲置的channel超时后自动断开。

每个用户连接是一个channel,在刚开始的时候便绑定，一个channel有一个selectKey用于workerEventLoop监听事件和处理，收到请求后，把写事件封装成一个个task放到taskQueue中，然后等读时间和accept事件(也是读事件)都处理完了，再去处理task。EventLoop是一个无限循环，netty几乎所有的事件和任务都是再EventLoop中执行得。而这些事件或任务大多会通过netty的ChannelPipeLine的Channelhandler链执行下去。每一个channel(链接)都会被绑定到一个管道上面，这条连接的读写操作都会通过这个管道里面的各种Channelhandler处理。

处理事件：读任务：通过pipeLine里面的ChannelinboundHandler处理读事件

写任务：通过ChanneloutboundHandler（都是一个handler链）处理写任务：

其中ChannelPipeLine有两个节点：分别是head和tail.读事件是head->ChannelinboundHandler链->tail;

写事件（task）： ChanneloutboundHandler链->head；因为head持有unsafe对象，可以直接操作底层缓冲区。与IO关系最密切。

具体细节：每一个user分配一个Channel.由workerGroup管理。bossGroup只管理接收请求，收到请求后，把这个请求封装成一个Channel，丢给workGroup进行管理，本质(每条线程)都是一个selector;线程组就有多个Selector,每个Selector只负责监控注册在它上面的channel.每个Selector的执行流程是，每一次循环，Select()轮询所有已经发生的事件selectKey，（每个selectKey 绑定了一条通道，每个通道一条连接）比如度或者写事件发生，所有实际操作都是UnSafe类是实现的，然后把这些事件包装成一个个Task(或者定时task),放到任务队列，然后下一个步骤就是去执行task，

双人（多人）通信：因为每人都有一个userID都绑定了一个channel,我们可以根据接收到的消息的接收方ID找到对应的channel,如果找不到说明不在线。以未签收状态入库，等用户上线后拉去，如果在线，入库后直接推送，签收后修改 状态即可。根据消息是否签收的状态，显示不同的状态，把每个人的历史记录都插到本地缓存的一个列表中。包括自己发送的关和接受的，根据时间线来存储。

添加好友：根据ID准确搜索用户，找到便添加。发送一个好友请求消息。未签收状态入库，如果在线直径推送。不在线等上线后推送。

通讯录，每次点击到这个页面，读取本地缓存。如果有添加好友等操作，成功后自动更新本地缓存，不用每次都去数据库里面读。

个人信息：缓存本地一份，每次更新后入库并更新缓存。二维码也是使用插件生成。保存账号信息。这里没有 加密，实际上应该是要使用加密算法加密的。只有账号信息没有吉他信息，所以也没什么关系。

头像上传到fastDFS文件服务器：使用Nginx代理的：



难点：数据库的设计：

比如好友：每一个好友请求需要添加两条记录。分别是userID和friendID.对调后在存储一次。因为我的好友是所有和我userID绑定的对应的ID;

不过这个可以使用分组查询优化：select friendID from my_friend  group by userID Having userID = "0123131 ";

朋友圈的设计：每个人发送一条朋友圈。那便像所有的未屏蔽的朋友都推送一遍。然后再他的缓存里面根据时间线插入对应朋友的朋友圈消息队列。然后这条消息还应该有一个字段来存储那些人评论和点赞了。这也会和其他人进行推送。但是只会推送到都能看到这条 朋友圈并且是好友的人的channel.这样才能双方可见。可以使用inner join ：既是这条消息的接收者也是自己的好友，就会显示。