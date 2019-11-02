### Netty 的主要组件
　　Netty 的主要组件有五个：

- **NioEventLoop，对应线程。** 在用 Socket 编程时，在服务端会使用两个线程，一个负责监听端口接收客户端的连接，另一个则是在接收到连接后新起一个线程负责处理数据的，NioEventLoop 则是同一线程处理这两部分；
- **Channel，对应 Socket。** Netty 的 Channel 底层还是用 Socket，只是将连接 Socket 进行封装，变成了 Channel，有 Nio、Bio 等；
- **ByteBuf，对应 IO Bytes。** 也是将 Java 底层的 IO 相关 API 封装成 ByteBuf；
- **PipeLine，为责任链。** 前面讲到服务端有一个线程来处理数据的，通常一个协议会有多个不同命令，比如登录、发送消息、创建群聊等命令，不同命令对应不同的处理流程。PipeLine 通过添加不同的 ChannelHandler 来处理不同的数据流程；
- **ChannelHandler，责任链上的每个处理节点。** PipeLine 使用链表结构，链表上每个节点为 ChannelHandler，数据进来会经过每个节点 ChannelHandler，每个节点会判断该数据是不是需要处理。
