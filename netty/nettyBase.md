#Netty基础杂谈

    Netty作为一个java的网络框架，主推的是nio，即所谓的非阻塞式IO，所以其中蕴涵了多种开发或设计的思想，旨在设计一个高效的非阻塞式框架。在探讨Netty源码之前，首先还是对其中一些基础内容做一些简介，才能更好的阐述Netty框架。

##通道Channel
Channel是一个通道，可以通过它读取和写入数据，通道和以往的流不同的地方是通道是双向的，流只能在一个方向移动，而且通道可以用于读，写或是同时进行读写。

java中有很多channel，包括文件的channel（FileChannel），网络的channel(SocketChannel)等， 但是能和Selector机制一起使用的是SocketChannel，主要是其继承了SelectableChannel。

***

##Selector机制
多路复用器Selector，是java NIO编程的基础，其名称估计是来自于Unix／Linux的select()，但实际底层代码采用的是epoll().一个网络读写Channel会在Selector上注册其所感兴趣的事件。
目前Selector所支持的事件有：
1.SelectionKey.OP_ACCEPT  接受一个连接
2.SelectionKey.OP_CONNECT 发起一个连接
3.SelectionKey.OP_READ    有读事件
4.SelectionKey.OP_WRITE  有写事件
从字面意思可以看出，Selector主要关注的还是网络读写。

Selector 会不断地轮询注册在其上的所有Channel，如果某一个Channel上有新的事件，则此Channel会处于就绪状态，被Selector轮询出来，交由用户进程来处理。一个多路复用器可以同时注册多个Channel，并且对他们进行轮询。

Selector可以同时作用于网络的服务端和客户端。

服务端的代码框架如下：

        Selector selector = Selector.open();
        ServerSocketChannel serverSocketChannel=ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.socket().bind(new InetSocketAddress(1234));
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        for(; ;) {
            selector.select();
            Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
            while(keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                if(key.isValid()) {
                    if(key.isAcceptable()) {
                        ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                        SocketChannel socketChannel = channel.accept();
                        socketChannel.configureBlocking(false);
                        socketChannel.register(selector, SelectionKey.OP_READ);//注册读事件
                    }
                    if(key.isReadable()) {
                        SocketChannel channel = (SocketChannel) key.channel();
                        //do read from channel
                    }
                    if(key.isWritable()) {
                        SocketChannel channel = (SocketChannel) key.channel();
                        //do write to channel
                    }
                }
                keyIterator.remove();
            }
        }

其中两个核心为：
* selector.select()
* selector.selectedKeys().iterator();

###selector.select()
java API中 selector的select方法有三个：
1.selector.select() 此方法会阻塞，直到有 channel事件发生
2.selector.select(long timeout) 此方法会阻塞直到有channel事件发生，或是超时timeout
3.selector.selectNow() 此方法不会阻塞，会立即返回，根据其返回值可知道有几个channel事件发生，如果返回值为0，则此次为空轮询，可进行重新进行轮询操作。

三个方法中 selector.select(long timeout)和selector.selectNow() 都可属于 非阻塞方法，也就可能造成 空轮询，造成资源的浪费，Netty才处理空轮询的时候 采用了 rebuildSelector的方法，这个暂没研究过，不说也罢。

###selector.selectedKeys().iterator();
在知道selector上有channel就绪之后，可以采用此方法获取到就绪的channel，根据其就绪的事件做相应的处理。


客户端的代码框架如下：

        Selector selector = Selector.open();
        SocketChannel channel = SocketChannel.open();
        channel.configureBlocking(false);
        channel.connect(new InetSocketAddress("127.0.0.1", 1234));
        channel.register(selector, SelectionKey.OP_CONNECT);
        for(;;) {
            selector.select();
            Iterator ite = selector.selectedKeys().iterator();
            while (ite.hasNext()) {
                SelectionKey key = (SelectionKey) ite.next();
                ite.remove();
                if (key.isConnectable()) {
                    if (channel.isConnectionPending()) {
                        if (channel.finishConnect()) {
                            //只有当连接成功后才能注册OP_READ事件
                            key.interestOps(SelectionKey.OP_READ);
                        } else {
                            key.cancel();
                        }
                    }
                }
                if (key.isReadable()) {
                    //do read from channel
                }
                if(key.isWritable()) {
                    //do write to channel
                }
            }
        }

其中大部分的代码和服务端的差不多，这里不再赘述，有一个点需要注意一下的事：

    channel.configureBlocking(false);
    channel.connect(new InetSocketAddress("127.0.0.1", 1234));

如果不设置channel.configureBlocking(false)的话，connect本身为阻塞方法， 直到连接服务端成功，或是超时。
如果设置了channel.configureBlocking(true)的话，connect则会直接返回，在进行select的时候，有channel事件就绪，再判断channel.finishConnect()完成与服务端的连接。


java的NIO 是Netty的底层基础，说的直白一点，netty是对Selector的封装，所以如果对Netty源码进行调试的话，到了真正网络的层次，都是java NIO 的代码。


##java Future
Future是一个接口，该接口用来返回异步的结果.
以前对于Future我一直不是很理解，但最近看了Netty的源码之后，大概对它有了一些概念。
Future表示的是从调用方法获取结果的一种凭据：所谓返回异步结果，我的理解是：在调用某一方法时，由于该方法比较耗时，如果长期阻塞在那里的话不太好，所以会在调用方法的内部会封装一个future对象直接返回给方法调用者。调用者在拿到future对象之后可以不去处理它，先处理其他事情，等真正需要用到调用结果的时候，在通过future.get()方法去获取，由于方法调用到获取结果之间有一段时间差，在这期间调用的结果可能已经出来，所以通过future.get()方法可以获取到结果。
future的get方法其实还是一个阻塞方法，只是他不回阻塞在方法调用的地方，而是在你需要调用结果的地方从凭据(future)中获取。
Netty扩展了java 的future类，添加了 监听器的机制，这样，甚至不需要调用 future.get()方法，只要注册对应的监听器到netty的future模块，在相关的调用完成之后，future模块调用已经注册的监听器，即可处理对应的业务逻辑。

##后续
本文主要介绍了Netty中可能用到的一些基础内容，下面会通过分析Netty example中echo来走进Netty的源码世界。














