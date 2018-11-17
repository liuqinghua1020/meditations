##Netty服务端接受连接

上一篇讲解了netty NIO服务端的启动。启动之后，网络相关的处理都会在NioEventLoop中循坏完成，上文中提到，Netty服务端会使用一个bossGroup处理网络接收请求

```java
    EventLoopGroup bossGroup = new NioEventLoopGroup(1); 
```
此bossGroup会有一个线程启动NioEventLoop循坏处理网络请求，下面来具体看一下NioEventLoop中的处理。
NioEventLoop的核心在其run方法上，其核心是一个 死循环

```java
     for (;;) {
         ...
          select(wakenUp.getAndSet(false));
          ...
     }
```

首先是一个select操作

```java
private void select(boolean oldWakenUp) throws IOException {
    // 记录下 Selector 对象
    Selector selector = this.selector;
    for (;;) {
        // 计算本次 select 的超时时长，单位：毫秒。
        long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
        // 如果超时时长，则结束 select
        if (timeoutMillis <= 0) {
            if (selectCnt == 0) { // 如果是首次 select ，selectNow 一次，非阻塞
                selector.selectNow();
                selectCnt = 1;
            }
            break;
        }

        // 若有新的任务加入
        if (hasTasks() && wakenUp.compareAndSet(false, true)) {
            // selectNow 一次，非阻塞
            selector.selectNow();
            // 重置 select 计数器
            selectCnt = 1;
            break;
        }

        // 阻塞 select ，查询 Channel 是否有就绪的 IO 事件
        int selectedKeys = selector.select(timeoutMillis);
        // select 计数器 ++
        selectCnt ++;

        // 结束 select ，如果满足下面任一一个条件：
        // 1. 有 selectedKeys 就绪
        // 2. oldWakenUp 为true
        // 3. wakenUp.get() 被别的线程手动唤醒
        // 4.hasTasks() / hasScheduledTasks() 从别的线程有任务提交到其EventLoop中
        if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
            break;
        }
        // 线程被打断。一般情况下不会出现，出现基本是 bug ，或者错误使用。
        if (Thread.interrupted()) {
            selectCnt = 1;
            break;
        }

        // 记录当前时间
        long time = System.nanoTime();
        // 符合 select 超时条件，重置 selectCnt 为 1
        if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
            // timeoutMillis elapsed without anything selected.
            selectCnt = 1;
        // 不符合 select 超时的提交，若 select 次数到达重建 Selector 对象的上限，进行重建
        } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {

            // 重建 Selector 对象
            rebuildSelector();
            // 修改下 Selector 对象
            selector = this.selector;

            // 立即 selectNow 一次，非阻塞
            selector.selectNow();
            // 重置 selectCnt 为 1
            selectCnt = 1;
            // 结束 select
            break;
        }

        currentTimeNanos = time;
    }
}
```

里面有很多逻辑和判断，例如 解决 selector空轮训问题等（通过计数确认空轮询，然后通过重建selector并转移Key达到解决空轮询的方法），不过对于JDK NIO操作来说，最主要的就是以下几句代码：

```java
    selector.select(timeoutMillis);
    selector.selectNow();
```

通过select操作，可以把就绪的Key放到一个集合中，这里值得说明一下的是：
为了提升效率，netty借助反射，将JDK底层Selector中的selectedKeys由原来的Set集合替换成了selectedKeys数组。

接下来是处理就绪的Key，方法为：

```java
    private void processSelectedKeysOptimized() {
        // 遍历就绪Key数组
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
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

            // TODO 1007 NioEventLoop cancel 方法
            if (needsToSelectAgain) {
                selectedKeys.reset(i + 1);
                selectAgain();
                i = -1;
            }
        }
    }
```

接下来是 processSelectedKey 方法：

```java
    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        ...
            // 获得就绪的 IO 事件的 ops
            int readyOps = k.readyOps();
           if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        ...
    }
```

接下来是 NioMessageUnsafe 的 read方法

```java
   public void read() {
       ...
       // 读取客户端的连接到 readBuf 中
       int localRead = doReadMessages(readBuf);
       ...
   }
```

来看一下 doReadMessages的处理

```java
     protected int doReadMessages(List<Object> buf) throws Exception {
         // 接受客户端连接
         SocketChannel ch = SocketUtils.accept(javaChannel());
         //转化为 Netty 自定义的 NioSocketChannel 返回
         buf.add(new NioSocketChannel(this, ch));
     }
```

在接收完所有的连接之后，接下来继续回到 NioMessageUnsafe 的 read()方法来看
```java
    public void read() {
        // 循环 readBuf 数组，触发 Channel read 事件到 pipeline 中。
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            // 在内部，会通过 ServerBootstrapAcceptor ，将客户端的 Netty NioSocketChannel 注册到 EventLoop 上
            pipeline.fireChannelRead(readBuf.get(i));
        }
        // 清空 readBuf 数组
        readBuf.clear();
        // 读取完成
        allocHandle.readComplete();
        // 触发 Channel readComplete 事件到 pipeline 中。
        pipeline.fireChannelReadComplete();
    }
```

其中最核心的方法是：
pipeline.fireChannelRead(readBuf.get(i));（pipeline处理后面会分析）
通过ServerBootstrapAcceptor来处理，下面来看一下 ServerBootstrapAcceptor中的处理逻辑

```java
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
         // 接受的客户端的 NioSocketChannel 对象
         final Channel child = (Channel) msg;
         // 添加 NioSocketChannel 的处理器
         child.pipeline().addLast(childHandler);

         // 注册客户端的 NioSocketChannel 到 work EventLoop 中。
         childGroup.register(child).addListener(new ChannelFutureListener() {

            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                // 注册失败，关闭客户端的 NioSocketChannel
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }

        });
    }
```

其中childHandler是我们在启动server端是设置的自定义handler

```java
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) throws Exception { // 设置连入服务端的 Client 的 SocketChannel 的处理器
            ChannelPipeline p = ch.pipeline();
            if (sslCtx != null) {
                p.addLast(sslCtx.newHandler(ch.alloc()));
            }
            p.addLast(new LineBasedFrameDecoder(Integer.MAX_VALUE));
            p.addLast(serverHandler);
        }
    })
```

接下来就是交给 childGroup来进行注册register(child)，调用的是 MultithreadEventLoopGroup 的register方法

```java
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
```

next()方法使用chooser进行EventLoop的选择，得到一个 NioEventLoop,然后注册Channel。

接下来是 SingleThreadEventLoop 的 register方法：

```java
     @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        // 注册 Channel 到 EventLoop 上
        promise.channel().unsafe().register(this, promise);
        // 返回 ChannelPromise 对象
        return promise;
    }
```

下面是 AbstractUnsafe 的 register方法

```java
     @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            register0(promise);
        }
```

下面是 AbstractUnsafe 的 register0方法：

```java

    private void register0(ChannelPromise promise) {
         // 执行注册逻辑
         doRegister();

         pipeline.invokeHandlerAddedIfNeeded();

         // 回调通知 `promise` 执行成功
         safeSetSuccess(promise);

         // 触发通知已注册事件
         pipeline.fireChannelRegistered();
    }

```

核心在于 doRegister 方法，其子类实现为 AbstractNioChannel的doRegister方法：

```java
    protected void doRegister() throws Exception {
        selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
    }
```

至此，整个接收连接完成，后续就是 这个 child NioEventLoop 轮询客户端读写操作了。






























