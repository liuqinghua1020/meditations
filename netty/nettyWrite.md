#Netty写

本文主要分析Netty写的流程，以客户端写为例子。

EchoClient并没有用户线程(main)主动写的流程，这里以SecureChatClient的代码为例，写的核心代码主要是以下两句

    lastWriteFuture = channel.write(line + "\r\n");
    if (lastWriteFuture != null) {
       lastWriteFuture.sync();
    }

下面一句句来分析。

##channel.write

channel.write的实现是在AbstractChannel中，其代码如下

    Channels.write(this, message);

该方法会返回一个future（future相关的内容，请参考[Netty基础杂谈](./nettyBase.md)），用于给用户线程一个凭据，确定写操作完成。

Channels的write方法代码如下：
    
    ChannelFuture future = future(channel);
    channel.getPipeline().sendDownstream(
            new DownstreamMessageEvent(channel, future, message, remoteAddress));
    return future;

核心逻辑是调用该channel所依附的pipeline的sendDownstream方法，直到最后到达NioClientSocketPipelineSink的eventSunk方法的如下分支（DownstreamMessageEvent继承自MessageEvent）

    else if (e instanceof MessageEvent) {
        MessageEvent event = (MessageEvent) e;
        NioSocketChannel channel = (NioSocketChannel) event.getChannel();
        boolean offered = channel.writeBufferQueue.offer(event);
        channel.worker.writeFromUserCode(channel);
    }

主要代码是

    boolean offered = channel.writeBufferQueue.offer(event);
    channel.worker.writeFromUserCode(channel);

将event投递到channel写队列中，然后在Worker线程中执行真正网络写操作，writeFromUserCode的核心代码如下
    
    write0(channel);

那在看下write0的代码逻辑

         synchronized (channel.writeLock) {
            channel.inWriteNowLoop = true;
            for (;;) {
                MessageEvent evt = channel.currentWriteEvent;
                SendBuffer buf = null;
                ChannelFuture future = null;
                if (evt == null) {
                    if ((channel.currentWriteEvent = evt = writeBuffer.poll()) == null) {
                        removeOpWrite = true;
                        channel.writeSuspended = false;
                        break;
                    }
                    future = evt.getFuture();
                    channel.currentWriteBuffer = buf = sendBufferPool.acquire(evt.getMessage());
                } else {
                    future = evt.getFuture();
                    buf = channel.currentWriteBuffer;
                }
                long localWrittenBytes = 0;
                for (int i = writeSpinCount; i > 0; i --) {
                    localWrittenBytes = buf.transferTo(ch);
                    if (localWrittenBytes != 0) {
                        writtenBytes += localWrittenBytes;
                        break;
                    }
                    if (buf.finished()) {
                        break;
                    }
                }
                if (buf.finished()) {
                    buf.release();
                    channel.currentWriteEvent = null;
                    channel.currentWriteBuffer = null;
                    evt = null;
                    buf = null;
                    future.setSuccess();
                } else {
                    addOpWrite = true;
                    channel.writeSuspended = true;
                    if (writtenBytes > 0) {
                        future.setProgress(
                                localWrittenBytes,
                                buf.writtenBytes(), buf.totalBytes());
                    }
                    break;
                }
            }
            channel.inWriteNowLoop = false;
            if (open) {
                if (addOpWrite) {
                    setOpWrite(channel);
                } else if (removeOpWrite) {
                    clearOpWrite(channel);
                }
            }
        }
        if (causes != null) {
            for (Throwable cause: causes) {
                // notify about cause now as it was triggered in the write loop
                fireExceptionCaught(channel, cause);
            }
        }
        if (!open) {
            close(channel, succeededFuture(channel));
        }
        if (iothread) {
            fireWriteComplete(channel, writtenBytes);
        } else {
            fireWriteCompleteLater(channel, writtenBytes);
        }

逻辑比较复杂，暂时还没去详细分析，但是可以看到这样几句代码：

    channel.currentWriteEvent = evt = writeBuffer.poll()

这一句对应的是NioClientSocketPipelineSink的eventSunk中的这一句
    
    boolean offered = channel.writeBufferQueue.offer(event);

    localWrittenBytes = buf.transferTo(ch);
这一句是write0方法中真正往java NIO Channel 写数据的方法。

最后，如果写完成且没有问题的话，会调用一下方法：
    
     if (iothread) {
        fireWriteComplete(channel, writtenBytes);
     } else {
        fireWriteCompleteLater(channel, writtenBytes);
     }

如果当前本身是Worker线程，则调用fireWriteComplete方法，直接在本线程中通知写操作完成事件。否则封装成线程任务，交由Wokrer线程来处理（fireWriteCompleteLater方法）。


至此，channel.write逻辑基本分析完毕。

##lastWriteFuture.sync

sync主要分析的是DefaultChannelFuture实现的sync方法，代码如下

    await();
    rethrowIfFailed0();

await方法是真正的阻塞方法，其内部

     if (Thread.interrupted()) {
        throw new InterruptedException();
     }
     synchronized (this) {
        while (!done) {
            checkDeadLock();
            waiters++;
            try {
                wait();
            } finally {
                waiters--;
            }
        }
     }

核心是调用了Object的wait方法，真正阻塞在这里，他需要等到别的地方，例如Worker线程中channel带着的future调用setSuccess或是setFailure方法唤醒才行，setSuccess方法如下

    synchronized (this) {
        if (done) {
            return false;
        }
        done = true;
        if (waiters > 0) {
            notifyAll();
        }
    }
    notifyListeners();

从代码可以看出，处理调用Object的notifyAll方法之外，还调用notifyListeners通知所有注册在此future中的listener，告知future 已经完成工程。

sync中的rethrowIfFailed0方法是处理失败的，这里不再分析。

至此，Netty的写操作分析完毕。
