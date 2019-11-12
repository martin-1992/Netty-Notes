### Netty 解码
　　Netty 解码是将一串二进制流解码成多个 ByteBuf（自定义协议数据包），然后交给业务逻辑进行处理。

### [ByteToMessageDecoder]()
　　Netty 底层的解码器都是基于 ByteToMessageDecoder 实现的，其执行流程是将二进制流数据解析成一个个对象，添加到对象列表中，然后传给下个节点。

### []()


