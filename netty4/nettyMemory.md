# netty内存管理

netty的内存管理是比较复杂的一个模块，需要慢慢分析才行，首先看一下 netty 内存分类的三个纬度：

1. Pooled/UnPooled 池化和非池化设计
2. byte[]/DirectByteBuffer  堆内存和直接内存
3. Unsafe/普通  采用Unsafe无锁化 和 普通处理方式


这里重点分析一下池化处理，以 PooledByteBufAllocator 为入口进行分析

在其中分别包含了两个重要属性

```java
    /**
     * Heap PoolArena 数组
     */
    private final PoolArena<byte[]>[] heapArenas;
    /**
     * Direct PoolArena 数组
     */
    private final PoolArena<ByteBuffer>[] directArenas;

```

主要关注一下 其 newHeapBuffer 方法吧

```java
    /**
     * 获得线程的 PoolThreadCache 对象
       从线程获取对应缓存的cache，避免加锁的处理
     */ 
    PoolThreadCache cache = threadCache.get();
    PoolArena<byte[]> heapArena = cache.heapArena;

    
    if (heapArena != null) {//线程缓存中有，直接从其中分配内存
            buf = heapArena.allocate(cache, initialCapacity, maxCapacity);
    // 直接创建 Heap ByteBuf 对象，基于非池化
    } else { 
        // 直接创建 Heap ByteBuf 对象，基于非池化
        buf = PlatformDependent.hasUnsafe() ?
                new UnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
                new UnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
    }
```

为了方便，主要看一下 heapArena.allocate 的方法

heapArena是一整个分配算法的核心，其核心原理是 jmalloc 算法，

```java

    PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
        // 创建 PooledByteBuf 对象
        PooledByteBuf<T> buf = newByteBuf(maxCapacity);
        // 分配内存块给 PooledByteBuf 对象
        allocate(cache, buf, reqCapacity);
        return buf;
    }
```

其分为两个步骤：
1. 创建 PooledByteBuf 对象，注意，测试并没有真正分配内存给它
2. 分配内存块给 PooledByteBuf 对象，这个时候，才是真正把 内存分配给 它的时候。

下面分别看看两个过程

```java
    @Override
    protected PooledByteBuf<byte[]> newByteBuf(int maxCapacity) {
        return HAS_UNSAFE ? PooledUnsafeHeapByteBuf.newUnsafeInstance(maxCapacity)
                : PooledHeapByteBuf.newInstance(maxCapacity);
    }
    
    // 以 PooledHeapByteBuf 为例子
    static PooledHeapByteBuf newInstance(int maxCapacity) {
        // 从 Recycler 的对象池中获得 PooledHeapByteBuf 对象
        PooledHeapByteBuf buf = RECYCLER.get();
        // 重置 PooledDirectByteBuf 的属性
        buf.reuse(maxCapacity);
        return buf;
    }

    final void reuse(int maxCapacity) {
        // 设置最大容量
        maxCapacity(maxCapacity);
        // 设置引用数量为 0
        setRefCnt(1);
        // 重置读写索引为 0
        setIndex0(0, 0);
        // 重置读写标记位为 0
        discardMarks();
    }
```

这个并不难理解，从RECYCLER中重用一个 PooledHeapByteBuf 对象（避免频繁的new出来），设置其一些属性，例如读写指针。

接下来才是核心的 allocate 方法

```java
     // 标准化请求分配的容量, 即 3（reqCapacity） --> 4（normCapacity）； 7 --> 8
    final int normCapacity = normalizeCapacity(reqCapacity);
```

如果是分配小内存的化，是如下内容

```java
          // capacity < pageSize(默认的pageSize是 8k)
          if (isTinyOrSmall(normCapacity)) { 
            int tableIdx;
            PoolSubpage<T>[] table;
            // 判断是否为 tiny 类型的内存块申请
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512 tiny 类型的内存块申请
                // 从 PoolThreadCache 缓存中，分配 tiny 内存块，并初始化到 PooledByteBuf 中。
                //PoolThreadCache 其实是在线程内部保存了一系列可服用的 chunk，具体可查看PoolThreadCache
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    return;
                }
                // 获得 tableIdx 和 table 属性
                /**
                 * 这里的意思是这样的：
                    1. tiny 是把一个chunk分成同等大小数组的小内存
                    2. 例如 ，每一个是 1k， 取3k的化，是获取到 数组第三个元素 即 table[tableIdx]
                 */
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {
                // 从 PoolThreadCache 缓存中，分配 small 内存块，并初始化到 PooledByteBuf 中。
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                // 获得 tableIdx 和 table 属性
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            // 获得 PoolSubpage 链表的头节点
            final PoolSubpage<T> head = table[tableIdx];

            // 从 PoolSubpage 链表中，分配 Subpage 内存块
            synchronized (head) { // 同步 head ，避免并发问题
                final PoolSubpage<T> s = head.next;
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    // 分配 Subpage 内存块
                    //核心，handle 其实是一个位置信息，指在此 Subpage 中哪一个数字的哪一块内容里面
                    long handle = s.allocate();
                    assert handle >= 0;
                    // 初始化 Subpage 内存块到 PooledByteBuf 对象中
                    //这里其实就是根据 handle的内容在 chunk中分配的起止位置
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                    // 增加 allocationsTiny 或 allocationsSmall 计数
                    incTinySmallAllocation(tiny);
                    // 返回，因为已经分配成功
                    return;
                }
            }
            // 申请 Normal Page 内存块。实际上，只占用其中一块 Subpage 内存块。
            synchronized (this) { // 同步 arena ，避免并发问题
                allocateNormal(buf, reqCapacity, normCapacity);
            }
            // 增加 allocationsTiny 或 allocationsSmall 计数
            incTinySmallAllocation(tiny);
            // 返回，因为已经分配成功
            return;
        }
```

如果是分配大内存的话，则是如下逻辑：

```java
    if (normCapacity <= chunkSize) {
            // 从 PoolThreadCache 缓存中，分配 normal 内存块，并初始化到 PooledByteBuf 中。
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            // 申请 Normal Page 内存块
            synchronized (this) { // 同步 arena ，避免并发问题
                allocateNormal(buf, reqCapacity, normCapacity);
                // 增加 allocationsNormal
                ++allocationsNormal;
            }
        } else {
            // 申请 Huge Page 内存块
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity);
        }
```

处理逻辑都比较复杂，下面慢慢来分析，这里首先清楚一个概念：
 1. 所有的内存都是在chunk下的，可以看看 PoolChunk的定义：
    ```java
        PoolChunk<T>{
            /**
            * 所属 Arena 对象
            */
            final PoolArena<T> arena;
            /**
            * 内存空间。
            *
            * @see PooledByteBuf#memory
            */
            final T memory;
        }
     ```
    其分为两个类型 PoolChunk<byte[]> 和 PoolChunk<ByteBuffer>,其就是 堆内存 和 直接内存；Netty自定义的ByteBuf内部其实就是包含了 一个PoolChunk,和一个对这块内存的读写指针，包含 readIndex,writeIndex, 容量等指针，限定了这个ByteBuf 在 这个 PoolChunk 上的读写范围。
2. 每次分配一个PoolChunk，PoolArea中包含了多个 PoolChunkList，然后PoolChunkList中包含了PoolChunk，PoolChunk中才包含了 PageSize(8k), 一个PoolChunk为 16M。
3. 为了快速定位 应该获取哪一个 PoolChunk，或是哪几个PoolChunk，或者是 哪一个PoolCunk中的哪一个PoolSubpage；Netty采用了一些结构，并且使用 long 类型的handle属性，将这些索引值整理成一个对象；并采用一些二叉树结构和位图结构进行了快速定位使用。

了解了以上几点，基本就能大体明白 Netty内存分配的大体逻辑了。

