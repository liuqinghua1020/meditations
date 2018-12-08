##Netty读

Netty发生在NioEventLoop中，由轮转线程确定，所以首先看 NioEventLoop.run方法的主干<br>

```java
    for(;;){
        ...
        case SelectStrategy.SELECT:
                        // 重置 wakenUp 标记为 false
                        // 选择( 查询 )任务
                        select(wakenUp.getAndSet(false));
        ...  


        // 处理 Channel 感兴趣的就绪 IO 事件
        processSelectedKeys();              

    }
```

首先是select轮训，得到就绪的感兴趣IO事件，然后通过 processSelectedKeys 来处理。<br>
先来看一下select方法<br>

```java
    int selectedKeys = selector.select(timeoutMillis);
    selector.selectNow();
```

整个select方法的处理比较复杂，这里暂不分析，就简单看到这两句，调用的是jdk的selector方法，获取就绪的IO时间，select方法会将就绪的IO相关内容放到  SelectedSelectionKeySet 这个数组里（值得说的是，这个是经过netty优化过的SelectedSelectionKeySet）。<br>

接下来就是processSelectedKeys的处理。

```java
    private void processSelectedKeys() {
        if (selectedKeys != null) {
            processSelectedKeysOptimized();
        } else {
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
```

主要看一下 processSelectedKeysOptimized，这是netty优化过的方法。<br>

```java
        private void processSelectedKeysOptimized() {
        // 遍历数组
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // null out entry in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.keys[i] = null;

            final Object a = k.attachment();

            // 处理一个 Channel 就绪的 IO 事件
            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            // 使用 NioTask 处理一个 Channel 就绪的 IO 事件
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
            ...
        }
    }
```

通过数组，获取到处理的key，从key中获取到netty的一个对象AbstractNioChannel，然后进行processSelectedKey。

```java
    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
            ...
          if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
           }
           ...
    }

```

和accpet类似，read也是调用unsafe的read方法，不同的是accept的unsafe的具体实现类是：NioMessageUnsafe；而read事件unsafe的具体实现类是NioByteUnsafe。

下面看一下 NioByteUnsafe 的 read方法

```java
    public final void read() {
        ...
        do {
                    // 申请 ByteBuf 对象
                    byteBuf = allocHandle.allocate(allocator);
                    // 读取数据
                    // 设置最后读取字节数，这里是从jdk底层读取网络数据的核心代码
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    // <1> 未读取到数据
                    if (allocHandle.lastBytesRead() <= 0) {
                        // 释放 ByteBuf 对象
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        // 置空 ByteBuf 对象
                        byteBuf = null;
                        // 如果最后读取的字节为小于 0 ，说明对端已经关闭
                        close = allocHandle.lastBytesRead() < 0;
                        // TODO
                        if (close) {
                            // There is nothing left to read as we received an EOF.
                            readPending = false;
                        }
                        // 结束循环
                        break;
                    }

                    // <2> 读取到数据

                    // 读取消息数量 + localRead
                    allocHandle.incMessagesRead(1);
                    // TODO 芋艿 readPending
                    readPending = false;
                    // 触发 Channel read 事件到 pipeline 中。 TODO
                    pipeline.fireChannelRead(byteBuf);
                    // 置空 ByteBuf 对象
                    byteBuf = null;
        } while (allocHandle.continueReading()); // 循环判断是否继续读取

        // 读取完成
        allocHandle.readComplete();
        // 触发 Channel readComplete 事件到 pipeline 中。
        pipeline.fireChannelReadComplete();
        ...
    }
```

在从网络中读取相关的信息之后，就会通过pipeline在 handler中进行传播，主要的两个方法是：

- pipeline.fireChannelRead(byteBuf);
- pipeline.fireChannelReadComplete();

pipeline会从HeadContext开始，一路经过netty启动是配置的所有handler，包括用户自行设置的handler；最终，数据可能会流向 TailContext 中。

```java
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            onUnhandledInboundMessage(msg);
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
            onUnhandledInboundChannelReadComplete();
        }

        protected void onUnhandledInboundMessage(Object msg) {
            try {
                logger.debug("Discarded inbound message {} that reached at the tail of the pipeline. " + "Please check your pipeline configuration.", msg);
            } finally {
                //最终释放掉read过程中得到的msg对象
                ReferenceCountUtil.release(msg);
        }

          protected void onUnhandledInboundChannelReadComplete() {
            } 
```

这就是netty中读取数据的简单解析，其他包括读取时申请的内存空间等，后续在做分析。

