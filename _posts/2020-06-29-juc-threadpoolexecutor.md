---
layout: post
title: ThreadPoolExecutor
categories: 并发
keywords: ThreadPoolExecutor, JUC
---

### JUC线程池：ThreadPoolExecutor

`ThreadPoolExecutor`是一个非常常用的线程池管理类，提供了对线程池内线程创建、线程资源分配、排队等一系列操作。下面对其一些执行原理和源码进行分析。

#### 运行逻辑

下面先给出一个使用`ThreadPoolExecutor`的例子来说明其使用方法和需要注意的一些细节：

```java
public class ThreadPoolExecutorTest {
    public static Runnable generateThread(int id, long mills) {
        return () -> {
            try {
                System.out.println(id + " is running");
                Thread.sleep(mills);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };
    }

    public static void testThreadPoolExecutor() throws InterruptedException {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(3, true), new ThreadPoolExecutor.AbortPolicy());
        // 前5个线程由核心线程执行
        for (int i = 0; i < 5; ++i) {
            executor.execute(generateThread(i, Integer.MAX_VALUE));
            Thread.sleep(100);
        }
        // 后加入的3个线程加入到了阻塞队列
        for (int i = 5; i < 8; ++i) {
            executor.execute(generateThread(i, Integer.MAX_VALUE));
        }
        // 由于阻塞队列满了，会创建新线程执行
        executor.execute(generateThread(8, 1000));
        // 如果当前非核心线程执行完了会让阻塞队列中的线程继续执行

        // 如果超过了最大线程数则按拒绝策略处理，这里是抛出异常
        for (int i = 10; i < 20; ++i) {
            executor.execute(generateThread(i, Integer.MAX_VALUE));
        }
    }

    public static void main(String[] args) throws Exception {
        testThreadPoolExecutor();
    }
}
```

上述代码首先定义一个任务生成函数`generateThread`，接着在`testThreadPoolExecutor`中定义了一个核心线程数为5、最大线程数为10、非核心线程空闲时间超过60秒销毁、支持公平锁的容量为3的阻塞队列、拒绝策略为抛出异常的线程池。紧接着创建了5个任务，很明显这5个任务都会交给核心线程执行；然后又创建了3个任务，这3个任务由于核心线程数满并且阻塞队列可以容纳，所以进入阻塞队列等待；接着再创建1个任务，由于此时核心线程数满、阻塞队列也满，所以会创建非核心线程执行（注意：这里非核心线程执行的是id为8的这个任务，这和公平与否没有关系，公平只是说在阻塞队列中排队的线程是公平的，但是队列满了，其它的线程就会创建非核心线程执行）；最后接着创建一些线程，如果此时超过了最大线程数，那么就会按拒绝策略处理，抛出异常。

综上所述，整个`ThreadPoolExecutor`的运行流程如下：

> 当一个任务提交至线程池之后:
>
> 1. 线程池首先当前运行的线程数量是否少于corePoolSize。如果是，则创建一个新的工作线程来执行任务。如果都在执行任务，则进入2
> 2. 判断BlockingQueue是否已经满了，倘若还没有满，则将线程放入BlockingQueue。否则进入3
> 3. 如果创建一个新的工作线程将使当前运行的线程数量超过maximumPoolSize，则交给RejectedExecutionHandler来处理任务。
>
> 当ThreadPoolExecutor创建新线程时，通过CAS来更新线程池的状态ctl.

#### 源码解析

`ThreadPoolExecutor`类的继承关系图如下：

<img src="/images/Concurrent/ThreadPoolExecutor-Arch.png" alt="ThreadPoolExecutor-Arch" style="zoom:50%;" />

##### 常量

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    // ctl是控制信息，其高3位用来标识当前线程池的状态，低29位用来标识线程池的工作线程数
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 设定的标志位长度
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 实际上就是高3为全0，低3位全1的一个掩码，可以用来快速计算线程池数量和状态
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    /**
     * 线程池状态标识
     * RUNNING：线程池正常运行，创建时的默认状态。
     * SHUTDOWN：线程池中断，不接受新建任务，但会等待当前任务执行完毕。(shutdown方法)
     * STOP：线程池中断，不接受新建任务，并且中断所有正在执行的任务。（shutdownNow方法）
     * TIDYING：线程池内线程队列为空，任务队列也为空。(过渡状态，难以检测)
     * TERMINATED：线程池生命周期结束。（terminated方法）
     *  */ 
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

     // 从ctl中取状态的函数
     private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
     private static int workerCountOf(int c)  { return c & COUNT_MASK; }
     private static int ctlOf(int rs, int wc) { return rs | wc; }
 
     // ctl相关控制函数
     private static boolean runStateLessThan(int c, int s) { return c < s; }
     private static boolean runStateAtLeast(int c, int s) { return c >= s; }
     private static boolean isRunning(int c) { return c < SHUTDOWN; }
  	 
  	 // 阻塞队列
  	 private final BlockingQueue<Runnable> workQueue;
  	 // 工作线程队列
  	 private final HashSet<Worker> workers = new HashSet<>();
     // 核心锁，主要用于对workers集合进行管理和统计
     private final ReentrantLock mainLock = new ReentrantLock();
}
```

这里使用ctl这个AtomicInteger变量的原因我认为还是为了提高效率，毕竟可以使用CAS而不用竞争重量级的锁，可以提高效率。那在此基础上，就要最大化利用这个32位变量的存储空间，将前3位作为标识位，后29位用来指示当前工作线程数。虽然`ThreadPoolExecutor`内部也使用了`HashSet<Worker>`来保存和统计工作线程的个数，但是很明显它是线程不安全的，而如果要实现同步的话就要使用lock或者synchronized来实现，这样显然效率太低，所以使用了ctl的低29位来标识线程总数，再利用mainLock来进行加锁。

所以说`ThreadPoolExecutor`的原理如下：

![ThreadPoolExecutor-Loop](/images/Concurrent/ThreadPoolExecutor-Loop.png)

> 其实java线程池的实现原理很简单，说白了就是一个线程集合workerSet和一个阻塞队列workQueue。当用户向线程池提交一个任务(也就是线程)时，线程池会先将任务放入workQueue中。workerSet中的线程会不断的从workQueue中获取线程然后执行。当workQueue中没有任务的时候，worker就会阻塞，直到队列中有任务了就取出来继续执行。

线程池状态变迁过程如下：

![ThreadPoolExecutor-StatusChange](/images/Concurrent/ThreadPoolExecutor-StatusChange.png)

##### 构造函数

```java
public ThreadPoolExecutor(int corePoolSize, //核心线程数量
   int maximumPoolSize, //最大线程数
   long keepAliveTime, //超时时间,超出核心线程数量以外的线程空余存活时间
   TimeUnit unit, //存活时间单位
   BlockingQueue<Runnable> workQueue, //保存执行任务的阻塞队列
  ThreadFactory threadFactory,//创建新线程使用的工厂
  RejectedExecutionHandler handler //当任务无法执行的时候的处理方式)
```

构造函数这里只看最复杂的一个的参数，它的源码就是一些合法性判断和赋值所以忽略了。

首先先来看阻塞队列的继承关系图：

![BlockingQueue-Arch](/images/Concurrent/BlockingQueue-Arch.png)

可以看到核心接口就是`BlockingQueue`，它实现了`Queue`接口，是一个单向队列，再次基础上扩展了`BlockingDeque`和`TransferQueue`两个接口，`BlockingDeque`是双向阻塞队列的接口，`TransferQueue`则扩展了生产者功能。我们这里着重分析每个实现类的特点：

- `LinkedBlockingQueue`和`LinkedBlockingDeque`是以链表实现的阻塞队列，是一个无界队列
- `ArrayBlockingQueue`是以数组实现的阻塞队列，是有界队列
- `DelayQueue`和`DelayWorkQueue`是实现了延迟执行功能的队列
- `PriorityBlockingQueue`则是支持优先级的阻塞队列
- `SynchronousQueue`是不存储元素的阻塞队列，也就是当放置对象时立即阻塞知道对象被取出（同样是生产者消费者模式的实例）
- `LinkedTransferQueue`则是存储对象的支持生产者消费者模式的阻塞队列

内置提供的四种拒绝策略被定义在`ThreadPoolExecutor`中，分别是：

- `AbortPolicy`：直接抛出异常
- `DiscardPolicy`：丢弃任务
- `DiscardOldestPolicy`：丢弃等待时间最长的任务
- `CallerRunPolicy`：由调用者调用

接着知道了构造函数的参数，那来看一下`Executors`类提供的一些快速生成线程池的方法：

- newFiexedThreadPool

```java
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

用于生成一个具有固定线程数的线程池（corePoolSize和maximunPoolSize一致），但是由于采用了`LinkedBlockingQueue`，所以作为无界队列，它可能会堆积大量的线程请求导致OOM

- newSingleThreadPool

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

用于生成一个只有1个线程的线程池，但是问题同上，可能会堆积请求导致OOM

- newCachedThreadPool

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

用于创建一个可缓存的线程池，这里采用了`SynchronousQueue`作为阻塞队列，虽然不会导致请求堆积，但是由于`maximunPoolSize`的大小为`Integer.MAX_VALUE`, 它可能会导致线程生成过多导致OOM。

所以以上三种方法要慎重使用。

我们从最开开始的实例中可以看出，`ThreadPoolExecutor`执行任务的入口即`execute`方法，我们来看这个方法的源码：

##### execute

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 如果当前线程数小于核心线程数，那么新建核心线程，把当前任务作为新建核心线程的firstTask
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        // 这里如果失败了说明存在CAS竞争，那么尝试其它方法
        c = ctl.get();
    }
    // 如果当前线程池还在运行并且成功将当前任务加入阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 这里进行二次检查，如果此时线程池处于SHUTDOWN状态以上的话，尝试将其从阻塞队列移出并交给拒绝策略处理
        if (!isRunning(recheck) && remove(command))
            reject(command);
        // 否则因为任务已经加入阻塞队列，如果此时没有工作的核心线程，那么创建一个非核心线程
        // 并且将firstTask设置为null表明需要从阻塞队列中获取任务进行执行
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 否则运行到这里说明线程池状态已经不是运行，或者阻塞队列添加失败（满或者其它原因）
    // 尝试创建非核心线程执行，失败则交给拒绝策略（addWorker会检查当前线程池状态）
    else if (!addWorker(command, false))
        reject(command);
}
```

这里就是3个步骤：首先如果线程数小于核心线程数，那么尝试创建核心线程；如果创建失败或者线程数大于核心线程数，那么将其加入阻塞队列，加入阻塞队列进行二次检查，防止在进入这个方法时线程池已经SHUTDOWN或STOP，那么就需要回滚入队操作并交给拒绝策略处理，或者出现刚好所有线程都恰好死亡，此时要创建新线程，防止阻塞队列中的任务无法停止地等待；最后如果阻塞队列满或者线程池已经中止，那么就尝试创建非核心线程（线程池中止由`addWorker`方法返回`false`），如果还是失败那么交给拒绝策略处理。

上述代码中最核心的就是`addWorker`方法了，下面我们来看它的实现：

##### addWorker

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 无限循环，目的是对wokerCount进行递增（即ctl的低29位）
    // workerCount是用来标记工作线程总数的标志，只有它成功递增了才能创建新线程
    for (int c = ctl.get();;) {
        /**
         * 直接返回添加失败的两个条件：
         * 1. 如果线程池处于STOP状态，不接受任务
         * 2. 如果线程池处于SHUTDOWN状态，且firstTask为空或阻塞队列为空
         * 解释：STOP状态会立即停止所有线程，并且拒绝新任务添加，所以直接返回false
         *      SHUTDOWN状态下就需要判断阻塞队列是否为空了，
         *          如果firstTask不为空说明这是新建任务，不允许创建
         *          如果阻塞队列为空那说明没有等待的任务了，不应该创建新线程
         *          否则说明还有任务没有执行完毕，需要等待，可以创建新线程辅助
         */
        if (runStateAtLeast(c, SHUTDOWN) &&
            (runStateAtLeast(c, STOP) || firstTask != null || workQueue.isEmpty()))
            return false;

        for (;;) {
            // 判断创建的核心线程是否超过核心线程上限，非核心线程是否超过总量上限
            if (workerCountOf(c)
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                return false;
            // CAS递增，成功则退出外层循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 否则重新判断是否处于SHUTDOWN以上状态，如果是则返回外层循环
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;
            // 否则说明只是CAS递增失败，可能只是有竞争，可以继续尝试这个过程
        }
    }

    // 标记任务是否成功启动
    boolean workerStarted = false;
    // 标记任务是否成功添加到阻塞队列
    boolean workerAdded = false;
    // Worker代表了一个不可重入的线程类，利用AQS实现资源管理
    // 自身实现Runnable接口实现线程作业
    Worker w = null;
    try {
        // 尝试创建一个Worker
        w = new Worker(firstTask);
        // 获取其中的实际执行线程（Worker持有一个线程对象用于执行任务）
        final Thread t = w.thread;
        if (t != null) {
            // 为了操作全局Worker集合，必须获取全局锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 进行二次检查，防止线程池处于STOP以上状态（SHUTDOWN不可以添加，但可以执行阻塞队列的任务）
                int c = ctl.get();
                
                if (isRunning(c) ||
                    (runStateLessThan(c, STOP) && firstTask == null)) {
                    if (t.getState() != Thread.State.NEW)
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    workerAdded = true;
                    int s = workers.size();
                    // 记录最大达到的线程池大小，没有用ctl的原因应该是这个变量只是为了分析使用，出于效率考虑采用非同步方法
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 如果Worker成功添加，那么启动Worker中的线程
                // 由于Worker本身就是Runnable对象，在构造内部Thread时将自己作为Thread构造函数的对象进行初始化
                // 所以这里实际上调用的是Worker的run方法
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

`addWorker`方法就干两件事：首先利用CAS来递增ctl中的workerCnt，成功后创建Worker对象，将任务添加到阻塞队列中或Worker的firstTask中，委托给Worker进行执行。参数中的firstTask可以为空，如果为空说明创建的新线程应该从阻塞队列中获取新任务进行执行，它容易出现在当前任务添加到阻塞队列后发现可以新建线程时，就可以传入null，这样Worker将从阻塞队列中获取任务进行执行。

接下来我们看Worker类的源码：

##### Worker类

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {

    private static final long serialVersionUID = 6138294804551838833L;

    // 当前Worker对应的线程  
    final Thread thread;
    // 初始任务，可以为空
    Runnable firstTask;
    // 每个线程已经完成的任务计数器
    volatile long completedTasks;

    // 构造函数
    Worker(Runnable firstTask) {
        // 运行前禁止中断
        setState(-1);
        this.firstTask = firstTask;
        // 因为Worker也是一个Runnable接口实现类，所以用自己（this）来初始化Thread
        this.thread = getThreadFactory().newThread(this);
    }

    // Runnable入口，实际上交给runWorker方法代理
    public void run() {
        runWorker(this);
    }

    // AQS排他锁检测
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    // CAS获取资源，参数unused
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    // 释放资源，这里调用这个方法的线程必然持有锁，否则不能调用
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    // 加锁函数
    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    // 中断函数，必须保证：持有锁、线程非空、线程没被中断过
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

`Worker`类的设计理念是通过继承AQS类的方式控制资源即线程对象，当线程空闲时为释放资源，当需要使用线程时则获取资源；同时通过实现`Runnable`类来将`Worker`本身作为一个可交由线程执行的对象，可以拥有一个默认任务firstTask，这个默认任务会被线程最先执行，同时为了是的`Worker`可以访问其它被阻塞的任务，所以这里实际调用的`run`方法委托给内部类外的`runWorker`方法执行。

`runWorker`的源码如下：

##### runWorker

```java
final void runWorker(Worker w) {
    /** 
     * 获取当前执行的线程
     * 我们知道这个方法的调用者是run方法，也就是由线程池中的线程调用的
     * 所以只需要获取当前线程就可以获得当前线程池中调用的线程了
    */
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    // 清空Worker的首个任务，防止被重复调用
    w.firstTask = null;
    // 释放获取的线程资源，目的是支持运行时被中断，同时在没有任务时能让线程得到利用
    w.unlock();
    // 中断标志
    boolean completedAbruptly = true;
    try {
        // 不断获取task，其中第一个task就是firstTask，当firstTask为空时
        // 从阻塞队列中获取一个新的任务，如果没有任务，getTask会阻塞
        while (task != null || (task = getTask()) != null) {
            // 加锁，获取线程资源
            w.lock();
            /**
             * 如果线程池处于STOP状态，并且当前线程还没有被中断
             * 同时如果第一次检查到线程池不处于STOP状态但是当前线程已经处于中断状态
             * 那么需要再次检查线程池是否处于STOP状态，防止在第一次检查时与shutdownNow竞争
             */
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

逻辑很简单，就分成运行前检查、运行前处理、运行和运行后处理四个部分，一次运行后从阻塞队列中取新任务，如果没有新任务则阻塞。这个取任务和阻塞的过程是由`getTask`函数处理的，其源码如下：

##### getTask

```java
private Runnable getTask() {
    // 标记当前取任务是否超时
    boolean timedOut = false;

    for (;;) {
        int c = ctl.get();

        // 如果处于STOP状态，或者处于SHUTDOWN状态且任务阻塞队列为空，那么线程计数减1
        // 返回null表示没有需要处理的任务了
        if (runStateAtLeast(c, SHUTDOWN) && 
            (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        // 获取当前工作线程数
        int wc = workerCountOf(c);
        // 标记是否需要考虑超时，只有核心线程允许超时或者存在非核心线程时才考虑超时
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 如果线程数超过最大线程数或者触发超时，同时有多于1个线程或者任务阻塞队列为空是，线程计数减1
        // 这就是说要么线程太多或超时，与此同时任务太少，这个时候就需要减少线程持有量
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 带超时机制和不带超时机制两种
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            // 如果r为null，就说明超时还没取到
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

很简单，就是要考虑一下超时的问题和线程过多的问题。返回null就会导致runWorker中对应的Worker退出使用，退出方法在`runWorker`函数最后的`processWorkerExit`中，源码如下：

##### processWorkerExit

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果说是被异常中断的，那么肯定没有减少Worker数量
    // Worker数量是在getTask返回null的时候递减的，所以这里要减1
    if (completedAbruptly)
        decrementWorkerCount();
    // 获取全局锁，用来操作Worker集合和相关的统计信息
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 记录完成的任务总数
        completedTaskCount += w.completedTasks;
        // 移出当前Worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // 如果这是最后一个线程那就设置线程为TIDING或TERNIMATE状态
    tryTerminate();

    int c = ctl.get();
    // 如果是SHUTDOWN或RUNNING状态还要考虑如果这是最后线程池中的最后一个线程
    // 同时此时又有新的线程提交到阻塞队列，此时就需要创建一个非核心线程来执行它
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

其中`tryTerminate`方法用来解决最后一个线程关闭后线程池状态转换的问题以及每当一个线程被销毁时同时尝试去销毁其它空闲线程。其中`tryTerminate`源码如下：

##### tryTerminate

```java
final void tryTerminate() {
    // 自旋
    for (;;) {
        int c = ctl.get();
        // 如果线程池正在运行，或者已经处于TIDYING或TERMINATED状态
        // 或者处于SHUTDOWN状态但是阻塞队列不空的情况，那么不应该中止线程池，直接返回
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateLessThan(c, STOP) && !workQueue.isEmpty()))
            return;
        // 如果线程池还存在线程，那么尝试停止其中的一个空闲线程（ONLE_ONE）
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        // 否则说明：线程池处于STOP状态或SHUTDOWN状态且阻塞队列为空
        // 那就需要完整清理了
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 首先设置ctl为TIDYING状态，工作线程数为0
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 执行中止函数，这个函数是一个回调函数，ThreadPoolExecutor中为空
                    // 可以自定义来执行一些自定义操作
                    terminated();
                } finally {
                    // 最后设置线程池状态为TERNIMATED，工作线程为0
                    // 同时唤醒所有等待线程池中止的线程
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // 如果自旋说明存在竞争，需要重新判断状态，尤其会出现在CAS设置为TIDYING的过程中
    }
}
```

逻辑很清楚，不多解释了，这里再看一下`interruptIdleWorkers`函数：

##### interruptIdleWorkers

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 遍历所有Worker
        for (Worker w : workers) {
            Thread t = w.thread;
            // 如果还没有被中断，且尝试加锁成功就说明空闲
            // 如果加锁失败就说明线程正忙，会直接退出
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            // 这里的onlyOne不是说只中断一个，而是只尝试队列中的第一个
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

也很简单，不解释，注意一下，空闲线程的判断利用了`tryLock`函数来实现，它尝试获取Worker的锁，如果获取成功就说明当前线程空闲，反之说明当前线程正忙。之所以可以这样获取就是因为在`runWorker`中，如果Worker在等待新任务它是不持有锁的，这也就给这里一个机会去释放这个线程。同样还要注意这个onleOne变量，它指的是只尝试一次，无论成功或者失败都立即退出，而不是一定会中断一个空闲Worker。

##### 任务的提交

由于`ThreadPoolExecutor`实现了`Executor`接口，所以它包含`execute`函数，其定义如下：

```java
void execute(Runnable command)
```

可以看到，它只接受`Runnable`任务，所以如果想要提交带返回值的任务，那么一个是可以用后面会说的`submit`方法，另一个则可以使用之前说过的`FutureTask`，它通过包装Callable接口对象，同时实现Runnable接口，最终实现了内部保存返回值的Runnable对象。

第二种提交任务的方式则是使用`submit`方法，它是`AbstractExecutorService`类提供的，定义如下：

```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
```

可以看到，它同时支持Runnable任务和Callable任务，但是它本质上也是通过FutureTask实现的，因为存储返回值的是`RunnableFuture`接口的实现类由newTaskFor方法返回，可以看到其源码就是返回了一个FutureTask对象。

综上所述，整个任务提交的流程可以概括成下图：

![ThreadPoolExecutor-Submit](/images/Concurrent/ThreadPoolExecutor-Submit.png)

##### 任务的关闭

最后我们再来分析任务的关闭，和FutureTask一样，任务取消有两种方式`shutdown`和`shutdownNow`。其中前者将线程池设置为SHUTDOWN状态，使其不接受新的线程，但会等待所有已经提交的线程执行完毕；而后者除了不接受新线程外，还会中断所有正在执行的任务。下面我们对比着看`shutdown`方法和`shutdownNow`方法：

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查权限
        checkShutdownAccess();
        // 修改线程池状态
        advanceRunState(SHUTDOWN);
        // 先尝试中断所有空闲线程
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        // 唯一不同就是这里会直接中断所有Worker
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}

private void advanceRunState(int targetState) {
    // assert targetState == SHUTDOWN || targetState == STOP;
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, targetState) ||
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
            break;
    }
}

private void interruptWorkers() {
    // assert mainLock.isHeldByCurrentThread();
    for (Worker w : workers)
        w.interruptIfStarted();
}

// 这个方法是内部类Worker的方法
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

`shutdownNow`方法实现线程池的关闭很简单，它通过立即中断线程实现。这里我们分析一下`shutdown`方法的执行流程以及它为什么能最终停下线程池：

1. 首先CAS修改线程池状态为SHUTDOWN，如果这里有竞争，说明可能存在多个线程将当前线程池状态设置为SHUTDOWN或STOP及以上状态，所以`advanceRunState`方法用了`runStateAtLeast`来判断，如果当前状态大于等于给定的状态就退出不再设置。当线程池状态被修改后，如果前面的`getTask`运行到检查方法时就会退出；如果`getTask`已经进入到workQueue的等待方法中时，则继续下一个步骤。
2. 尝试中断空闲线程，我们在`interruptIdleWorkers`和前面分析`runWorker`时说过，当Worker在`getTask`中等待时是不持有锁的，也就是空闲对象，此时对这类空闲对象直接`tryLock`是可以成功的。所以通过这个方法，就可以中断所有正在等待新任务的空闲Worker。
3. 而对于正在执行的线程即运行到`runWorker`循环内的线程，在执行完毕回到while循环的`getTask`方法时，也会因为`getTask`的状态判断为SHUTDOWN而返回null，导致Worker被销毁。这样就实现了所有Worker的销毁工作。
4. 当所有任务都结束时，在`tryTerminate`中将线程池修改为TIDYING状态，当自定义的中止函数`terminated`运行结束后，线程池转入TERMINATED状态。

##### 配置线程池考虑的因素

> 从任务的优先级，任务的执行时间长短，任务的性质(CPU密集/ IO密集)，任务的依赖关系这四个角度来分析。并且近可能地使用有界的工作队列。
>
> 性质不同的任务可用使用不同规模的线程池分开处理:
>
> - CPU密集型: 尽可能少的线程，Ncpu+1
> - IO密集型: 尽可能多的线程, Ncpu*2，比如数据库连接池
> - 混合型: CPU密集型的任务与IO密集型任务的执行时间差别较小，拆分为两个线程池；否则没有必要拆分。