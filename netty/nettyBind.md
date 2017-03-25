#Netty服务器启动

    本文主要以netty-3.10的 org.jboss.netty.example.echo.EchoServer 的代码来分析netty NIO 服务端的启动流程。

##EchoServer源码
EchoServer只要一个main启动方法，以下是其核心代码：

        ／／初始化 ServerBootstrap 这一辅助构造服务器的类
        ServerBootstrap bootstrap = new ServerBootstrap(
                new NioServerSocketChannelFactory(
                        Executors.newCachedThreadPool(),
                        Executors.newCachedThreadPool()));

        //构造 pipelineFactory，以便能从factory类中构造 pipeline
        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            public ChannelPipeline getPipeline() {
                ChannelPipeline p = Channels.pipeline();
                p.addLast("echo", new EchoServerHandler());
                return p;
            }
        });

        ／／设置一些网络参数
        bootstrap.setOption("child.tcpNoDelay", true);
        bootstrap.setOption("child.receiveBufferSize", 1048576);
        bootstrap.setOption("child.sendBufferSize", 1048576);

        //绑定指定端口，并开始接受连接。
        bootstrap.bind(new InetSocketAddress(PORT));

下面开始逐个步骤分析
### 初始化 ServerBootstrap
ServerBootstrap 是一个辅助构造 服务端的类，里面主要是设置一些参数和配置信息，具体的可以去研究下这个类，不再展开阐述，这里主要说明一下该构造方法传入的参数：NioServerSocketChannelFactory。

从命名上我们可以看出，(AA)Factory之类的方法在Netty中主要的作用就是为了初始化前面的(AA)所存在的类，NioServerSocketChannelFactory这个类主要的作用则是构造NioServerSocketChannel,而在该构造方法中，重点是其两个参数： Executors.newCachedThreadPool() 。

通过调试代码，可以追溯到 最终的NioServerSocketChannelFactory构造方法的原型：

    public NioServerSocketChannelFactory(BossPool<NioServerBoss> bossPool, WorkerPool<NioWorker> workerPool);

注意到两个参数的名称，(AA)Pool 表明的是(AA)这个对象的对象池，而重点需要关注的是这里两个对象NioServerBoss和NioWork，关于这两个对象，请查看本系列的另一篇文章[Netty中的Boss和Worker](./bossAndWorker.md)，这里主要说一下，构造bossPool和workerPool的过程，其中是启动了Boss线程(NioServerBoss类)和16个Work线程(NioWork类)，这里简单描述一下 NioServerBoss 和 NioWork线程的主要用，更详细的信息请查看文章[Netty中的Boss和Worker](./bossAndWorker.md)：

* Boss线程的主体是一个for循环， 主要处理 指派给Boss线程的任务(task)和处理其NioServerBoss对象内部的多路复用器selector的轮询。

* Worker线程的主体也是一个循环， 主要处理的也是指派给 Worker线程的任务(task)和处理其NioWorker对象内部的多路复用器selector的轮询。

 从上面的描述可以看出Boss线程做的事和Worker线程做的事差不多，当然，因为他们是继承同一个类AbstractNioSelector。
 不同的是 指派给 他们的任务不一样,NioServerBoss和NioWorker内部分别由一个RegisterTask内部类，分别用于构造指派给Boss线程和Worker线程的任务类；
 同时，两者的多路复用器selector关注的事件也不一样，Worker的多路复用器selector主要关注的是读写事件，Boss的多路复用器主要关注的是连接事件(奇怪的是，netty-3.10的代码中 没有在NioServerBoss的selector中注册Channel的连接事件，采用的是另一种方式)。

    ServerBootstrap bootstrap = new ServerBootstrap(
            new NioServerSocketChannelFactory(
                    Executors.newCachedThreadPool(),
                    Executors.newCachedThreadPool()));

此段代码完成之后，用户线程(main)线程负责继续往下执行用户代码，Boss线程和16个Worker线程则通过轮询不断查看是否有Channel就绪或是有task任务需要执行。

##构造 pipelineFactory

pipeline相关的信息请查阅系列另外一篇文章[pipeline,channel和handler](./channelHandler.md)。
这里的代码主要是在pipeline中设置相关的处理handler，以便在相关的事件发生时，能交由对应的handler类处理，例子中自己实现了一个EchoServerHandler，其中主要有两个个方法：

1.messageReceived 当有消息从客户端传过来时会回调此方法。

2.exceptionCaught 当异常发生时会回调此方法。


##设置一些网络参数
不再赘述，就是设置一些socket编程的参数。

##绑定指定端口，并开始接受连接
此为服务端启动的核心方法，其核心在 ServerBootstrap 的 

    bindAsync(final SocketAddress localAddress) 方法。

bindAsync方法的核心逻辑如下：
    
    Binder binder = new Binder(localAddress);
    ChannelPipeline bossPipeline = pipeline();
    bossPipeline.addLast("binder", binder);
    Channel channel = getFactory().newChannel(bossPipeline);

Binder是一个handler，通过调用pipeline的addLast方法将其加入到pipeline中，在这段代码中Binder还没有发挥其作用，只是设置到了pipeline中而已。
这段代码块的重点在于 newChannel 这个方法，在服务端，该方法主要是NioServerSocketChannelFactory的newChannel方法， 用于创建NioServerSocketChannel对象。

那么，重点来了，在NioServerSocketChannel构造方法中，有如下核心代码：
    
        socket = ServerSocketChannel.open();
        socket.configureBlocking(false);
        fireChannelOpen(this);

前两行代码就是java NIO编程的代码，这里不在赘述，可以查看 [Netty基础杂谈](./nettyBase.md) ;fireChannelOpen方法的逻辑如下：

    channel.getPipeline().sendUpstream(
                new UpstreamChannelStateEvent(
                        channel, ChannelState.OPEN, Boolean.TRUE));

这里代码块可查看 [pipeline,channel和handler](./channelHandler.md)。这里简单说明一下，主要是 构造一个 UpstreamChannelStateEvent 事件，然后在pipeline里面流转该事件，最终会调用到Binder类的handleUpstream方法(其实是SimpleChannelUpstreamHandler类的handleUpstream方法，因为Binder继承自SimpleChannelUpstreamHandler)。
为什么最终会调用到Binder的handleUpstream方法捏，主要是上面代码块中的

    bossPipeline.addLast("binder", binder);

这一方法。

Binder的handleUpstream方法会根据接受到的不同的event事件，分发不同的处理，当接受到上述的事件的处理代码如下：
    
        case OPEN:
            if (Boolean.TRUE.equals(evt.getValue())) {
                channelOpen(ctx, evt);
            } else {
                channelClosed(ctx, evt);
            }

Binder的channelOpen方法如下：
    
    evt.getChannel().bind(localAddress).addListener(new ChannelFutureListener() {
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isSuccess()) {
                        bindFuture.setSuccess();
                    } else {
                        bindFuture.setFailure(future.getCause());
                    }
                }
    });

bind方法返回的是一个future，给其添加listener，在其完成之后，可以回调listener里面的方法，这里listener的方法处理的是 说明bind操作已经成功。

那么，重点就又到了evt.getChannel().bind(localAddress)这里方法上。看其源码处理的逻辑是：
    
        ChannelFuture future = future(channel);
        channel.getPipeline().sendDownstream(new DownstreamChannelStateEvent(
                channel, future, ChannelState.BOUND, localAddress));
        return future;

future就是上面说的添加listener的future，这里的重点就是pipeline的sendDownstream。(话说，这一过程也真够复杂的，先是sendUpstream，再是sendDownstream，都快绕晕了。)
最终，该方法会调用到NioServerSocketPipelineSink的eventSunk方法（至于为啥，请参考[pipeline,channel和handler](./channelHandler.md)）。

eventSunk的核心逻辑如下：
    
        Channel channel = e.getChannel();
        if (channel instanceof NioServerSocketChannel) {
            handleServerSocket(e);
        } else if (channel instanceof NioSocketChannel) {
            handleAcceptedSocket(e);
        }

由于这里分析的是服务端启动过程，所以走的是  handleServerSocket 这一分支，handleServerSocket处理方法如下：
    
    switch (state) {
        case OPEN:
            if (Boolean.FALSE.equals(value)) {
                ((NioServerBoss) channel.boss).close(channel, future);
            }
            break;
        case BOUND:
            if (value != null) {
                ((NioServerBoss) channel.boss).bind(channel, future, (SocketAddress) value);
            } else {
                ((NioServerBoss) channel.boss).close(channel, future);
            }
            break;
        default:
            break;
    }

走的是BOUND分支，调用的是 NioServerBoss 的bind 方法，该方法如下：
    
    registerTask(new RegisterTask(channel, future, localAddress));

向NioServerBoss的任务队列中注册了一个NioServerBoss的RegisterTask任务对象。Boss线程在for循环过程中会从任务队列中获取一个RegisterTask任务，调用起run方法。
NioServerBoss的RegisterTask的run方法逻辑如下：

    channel.socket.socket().bind(localAddress, channel.getConfig().getBacklog());
    bound = true;
    future.setSuccess();
    fireChannelBound(channel, channel.getLocalAddress());
    channel.socket.register(selector, SelectionKey.OP_ACCEPT, channel);
    registered = true;

终于看到核心代码了！！！！

    channel.socket.socket().bind(localAddress, channel.getConfig().getBacklog());
    channel.socket.register(selector, SelectionKey.OP_ACCEPT, channel);

这两句基本就是 java NIO的服务端的核心代码了，在register完成之后，Boss线程的多路复用器selector在有客户端连接上来之后，则会channel就绪。


这里需要说明一个关系是：

* 一个 channel <---> 一个 pipeline
* 一个 channel <---> 一个 boss线程
* 一个 channel <---> 16个 worker线程
































