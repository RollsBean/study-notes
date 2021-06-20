# 第十五章 - ByteBuf和相关辅助类

## 15.1 ByteBuf功能说明

基础类型都有自己的额缓存区。

ByteBuffer 缺点：

1. ByteBuffer 长度固定，一旦分配完成，它的容量不能动态扩展和收缩；
2. ByteBuffer 只有一个标识位置的指针position，读写需要手工调用flip()和rewind()方法，使用不当容易处理失败；
3. API 功能有限；

为了弥补这些不足，Netty自己实现了ByteBuf。

### 15.1.1 ByteBuf的工作原理

ByteBuf 通过两个位置指针来协助缓冲区的读写操作，读操作使用readIndex，写操作使用writeIndex。

ByteBuf 如何动态扩展。通常，对 ByteBuffer 进行put操作时，如果缓冲区不够会抛出异常。为了避免这个问题，通常在put操作时对剩余可用空间进行校验，
如果剩余空间不足，需要重新创建一个ByteBuffer，并将之前的 复制进来，最后释放老的ByteBuffer。


## 15.2 ByteBuf 源码分析

### 15.2.1 ByteBuf 的主要类继承关系

### 15.2.2 AbstractByteBuf 源码分析

leakDetector 被定义成static，意味着所有ByteBuf实例共享一个ResourceLeakDetector对象，它用于检测对象是否泄漏。
```java
static final ResourceLeakDetector<ByteBuf> leakDetector =
            ResourceLeakDetectorFactory.instance().newResourceLeakDetector(ByteBuf.class);
```

#### 1. 主要成员变量

### 15.2.3 AbstractReferenceCountedByteBuf 源码分析

**作用**：该类主要是对引用进行计数，类似于JVM内存回收的对象引用计数器，用于跟踪对象的分配和销毁，做自动内存回收。

#### 1. 成员变量

```java
    private static final long REFCNT_FIELD_OFFSET =
            ReferenceCountUpdater.getUnsafeOffset(AbstractReferenceCountedByteBuf.class, "refCnt");
    private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> AIF_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");

    private static final ReferenceCountUpdater<AbstractReferenceCountedByteBuf> updater =
            new ReferenceCountUpdater<AbstractReferenceCountedByteBuf>() {
        @Override
        protected AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> updater() {
            return AIF_UPDATER;
        }
        @Override
        protected long unsafeOffset() {
            return REFCNT_FIELD_OFFSET;
        }
    };

    // Value might not equal "real" reference count, all access should be via the updater
    @SuppressWarnings("unused")
    private volatile int refCnt = updater.initialValue();
```

`AIF_UPDATER` 是 AtomicIntegerFieldUpdater 类型变量，通过原子方法进行更新等操作，以实现线程安全，消除锁。

volatile修饰的 refCnt 字段用于跟踪对象的引用次数，使用volatile是为了解决多线程并发访问的可见性问题。

### 2. 对象引用计数器

每调用一次retain方法，引用计数器会加一，由于有多线程并发场景，所以累加操作必须是线程安全的。

通过自旋对引用计数器进行jia加一操作，由于计数器初始值为1，所以只有申请和释放能正确使用，它的最小值就为1。

    ### 15.2.4 UnpooledHeapByteBuf 源码分析

**UnpooledHeapByteBuf** 是基于堆内存进行内存分配的字节缓冲区，它没有基于对象池技术实现，这意味着每次I/O都会创建一个新的 UnpooledHeapByteBuf，
虽然进行大内存分配和回收会有性能影响，但是相比堆外内存的申请释放，它的成本更低。

相比于 PooledHeapByteBuf，UnpooledHeapByteBuf 的实现原理简单，也不容易出现内存管理方面的问题，在满足性能的情况下，推荐使用UnpooledHeapByteBuf。

UnpooledHeapByteBuf 代码实现：

#### 1. 成员变量

```java
private final ByteBufAllocator alloc;
byte[] array;
private ByteBuffer tmpNioBuf;
```

ByteBufAllocator 用于内存分配， tmpNioBuf 实现了 Netty ByteBuf 和JDK NIO ByteBuffer 的转换。

#### 2. 动态扩展缓冲区

如果新的容量大于当前的缓冲区容量，需要进行动态扩展并通过 System.arraycopy 进行内存复制。

```java
@Override
public ByteBuf capacity(int newCapacity) {
    checkNewCapacity(newCapacity);
    byte[] oldArray = array;
    int oldCapacity = oldArray.length;
    if (newCapacity == oldCapacity) {
        return this;
    }

    int bytesToCopy;
    if (newCapacity > oldCapacity) {
        bytesToCopy = oldCapacity;
    } else {
        trimIndicesToCapacity(newCapacity);
        bytesToCopy = newCapacity;
    }
    byte[] newArray = allocateArray(newCapacity);
    System.arraycopy(oldArray, 0, newArray, 0, bytesToCopy);
    setArray(newArray);
    freeArray(oldArray);
    return this;
}
```

### 15.2.5 PooledByteBuf 内存池原理分析

#### 1. PoolArena

**Arena** 本身是指一块区域，在内存管理中，Memory Arena 是指内存中的一大块连续区域。PoolArena 就是 Netty 的内存池实现类。

为了集中管理内存，同时提高内存分配和释放的性能，很多都会预先申请一大块内存，这样就不需要频繁使用系统调用来分配和释放内存了。这种情况下，预先申请的大块
内存就被称为 **Memory Arena**。

不同框架，Memory Arena 的实现也不同，Netty 的 PoolArena 由多个 Chunk 组成，每个 Chunk 又由多个 Page 组成。

```java
abstract class PoolArena<T> extends SizeClasses implements PoolArenaMetric {
    static final boolean HAS_UNSAFE = PlatformDependent.hasUnsafe();

    enum SizeClass {
        Small,
        Normal
    }

    final PooledByteBufAllocator parent;

    final int numSmallSubpagePools;
    final int directMemoryCacheAlignment;
    private final PoolSubpage<T>[] smallSubpagePools;

    private final PoolChunkList<T> q050;
    private final PoolChunkList<T> q025;
    private final PoolChunkList<T> q000;
    private final PoolChunkList<T> qInit;
    private final PoolChunkList<T> q075;
    private final PoolChunkList<T> q100;

    private final List<PoolChunkListMetric> chunkListMetrics;
}
```

#### 2. PoolChunk

Chunk 用来组织和管理多个 Page 的内存分配和释放，在 Netty 中，Chunk 中的 Page 被构建成一颗二叉树。

Page 的大小是4个字节，Chunk 的大小是64个字节（4 x 16）。

每个节点都记录了偏移地址，一个节点被分配出去后，这个节点被标记为已分配。对树的遍历采用深度优先算法，但是从哪个节点遍历是随机的。

![Chunk data arch](../../image/java/Netty权威指南/Chunk%20data%20arch.png)

#### 3. PoolSubpage

每个 Page 被切分为大小相等的多个存储块，存储块的大小由第一次申请的内存大小决定，如果是申请4个字节，则这个 Page 包含两个存储块。

Page 中存储区域的使用状态通过一个 long 数组维护，数组中每个 long 的每一位代表一个块存储区域的占用情况，0表示未占用，1表示已占用。

#### 4. 内存回收策略

都是通过状态位来标识内存是否可用。

### 15.2.6 PooledDirectByteBuf 源码分析

PooledDirectByteBuf 基于内存池实现，与 UnpooledDirectByteBuf 的唯一不同就是缓冲区的分配和销毁策略不同。

