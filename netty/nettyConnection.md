#Netty客户端连接

本文以Netty中Echo例子主要分析Netty的客户端的连接操作，这个会涉及客户端和服务端，下面分别来进行描述。


##客户端connect
客户端开始连接的代码为
    
    ChannelFuture future = bootstrap.connect(new InetSocketAddress(HOST, PORT));

调用的是ClientBootstrap的connect方法，经过层层调用，最终调用的方法为

    public ChannelFuture connect(final SocketAddress remoteAddress, final SocketAddress localAddress) {    
        ChannelPipeline pipeline = getPipelineFactory().getPipeline();
        Channel ch = getFactory().newChannel(pipeline);
        ch.getConfig().setOptions(getOptions());
        // 调用netty channel的Connect方法
        return ch.connect(remoteAddress);
    }

这里首先构造一个DefaultChannelPipeline,然后构造一个依附着DefaultChannelPipeline的NioClientSocketChannel， 设置相关的option，最终调用channel的connect方法。

NioClientSocketChannel的connect方法如下

    Channels.connect(this, remoteAddress);

Channels的connect方法源码如下

        ChannelFuture future = future(channel, true);
        channel.getPipeline().sendDownstream(new DownstreamChannelStateEvent(
                channel, future, ChannelState.CONNECTED, remoteAddress));

最终会构造DownstreamChannelStateEvent事件在pipeline上Downstream，最终到达Sink。（关于pipeline请查看[pipeline,channel和handler](./channelHandler.md)）
在NioClientSocketPipelineSink的eventSunk方法中核心代码如下

     case CONNECTED:
            connect(channel, future, (SocketAddress) value);
            break;

看下NioClientSocketPipelineSink的connect方法

        channel.requestedRemoteAddress = remoteAddress;
        try {
            if (channel.channel.connect(remoteAddress)) {
                channel.worker.register(channel, cf);
            } else {
                channel.getCloseFuture().addListener(new ChannelFutureListener() {
                    public void operationComplete(ChannelFuture f)
                            throws Exception {
                        if (!cf.isDone()) {
                            cf.setFailure(new ClosedChannelException());
                        }
                    }
                });
                cf.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                channel.connectFuture = cf;
                nextBoss().register(channel, cf);
            }
        } catch (Throwable t) {
            if (t instanceof ConnectException) {
                Throwable newT = new ConnectException(t.getMessage() + ": " + remoteAddress);
                newT.setStackTrace(t.getStackTrace());
                t = newT;
            }
            cf.setFailure(t);
            fireExceptionCaught(channel, t);
            channel.worker.close(channel, succeededFuture(channel));
        }

最核心的代码即是

    if (channel.channel.connect(remoteAddress)) {
        channel.worker.register(channel, cf);
    }else {
        channel.getCloseFuture().addListener(new ChannelFutureListener() {
            public void operationComplete(ChannelFuture f)
                    throws Exception {
                if (!cf.isDone()) {
                    cf.setFailure(new ClosedChannelException());
                }
            }
        });
        cf.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
        channel.connectFuture = cf;
        nextBoss().register(channel, cf);
     }

java NIO channel 的connect方法上有如下说明
    
     * @return  <tt>true</tt> if a connection was established,
     *          <tt>false</tt> if this channel is in non-blocking mode
     *          and the connection operation is in progress

即如果是阻塞是IO的话，connect成功就返回true，如果是非阻塞是IO的话，则返回false。
由于分析的是NIO的连接流程，所以在调用了connect方法之后返回false，走的是else分支，
至于连接服务端是否成功，主要是下面 

    nextBoss().register(channel, cf);

这句代码，它会将channel和此次操作对应的future组装成NioClientBoss的任务，然后投递到 客户端 Boss线程的 taskQueue 中去，由Boss线程去轮询该任务看是否连接成功。

NioClientBoss线程的RegisterTask的run方法代码如下
    
    int timeout = channel.getConfig().getConnectTimeoutMillis();
    if (timeout > 0) {
        if (!channel.isConnected()) {
            channel.timoutTimer = timer.newTimeout(wakeupTask,
                    timeout, TimeUnit.MILLISECONDS);
        }
    }
    try {
        channel.channel.register(
                boss.selector, SelectionKey.OP_CONNECT, channel);
    } catch (ClosedChannelException e) {
        channel.worker.close(channel, succeededFuture(channel));
    }
    int connectTimeout = channel.getConfig().getConnectTimeoutMillis();
    if (connectTimeout > 0) {
        channel.connectDeadlineNanos = System.nanoTime() + connectTimeout * 1000000L;
    }

其核心代码是向 NioClientBoss的多路复用器Selector注册OP_CONNECT事件。
    
    channel.channel.register(
                boss.selector, SelectionKey.OP_CONNECT, channel);

这样，在Client的Boss线程（相关请查看[Netty中的Boss和Worker](./bossAndWorker.md)） 处理的 processSelectedKeys 逻辑中就有如下代码

     if (k.isConnectable()) {
        connect(k);
     }

而connect 方法代码如下

     if (ch.channel.finishConnect()) {
        k.cancel();
        if (ch.timoutTimer != null) {
            ch.timoutTimer.cancel();
        }
        ch.worker.register(ch, ch.connectFuture);
     }

ch.channel.finishConnect() 返回true，则表明网络连接成功，此时会向worker投递NioWorker的RegisterTask任务，向NioWorker的多路复用器Selector注册Channel的读写事件。客户端连接任务就此完成。


##服务端accept










