---
layout: post
title: FutureTask
categories: 并发
keywords: FutureTask, JUC
---

### JUC线程池：FutureTask

FutureTask个人感觉就是用来包装Runnable或者Callable对象的，用于监测线程状态、获取异常、控制运行等。其继承结构如下：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/FutureTask-Arch.png" alt="FutureTask-Arch" style="zoom:50%;" />

可以从上图看出，`FutureTask`类本身实现了`Runnable`接口，同时也持有一个`Runnable`对象，类似一个适配器，同时被`FuturenTask`包装的类也可以直接被线程池执行。

#### 源码解析

##### 常量

```java
public class FutureTask<V> implements RunnableFuture<V> {
    // 当前任务执行状态
    private volatile int state;
    // 新建状态，当前类被实例化后到开始运行前就处于这个状态
    private static final int NEW = 0;
    // 过渡状态，表示任务执行完毕，正在做最后的设置
    // 成功时设置为NORMAL，异常时设置为EXCEPTIONAL等
    private static final int COMPLETING = 1;
    // 完成状态，正常结束
    private static final int NORMAL = 2;
    // 完成状态，发生异常
    private static final int EXCEPTIONAL = 3;
    // 完成状态，用户调用了cancle取消了任务但不中断任务
    private static final int CANCELLED = 4;
    // 过渡状态，用户调用了cancle取消了任务并且尝试中断任务
    private static final int INTERRUPTING = 5;
    // 完成状态，任务被中断
    private static final int INTERRUPTED = 6;

    // 包装的Callable对象，运行结束后可能会被设置为null（除非使用reset）
    private Callable<V> callable;
    // 运行结果或异常
    private Object outcome; // non-volatile, protected by state reads/writes
    // 运行该任务的线程
    private volatile Thread runner;
    // 等待任务结果的线程队列（单向链表组织）
    private volatile WaitNode waiters;
}
```

`FutureTask`任务状态的变迁图如下：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/FutureTask-Status-Change.png" alt="FutureTask-Status-Change" style="zoom:75%;" />

上述状态中要注意，CANCELLED状态意味着用户取消了任务，它并不会导致任务抛出异常（但是取消之后再去取结果会导致`CancellationException`异常），而如果中断了线程则在run函数运行期间就会导致异常被设置到outcome中。同时要注意outcome有两个语义：一个是正常运行结束后的运行结果，另一个是异常运行抛出的异常。

##### 构造函数

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

可以看到`FutureTask`类原生使用的是Callable对象来存储，所以对于传入的Callable对象直接赋值即可。而对于传入的Runnable对象实际上将其包装成了一个Callable对象，即利用适配器模式包装，而第二个参数result就是返回值，传入了什么就返回什么，如果不需要返回值可以传入`null`。最后在构造对象时将任务状态设置成了`NEW`来保证其可以被线程正确调用。

**我们从构造函数和FutureTask的继承关系可以看到，FutureTask可以实现将Callable接口的任务传入只支持Runnable接口的函数中，因为FutureTask包装了Callable接口（类似适配器），同时FutureTask又是一个实现Runnable接口的类，所以实现了对Callable接口任务的适配工作。这在ThreadPoolExecutor的execute方法中会用到，如果想传入Callable任务，除了使用submit方法外，也可以使用FutureTask来包装**

##### run

```java
public void run() {
    // 如果任务状态不是新建那么说明当前任务已经结束了
    // 如果CAS将当前线程设置为运行线程失败则说明存在线程竞争，当前任务可能已经被分配
    // 所以出现这两种情况直接返回即可
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        // 再次检查当前Callable对象存在（虽然构造函数不允许Callable为空）
        // 但是运行结束后会将其设置为空来表示任务完成（除了reset运行）
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                // 出现异常
                setException(ex);
            }
            if (ran)
                // 运行结束，设置结果
                set(result);
        }
    } finally {
        // 必须等到所有结果设置完毕，任务状态设置完毕才能清空当前线程标志
        runner = null;
        // 同理中断处理必须等到所有操作都结束了并且设置了状态之后处理
        // 否则可能导致任务被重入导致中断被忽略
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

原理很简单，主要就是使用包装的Callable对象进行运行，在运行之前设置状态，在运行之后设置状态、设置返回值或异常、处理中断，唯一要注意的就是防止任务被多个线程重复执行，要做一些保护措施，主要就是利用当前运行线程`runner`、是否为空的任务`callable`来判断该任务是否需要执行，再利用`state`判断是否执行完成。

##### set

```java
// 中间的set方法用于设置返回值
protected void set(V v) {
  	// COMPLETING状态表示正在设置结果，是一个中间状态，当遇到这种状态时表示设置还未结束，需要等待
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
      	// 设置结果值
        outcome = v;
      	// 设置状态为正常结束
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
      	//执行完毕，唤醒等待线程
        finishCompletion();
    }
}
```

##### finishCompletion

```java
private void finishCompletion() {
    // 当前状态必定>=COMPLETING, 无论是正常结束还是异常结束
    // 遍历所有等待节点，死循环知道没有等待获取结果的线程
    for (WaitNode q; (q = waiters) != null;) {
        // 尝试清除所有等待线程（即链表的第一个节点，因为剩下的节点由内部保证清空）
        if (WAITERS.weakCompareAndSet(this, q, null)) {
            for (;;) {
                // 获取被阻塞的线程
                Thread t = q.thread;
                if (t != null) {
                    // 清空等待线程并且唤醒该线程，置null的原因是消除强引用
                    // 如果这个线程执行完毕可以提前被GC回收，而不用等待整个FutureTask被回收
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                // 如果存在下一个线程，则处理下一个线程，否则退出
                WaitNode next = q.next;
                if (next == null)
                    break;
                // 同理去除下一个节点的强引用，让GC可以回收
                q.next = null;
                q = next;
            }
            break;
        }
    }
    // 模板方法，运行用户自定义
    done();
    // 标志着整个任务执行完毕，清空callable，防止任务被反复执行
    callable = null;
}
```

这个方法的作用是唤醒所有等待结果的线程，并且清空任务防止重复执行。这个方法的关键在两个循环，外层的循环是用来清空等待队列的，防止多个线程同时清空，同时也是为了辅助GC；内层的循环是用来逐个唤醒线程并且清空等待队列的强引用的。

##### handlePossibleCancellationInterrupt

```java
private void handlePossibleCancellationInterrupt(int s) {
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield();
}
```

可以看到这个方法检查到当前任务正在设置中断时，即出现INTERRUPTING状态时，强制让出时间片，等待设置完毕之后才返回

##### get

get方法是一个核心方法，它的作用是获取返回值或返回的异常值，这里不分析带超时机制的get方法，只分析普通的get方法。

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

##### awaitDone

很明显，如果任务还没有执行完毕，那么`get`方法会等待，但是等待的方法会根据state的不同而发生改变：

```java
private int awaitDone(boolean timed, long nanos) throws InterruptedException {
        long startTime = 0L;    // Special value 0L means not yet parked
        // 如果需要阻塞等待则创建等待节点q
        WaitNode q = null;
        // 标识是否成功进入阻塞队列
        boolean queued = false;
        for (;;) {
            int s = state;
            // 如果任务运行完成，那么直接返回状态即可
            // 如果当前任务运行完毕，那么清空node的运行线程用来辅助GC
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            // 如果任务正处于设值状态，那么让出CPU时间片等待设值
            else if (s == COMPLETING)
                // We may have already promised (via isDone) that we are done
                // so never return empty-handed or throw InterruptedException
                Thread.yield();
            // 如果当前等待线程被中断，那么将当前等待线程移除阻塞队列并且抛出中断异常
            // 这个中断指的是调用get函数的线程被中断而不是执行FutureTask的线程被中断
            else if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }
            // 否则说明当前获取的值还在运行中，可以将线程挂到阻塞队列中
            else if (q == null) {
                // 如果使用了超时机制并且设置的时间小于0说明不允许等待，直接返回
                if (timed && nanos <= 0L)
                    return s;
                // 否则创建阻塞队列节点
                q = new WaitNode();
            }
            // 到这里说明阻塞队列节点存在，但是还没有加入阻塞队列，所以这个if尝试将当前节点入队
            else if (!queued)
                // 利用CAS将队列头结点赋值给当前节点，将队列头结点设置为当前节点
                // 这里可能失败的原因就是有多个线程在尝试get，出现了头结点插入的竞争
                queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q);
            // 如果插入成功，这里要考虑超时问题（如果有）
            else if (timed) {
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING)
                    LockSupport.parkNanos(this, parkNanos);
            }
            // 否则，调用无超时的阻塞
            else
                LockSupport.park(this);
        }
    }
```

当调用get函数时，会根据当前任务的状态不同来决定怎么做。对于即将完成的状态（COMPLETING），会让出时间片等待；对于完成的方法直接返回状态（用来区分是正常完成还是异常完成）；对于线程中断的方法，则移出等待队列；否则则将当前节点CAS入队，并挂起当前线程等待。当任务执行完毕后会通过`finishCompletion`方法唤醒当前线程。此时上面的方法重新循环后必定进入第一个if语句，从而将状态返回。

这里稍微解释一下为什么`awaitDone`函数和`finishCompletion`函数都存在将thread置为null的操作。这两个操作不矛盾，因为并不是所有线程都有机会被`finishCompletion`执行，考虑一种高并发情况，不断有线程加入阻塞队列，而某一个线程多次CAS失败会一直在for循环里自旋，如果自旋过程中任务结束了，也就是满足第一个if了，那么此时的WaitNode也就是q对象还没有加入到阻塞队列中并且由于任务结束了也不可能再加入到阻塞队列里了，这时候就需要手动清除这个q中的thread，防止q对thread的强引用导致整个FutureTask都无法被GC。**总而言之就是如果在for循环的过程中任务结束了，那么finishCompletion方法就不会清除q这个游离的WaitNode节点。**

##### report

返回后的方法由`report`进行处理，对于正常返回那么就将outcome作为返回值，对于取消返回那么抛出`CancellationException`异常，对于运行出错那么抛出`ExecutionException`

```java
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

##### removeWaiter

这个方法用于移除处于等待队列中的节点，唯一要注意一下的就是这个方法的没有用常见的q == node来判断，而是直接判断q.thread != null。因为这种判断更具有普遍性，如果自旋过程中发现了其它可以被清理的节点，那么这个函数也可以实现清理别的节点的工作。

```java
private void removeWaiter(WaitNode node) {
    if (node != null) {
        // 清空线程，防止强引用
        node.thread = null;
        retry:
        for (;;) {          // restart on removeWaiter race
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                // 这里就是单向链表的删除，pred指向前一个节点，q指向当前节点，s指向下一个节点
                // 这里判断是不是这个node是通过thread是否为空来判断的
                // 因为前面将node的thread置空，而正常阻塞队列中的thread不可能为空，所以只需要判断
                // thread是否为空就可以判断出是不是node节点了
                s = q.next;
                // thread非空，肯定不是，继续下一个节点
                if (q.thread != null)
                    pred = q;
                // thread为空说明就是这个节点，如果pred非空则说明有前驱节点，将pred和next连接即可
                else if (pred != null) {
                    pred.next = s;
                    // 如果此时pred的thread为空说明存在多个线程竞争，需要重新扫描一遍
                    if (pred.thread == null)
                        continue retry;
                }
                // 如果是这个节点且前驱为空说明是头结点，同样CAS设置头结点，如果失败则说明有竞争
                // 原来的头结点可能已经不是头结点了（有新的线程加入阻塞队列），所以重新扫描
                else if (!WAITERS.compareAndSet(this, q, s))
                    continue retry;
            }
            break;
        }
    }
}
```

##### cancel

最后一个方法就是cancel方法了：

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 如果CAS设置失败或不处于新建或运行状态，那么直接返回
    // CAS失败就说明已经有其他线程设置了它的属性了，这里并不覆盖这个新属性
    if (!(state == NEW && STATE.compareAndSet
          (this, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {
        // 如果需要中断线程，那么获取运行线程并中断
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { 
                // 中断后设置为中断状态，否则就是上一个if设置的取消状态
                STATE.setRelease(this, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```

cancel方法很简单，如果参数为true说明需要中断方法执行，否则只取消方法执行并不一定中断。cancel结束后同样要唤醒所有等待取值的线程，这些线程因为当前任务状态为CANCELLED或INTERRUPTED而收到异常。