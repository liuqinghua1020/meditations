#pipeline,channel和handler

Channel是Netty编程的核心，可理解为一个管道，数据在管道中流转；Netty自定义了一套自己的Channel，但是在java中要想真正使用java的NIO，还是得用到java的Channel才行。所以，在Netty的Channel中包含了java的Channel(例如SocketChannel和ServerSocketChannel)。

如果需要对在Channel中流转的数据做处理（例如编解码，或是自定义协议的组装和解析），则需要用到pipeline和handler了。pipeline是依附在Channel上的，一个Channel对应一个pipeline。pipeline上包含了处理数据的handler的集合。pipeline和handler采用责任链模式，从第一个handler开始，处理完后，交由pipeline指派下一个handler继续对数据做处理，直到没有handler为止，这是AOP(面向切面编程)的一种体现。


##ChannelPipeline

ChannelPipeline是Netty中 pipeline 的接口，下文是netty源码注释中的一个主要配图：

     -                                       I/O Request
     -                                     via {@link Channel} or
     -                                 {@link ChannelHandlerContext}
     -                                           |
     -  +----------------------------------------+---------------+
     -  |                  ChannelPipeline       |               |
     -  |                                       \|/              |
     -  |  +----------------------+  +-----------+------------+  |
     -  |  | Upstream Handler  N  |  | Downstream Handler  1  |  |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |            /|\                         |               |
     -  |             |                         \|/              |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |  | Upstream Handler N-1 |  | Downstream Handler  2  |  |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |            /|\                         .               |
     -  |             .                          .               |
     -  |     [ sendUpstream() ]        [ sendDownstream() ]     |
     -  |     [ + INBOUND data ]        [ + OUTBOUND data  ]     |
     -  |             .                          .               |
     -  |             .                         \|/              |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |  | Upstream Handler  2  |  | Downstream Handler M-1 |  |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |            /|\                         |               |
     -  |             |                         \|/              |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |  | Upstream Handler  1  |  | Downstream Handler  M  |  |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |             .                          .               |
     -  |             .                         \|/              |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |  |                      |  |          sink          |  |
     -  |  +----------+-----------+  +-----------+------------+  |
     -  |            /|\                         |               |
     -  |             |                         \|/              |
     -  |            /|\                         |               |
     -  +-------------+--------------------------+---------------+
     -                |                         \|/
     -  +-------------+--------------------------+---------------+
     -  |             |                          |               |
     -  |     [ Socket.read() ]          [ Socket.write() ]      |
     -  |                                                        |
     -  |  Netty Internal I/O Threads (Transport Implementation) |
     -  +--------------------------------------------------------+

其中最上面的 I/O Request 表示的是 我们应用程序的读写流，可能是由用户线程(main)发起，也可能是由某一事件触发; 最底层则是 Socket的IO流，负责java Socket IO的读写处理。

从上图可以简单分析下Netty的读写：

1.写
    一般是调用channel(AbstractChannel类，PS：值得注意的是 AbstractServerChannel 不支持 write方法， AbstractServerChannel只是用于接收客户端的连接，具体的读写还是会根据AbstractServerChannel的相关属性重新创建一个AbstractChannel(具体为其子类NioAcceptedSocketChannel))的write方法，其内部调用的是Channels的write方法

        Channels.write(this, message);

Channels是一个关于Channel读写的静态方法的集合。Channels.write()会根据调用的Channel来获取依附在其中的pipeline来处理写，代码如下

    ChannelFuture future = future(channel);
    channel.getPipeline().sendDownstream(
                new DownstreamMessageEvent(channel, future, message, remoteAddress));

其中 channel.getPipeline().sendDownstream() 方法 对应于上图Downstream Handler 这一边的处理。他会调用pipeline上所有符合的Downstream Handler来处理Channel上的数据，直到最后交由Sink来处理。

     DefaultChannelHandlerContext tail = getActualDownstreamContext(this.tail);
     if (tail == null) {
        try {
            getSink().eventSunk(this, e);
            return;
        } catch (Throwable t) {
            notifyHandlerException(e, t);
            return;
        }
     }
     sendDownstream(tail, e);

Sink是专门处理Netty中JAVA NIO的类。


2.读
    读一般由底层Socket触发，在[Netty中的Boss和Worker](./bossAndWorker.md) 描述过 读操作由NioWorker的多路复用器Selector感知，在NioWorker的 read(SelectionKey k) 方法中处理，在将网络数据流读取到Netty的Buffer之后，通过Channels的fireXX方法通知上层应用

        Channels.fireMessageReceived(channel, buffer);

fireMessageReceived方法的内部一样是通过pipeline的sendUpstream方法最终通知到用户进程(main)有数据传输过来

Channels.fireMessageReceived：

     channel.getPipeline().sendUpstream(
            new UpstreamMessageEvent(channel, message, remoteAddress));

channel.getPipeline().sendUpstream：
    
    DefaultChannelHandlerContext head = getActualUpstreamContext(this.head);
        if (head == null) {
            return;
        }
        sendUpstream(head, e);

值得一提的是，UpStream并没有sink对象来做最终处理，Sink处理的是最终传输给网络的数据，UpStream最终处理的Handler是用户编写的Handler逻辑。


##ChannelHandler
ChannelHandler是处理数据的接口，但是他本身并没有定义什么处理方法；真正有用的是ChannelDownstreamHandler和ChannelUpstreamHandler两个子接口(PS: 其实还有一个LifeCycleAwareChannelHandler子接口，还没有作过分析，暂且不论)。

来看下ChannelUpstreamHandler的接口定义：
    
    public interface ChannelUpstreamHandler extends ChannelHandler {
        void handleUpstream(ChannelHandlerContext ctx, ChannelEvent e) throws Exception;
    }

可以看出,ChannelUpstreamHandler只处理UpStream方向，即处理从网络端到用户端的数据流。

同理，看一下ChannelDownstreamHandler的接口定义：

    public interface ChannelDownstreamHandler extends ChannelHandler {
        void handleDownstream(ChannelHandlerContext ctx, ChannelEvent e) throws Exception;
    }

可以看出，ChannelDownstreamHandler只处理DownStream方向，即处理从用户端到网络端的数据流。

如果要自定义一个处理数据的Handler，则需要开发者自己分清楚处理的是从网络端到用户端的数据，还是处理从用户端到网络端的数据，则可分别实现ChannelUpstreamHandler接口或是ChannelDownstreamHandler接口。


##DefaultChannelPipeline
在简单分析了ChannelHandler之后，我们来看下ChannelPipeline的缺省实现DefaultChannelPipeline，在上面分析ChannelPipeline的时候，大部分的方法实现代码都是来自DefaultChannelPipeline。

我们先来看下DefaultChannelPipeline的一些重要属性

    public class DefaultChannelPipeline implements ChannelPipeline {
        private volatile Channel channel;
        private volatile ChannelSink sink;
        private volatile DefaultChannelHandlerContext head;
        private volatile DefaultChannelHandlerContext tail;
    }

其中 channel表示 pipeline所依附的通道， sink 表示数据流到达网络端前的处理，这里还有两个属性head和tail，从名称看来，基本确认是属于链表的头尾节点，而其类为DefaultChannelHandlerContext，从名字来看是属于 DefaultChannelHandler的上下文类，下面在来看下DefaultChannelHandlerContext的类定义
    
    private final class DefaultChannelHandlerContext implements ChannelHandlerContext {
        volatile DefaultChannelHandlerContext next;
        volatile DefaultChannelHandlerContext prev;
        private final String name;
        private final ChannelHandler handler;
        private final boolean canHandleUpstream;
        private final boolean canHandleDownstream;
        private volatile Object attachment;
    }

可以看出，DefaultChannelHandlerContext通过prev和next来组成一个链式结构，而DefaultChannelPipeline则拥有该链的head和tail节点，所以在pipeline上即可以从head访问pipeline上的所有handler，也可以从tail访问pipeline上的所有handler。

DefaultChannelHandlerContext的handler属性是真正处理Channel数据流的对象，canHandleUpstream和canHandleDownstream表明这个Context中的handler能否处理UpStream或是DownStream。

下面主要来分析DefaultChannelPipeline的sendUpstream和sendUpstream，上面的ChannelPipeline有过简单分析，这里在着重分析一下。

sendUpstream提供给外部的方法源码如下
    
    DefaultChannelHandlerContext head = getActualUpstreamContext(this.head);
    if (head == null) { 
        return;
    }
    sendUpstream(head, e);

getActualUpstreamContext方法从pipeline的head出发，不断从DefaultChannelHandlerContext的next查找能处理Upstream的handler，直到找到第一个为止，getActualUpstreamContext方法代码如下：
    
    if (ctx == null) {
        return null;
    }
    DefaultChannelHandlerContext realCtx = ctx; //初次调用的时候，ctx为head
    while (!realCtx.canHandleUpstream()) {
        realCtx = realCtx.next;
        if (realCtx == null) {
             return null;
        }
    }
    return realCtx;


之后，如果head为空，则直接返回，否则调用sendUpstream(head, e)继续处理，sendUpstream(head, e)代码如下：

    try {
        ((ChannelUpstreamHandler) ctx.getHandler()).handleUpstream(ctx, e);
    } catch (Throwable t) {
        notifyHandlerException(e, t);
    }

获取DefaultChannelHandlerContext中的handler来处理数据，在某些非终端的handler处理完数据之后，会调用 ctx.sendUpstream(e) 的方法 继续交由下一DefaultChannelHandlerContext来处理。可以看一下 DefaultChannelHandlerContext的sendUpstream方法。

     DefaultChannelHandlerContext next = getActualUpstreamContext(this.next);
     if (next != null) {
         DefaultChannelPipeline.this.sendUpstream(next, e);
     }

最终DefaultChannelHandlerContext又会通过DefaultChannelPipeline来调用sendUpstream方法处理。

这里说明一下，由于upStream的handler一旦为获取到空，则直接返回，所以一般情况下要确保用户编写的业务upStream的handler一定是添加在pipeline的最后。

sendDownstream方法大体逻辑和sendUpstream方法差不多，不同的是，他是从pipeline的tail出发，不断获取能处理Downstream的handler，直到找到第一个为止。sendDownstream方法源码如下：

    DefaultChannelHandlerContext tail = getActualDownstreamContext(this.tail);
    if (tail == null) {
        try {
            getSink().eventSunk(this, e);
            return;
        } catch (Throwable t) {
            notifyHandlerException(e, t);
            return;
        }
    }
    sendDownstream(tail, e);

和sendUpstream不同的是，如果获取到的tail为空，则会调用sink的eventSunk方法来最终处理网络数据。

在DefaultChannelHandlerContext中sendDownstream的代码也类似

    DefaultChannelHandlerContext prev = getActualDownstreamContext(this.prev);
    if (prev == null) {
        try {
            getSink().eventSunk(DefaultChannelPipeline.this, e);
        } catch (Throwable t) {
            notifyHandlerException(e, t);
        }
    } else {
        DefaultChannelPipeline.this.sendDownstream(prev, e);
    }

Sink具体指的是ChannelSink及其一系列子类，在ChannelSink的源码注释上有这样一句话

    Receives and processes the terminal downstream {@link ChannelEvent}s.

表示主要是在最底层处理downStream的一些信息(主要是ChannelEvent)。


##ChannelEvent

在上面的对pipeline的分析中，我们可以看到不管是sendUpstream还是sendDownstream，传输的都是ChannelEvent。

ChannelEvent是Netty中的事件机制，主要是在pipeline中流转，不同的handler如果碰到自己感兴趣的Event，则进行处理，然后传递给下一个handler。

主要是读写的Event和Channel状态改变的Event，以及一些异常Event，这里不再对这里Event分类说明。





