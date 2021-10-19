---
layout: post
title: ReentrantLock
categories: 并发
keywords: ReentrantLock, JUC
---

### JUC锁：ReentrantLock

下面我们结合`ReentrantLock`看一下AQS中`tryAcquire`和`tryReleas`是怎么交给子类实现的。`ReentrantLock`中定义了内部类`Sync`来实现同步锁的效果，`Sync`类继承自AQS实现了一部分方法，而`FairSync`和`NonfairSync`作为公平和非公平锁的实现，各自定义了`tryAcquire`方法，我们先来看一下整个类的继承层次：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/ReentrantLock-Arch.png" alt="ReentrantLock-Arch" style="zoom:50%;" />

然后我们看一下Sync类：

#### Sync

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    // 默认提供了非公平锁的实现，但是非公平类需要重写tryAcquire方法来使用这个方法
    // 非公平性就体现在获取资源时，无论阻塞队列是否存在都尝试直接争抢资源
    // 只有争抢不到时才会进入阻塞队列阻塞
    @ReservedStackAccess
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // c为0说明当前资源没人使用
        if (c == 0) {
            // 尝试CAS争抢，如果争抢成功就将当前线程设置为资源的拥有线程
            // 返回true，到AQS的acquire函数处，就表明资源已经获取，无需入队等待
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 否则这里要考虑资源是否被当前线程重入, 即当前线程重复获取资源
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            // 资源为负数说明溢出
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 否则资源获取失败，返回false，由acquire负责入队阻塞
        return false;
    }

    @ReservedStackAccess
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        // 如果当前线程不是资源的使用者，不允许释放
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            // 如果当前线程持有的资源全部被释放，那么要清空持有线程标记
            // 并且向release函数返回true，表示资源完全被释放
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class

    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

可以看到`Sync`类是抽象类，它实现了非公平加锁和释放的方法，但是为了让子类可以利用多态特性，这里并没有使用`tryAcquire`的名字，而是使用了`nonfairTryAcquire`，子类需要使用时是需要重新实现`tryAcquire`方法，然后调用已经定义好的非公平加锁方法的。

#### NonfairSync

`ReentrantLock`中的非公平锁实现就是这样的：

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

简单的调用了`Sync`类的非公平加锁实现即可。接下来我们来看公平锁的实现：

#### FairSync

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    @ReservedStackAccess
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 唯一的不同就在这里，当资源可用时会先判断是否存在等待队列
            // 如果不存在等待队列才会尝试CAS获取资源，否则直接返回false
            // 由acquire方法负责加入队列
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 处理可重入锁
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

可以看到公平锁的实现和公平锁几乎一模一样，只是公平锁多了判断是否存在排队的结点这一条语句，其他的一模一样。我们来看一下这个判断函数`hasQueuedPredecessors`，它是在AQS中实现的：

```java
public final boolean hasQueuedPredecessors() {
    Node h, s;
    // 判断头结点是否存在
    if ((h = head) != null) {
        // 让s成为第一个有效的结点，包括s本身不为空，且状态不为CANCELLED
        // 否则需要从队尾向队头寻找一个有效的结点
        if ((s = h.next) == null || s.waitStatus > 0) {
            s = null; // traverse in case of concurrent cancellation
            for (Node p = tail; p != h && p != null; p = p.prev) {
                if (p.waitStatus <= 0)
                    s = p;
            }
        }
        // 如果找到这样的结点，即s不为空，且s也不是当前线程的结点
        // 那么说明存在等待队列，公平锁不能直接获取资源
        if (s != null && s.thread != Thread.currentThread())
            return true;
    }
    // 否则说明队列为空，或者队列里都是被取消的结点
    return false;
}
```

很简单，就是寻找是否存在排队的结点，包括队列非空和结点没有被取消两个判断。

以上就是ReentrantLock的实现了，ReentrantLock是一个典型的独占模式的AQS实现，通过重写`tryAcquire`和`tryRelease`的方法实现了公平和非公平加锁的特点，接下来我们看一个共享锁的实现。

### JUC锁：ReentrantReadWriteLock

`ReentrantReadWriteLock`是同时实现了AQS `tryAcquire`、`tryRelease`、`tryAcquireShared`、`tryReleaseShared`的一个既支持共享锁，又支持独占锁的类，其类定义如下：

```java
public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable {}
```

发现它实现了`ReadWriteLock`接口，`ReadWriteLock`接口定义了获取读锁和写锁的规范，具体需要实现类去实现；同时其还实现了`Serializable`接口，表示可以进行序列化，在源代码中可以看到ReentrantReadWriteLock实现了自己的序列化逻辑。

这个类的内部结构和`ReentrantLock`相似，都是自定义了公平锁和非公平锁的实现，但是除了锁，它还定义了读锁和写锁的内部类，类图如下：

![RWLock-InncerClass-Arch](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/RWLock-InncerClass-Arch.png)

下面我们还是先从Sync类开始分析：

#### Sync

`Sync`类定义了从AQS获取资源和释放资源的一系列方法，但是这里比较复杂，因为ReadWriteLock同时支持独占和共享，所以`Sync`类也就要同时考虑读锁和写锁两方面的内容。

Sync内部定义了两个内部类来支持读锁的重入计数：

```java
static final class HoldCounter {
    int count;          // initially 0
    // Use id, not reference, to avoid garbage retention
    final long tid = LockSupport.getThreadId(Thread.currentThread());
}

static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
            return new HoldCounter();
    }
}
```

##### 属性

类的属性包括以下：

```java
// 读锁是高16位，写锁是低16位
static final int SHARED_SHIFT   = 16;
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
// 本地线程计数器，用于记录重入的读锁个数
private transient ThreadLocalHoldCounter readHolds;
// 缓存的计数器
private transient HoldCounter cachedHoldCounter;
// 第一个读线程
private transient Thread firstReader = null;
// 第一个读线程的计数
private transient int firstReaderHoldCount;

/** Returns the number of shared holds represented in count. */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count. */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

##### 构造函数

构造函数如下，只是简单的进行初始化：

```java
Sync() {
    readHolds = new ThreadLocalHoldCounter();
    setState(getState()); // ensures visibility of readHolds
}
```

接下来我们看资源获取的过程：

##### 写锁获取：tryAcquire

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    // c是状态的总数
    int c = getState();
    // w是由state计算出的写锁的个数（低16位）
    int w = exclusiveCount(c);
    if (c != 0) {
        // c不等于0表示至少有读锁或写锁存在
        // 如果此时w=0，说明存在读锁，那么不能获取写锁
        // 否则说明当前存在写锁，那么判断是否可重入，如果不可重入即不是写锁的拥有者
        // 那么也不能获得写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 溢出
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 否则说明可重入，那么获取可重入锁即可
        setState(c + acquires);
        return true;
    }
    // 否则说明没有资源，这需要交给子类判断一下是否需要阻塞
    // 也就是公平和非公平的判断，看看是不是能直接抢资源
    // 如果是，但是CAS失败，那么会返回到AQS的acquire方法中入队处理
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 否则说明获得资源成功，设置资源所有者，返回true
    setExclusiveOwnerThread(current);
    return true;
}
```

这个逻辑很简单，写锁是独占锁，要获取它的话不能存在任何共享锁，同时也可以重入获取。这里的`writerShouldBlock`是由Sync的子类`FairSync`和`NonfairSync`实现的，目的就是为了保证公平性，我们看一下这两个类实现的源码：

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}

static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }
}
```

很明显，公平锁无论获取读锁还是写锁都需要判断是否存在队列，而非公平锁对于写锁的获取则可以直接抢锁，而对于读锁的获取，则需要进一步判断，当读锁和写锁同时等待时，为了避免写锁饿死，读锁不一定能直接抢占。

整个写锁获取的流程图如下：

![RWLock-tryAcquire-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/RWLock-tryAcquire-Process.png)

##### 写锁释放：tryRelease

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

独占锁的释放没有什么好说的，就是直接释放资源，注意一下是不是重入资源都被释放即可，很简单。流程图如下：

![RWLock-tryRelease-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/RWLock-tryRelease-Process.png)

##### 读锁获取：tryAcquireShared

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 如果当前存在写锁，并且写锁的持有者不是自己的时候，返回-1，获取失败
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    // 获取读锁的计数
    int r = sharedCount(c);
    // 同样判断，如果读锁可以直接抢，并且读锁的总数小于最大值(0xffff)
    // 同时CAS没有竞争，设值成功（只能加1，高16为加1）
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 如果此前读锁为0，说明之前没有占用读锁的线程
        // 当前线程是第一个获取读锁的，所以设置为firstReader
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 如果此前存在读锁，但是此前的读锁也是当前线程获取的
            // 那么增加重入计数即可
            firstReaderHoldCount++;
        } else {
            // 否则说明此前存在读锁且不是当前线程，那么首先获取缓存的Counter
            // 这样做的目的是减少通过ThreadLocal取值的开销
            // 如果上一个获取读锁的就是当前线程，那么直接操作cachedHoldCounter加一即可
            // 否则需要重新读取缓存
            HoldCounter rh = cachedHoldCounter;
            if (rh == null ||
                rh.tid != LockSupport.getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        // 至此读锁获取成功，返回1
        return 1;
    }
    // 否则读锁获取失败，可能需要排队
    return fullTryAcquireShared(current);
}
```

读锁也就是共享锁的获取也是比较简单的，因为复杂的列队部分都在AQS的`acquireQueued`中处理好了，这里只需要对读锁的重入个数进行计数即可，对于`ReentrantReadWriteLock`来说，只能一次获得1个读锁。这里利用了线程独有和缓存两个值来记录，目的就是提高由于`ThreadLocal`插入内存屏障可能带来的性能损失。

当读锁需要排队，或者CAS出现竞争时，会交给`fullTryAcquireShared`方法进行执行：

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        // 判断写锁是否存在，如果存在且不为当前线程，那么直接失败
        // 否则说明就是写锁线程调用了这个方法，是可以操作读锁的
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
        } else if (readerShouldBlock()) {
            // 如果读锁需要阻塞，首先判断是不是已经获得读锁
            // 防止重入时线程被阻塞导致锁可能不会被放弃
            // 如果是第一个获得读锁的情况下，直接退出，不需要阻塞
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                // 否则需要获取重新获取读锁计数器，判断读锁是否存在
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null ||
                        rh.tid != LockSupport.getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                // 如果没有读锁，说明确实需要阻塞，返回-1，交给AQS的acquireShared
                // 进行排队阻塞
                if (rh.count == 0)
                    return -1;
            }
        }
        // 如果是因为读锁超出最大值进入这里的，直接抛出异常
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 否则是因为CAS失败进入的，需要不断CAS尝试，后续和tryAcquireShared一样
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null ||
                    rh.tid != LockSupport.getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

这个方法和`tryAcquireShared`很相似，目的就是为了处理`tryAcquireShared` if语句失败后的三种情况。

总而言之，获取读锁的流程图如下：

![RWLock-tryAcquireShared-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/RWLock-tryAcquireShared-Process.png)

最后就是读锁的释放过程了：

##### 读锁释放：tryReleaseShared

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // 如果当前线程是第一个线程，那么需要清空这个标记
    if (firstReader == current) {
        // 如果读锁持有量为1，说明这是最后一个读锁，要情况firstReader
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            // 否则说明是重入的读锁，直接递减即可
            firstReaderHoldCount--;
    } else {
        // 如果不是第一个读锁，那么一样先查询cachedHoldCounter
        // 如果不是则获取ThreadLocalCounter
        // 将获取到的HoldCounter减1即可，如果已经没有读锁了，那么就溢出这个HoldCounter
        HoldCounter rh = cachedHoldCounter;
        if (rh == null ||
            rh.tid != LockSupport.getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    // 最后CAS自旋减少state总量即可
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

流程很简单，就是需要更新资源和读锁重入数两个，这里不再赘述，直接看流程图：

![RWLock-tryReleaseShared-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/RWLock-tryReleaseShared-Process.png)

好了以上就是Sync类的整个流程了，因为`ReentrantReadWriteLock`由Sync类实现了所有获取和释放的过程，所以它的FairSync和NonfairSync类的功能只在于判断是否能够立即抢锁上（即`writerShouldBlock`和`readerShouldBlock`）两个函数，主体功能都由Sync实现好了。而ReadLock和WriterLock也只是调用了Sync类的方法来实现的，这里就不再详细说了。