#Netty读

这里主要以Netty服务端的读为例，其实Netty客户端，服务端的读逻辑都是一致的。读是被动触发，所以其核心在Worker线程的轮询中，其核心代码为

    int selected = select(selector);
    process(selector);

一旦有读事件就绪，则selected的值不为0，然后接下来调用 process(selector) 方法处理读事件。

Worker线程process(selector)是在AbstractNioWorker类中的（关于Worker线程的分析，请查看[Netty中的Boss和Worker](./bossAndWorker.md)），其读方法核心如下


        for (Iterator<SelectionKey> i = selectedKeys.iterator(); i.hasNext();) {
            SelectionKey k = i.next();
            i.remove();
            try {
                int readyOps = k.readyOps();
                if ((readyOps & SelectionKey.OP_READ) != 0 || readyOps == 0) {
                    if (!read(k)) {
                        // Connection already closed - no need to handle write.
                        continue;
                    }
                }
                ...
            }
            ...
        }

接下来便是 read(k) 方法，该方法在NioWorker类中

        final SocketChannel ch = (SocketChannel) k.channel();
        final NioSocketChannel channel = (NioSocketChannel) k.attachment();
        ByteBuffer bb = recvBufferPool.get(predictedRecvBufSize).order(bufferFactory.getDefaultOrder());
        
        while ((ret = ch.read(bb)) > 0) {
            readBytes += ret;
            if (!bb.hasRemaining()) {
                break;
            }
        }
        if (readBytes > 0) {
            bb.flip();
            final ChannelBuffer buffer = bufferFactory.getBuffer(readBytes);
            buffer.setBytes(0, bb);
            buffer.writerIndex(readBytes);
            predictor.previousReceiveBufferSize(readBytes);
            // 通知此事件.
            fireMessageReceived(channel, buffer);
        }

真正读的方法是
    
    (ret = ch.read(bb)) > 0

读完成之后，会调用fireMessageReceived方法通知此事件，fireMessageReceived方法会调用到Channels的fireMessageReceived方法

     channel.getPipeline().sendUpstream(
                new UpstreamMessageEvent(channel, message, remoteAddress));

通过channel的pipeline Upstream读事件，知道到达用户编写的handler的messageReceived方法（关于pipeline，请参考[pipeline,channel和handler](./channelHandler.md)）

至此，netty读分析完毕。


 




