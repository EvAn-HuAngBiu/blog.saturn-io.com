---
layout: post
title: ConcurrentHashMap 8
categories: 并发
keywords: ConcurrentHashMap, JUC
---

# ConcurrentHashMap (8+)

## 常量

下面是ConcurrentHashMap中定义的一些常量，主要包括了一些预置的默认值：

```java
// node数组最大容量：2^30=1073741824
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认初始值，必须是2的幕数
private static final int DEFAULT_CAPACITY = 16;

//数组可能最大值，需要与toArray（）相关方法关联
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//并发级别，遗留下来的，为兼容以前的版本
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

// 负载因子
private static final float LOAD_FACTOR = 0.75f;

// 链表转红黑树阀值,> 8 链表转换为红黑树
static final int TREEIFY_THRESHOLD = 8;

//树转链表阀值，小于等于6（tranfer时，lc、hc=0两个计数器分别++记录原bin、新binTreeNode数量，<=UNTREEIFY_THRESHOLD 则untreeify(lo)）
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
private static final int MIN_TRANSFER_STRIDE = 16;
private static int RESIZE_STAMP_BITS = 16;

// 2^15-1，help resize的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

// 32-16=16，sizeCtl中记录size大小的偏移量
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

// forwarding nodes的hash值
static final int MOVED     = -1;

// 树根节点的hash值
static final int TREEBIN   = -2;

// ReservationNode的hash值
static final int RESERVED  = -3;

// 可用处理器数量
static final int NCPU = Runtime.getRuntime().availableProcessors();

//存放node的数组
transient volatile Node<K,V>[] table;

/*控制标识符，用来控制table的初始化和扩容的操作，不同的值有不同的含义
 *当为负数时：-1代表正在初始化，-N代表有N-1个线程正在 进行扩容
 *当为0时：代表当时的table还没有被初始化
 *当为正数时：表示初始化或者下一次进行扩容的大小*/
private transient volatile int sizeCtl;
```

## 构造函数

```java
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```



## put

插入操作：

```java
// 继承自Map接口的put函数，这个是所有插入操作的入口，但实际调用的是下面的putVal函数
public V put(K key, V value) {
   return putVal(key, value, false);
}

    // 同时支持put和putIfAbsent方法
    final V putVal(K key, V value, boolean onlyIfAbsent) {
      	// 不允许插入空值和空键
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
      	// 计算当前链表的长度，用来后续辅助判断是否需要树化
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
              	// 如果当前table为空那么先进行初始化，然后接着循环判断
                tab = initTable();
          
          	// f幅值为table数组中当前键对应下标位置上的Node，如果这个Node为空说明当前桶是空的，直接尝试CAS赋值即可
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                  	// 如果CAS赋值成功则退出循环，插入结束
                  	// 如果CAS赋值失败说明存在竞争，那么重新循环判断当前桶是否还是空的，如果还是则继续尝试CAS，否则执行下面的插入过程
                    break;                   // no lock when adding to empty bin
            }
          
          	// fh是当前节点的hash值
            else if ((fh = f.hash) == MOVED)
              	// 如果当前节点hash值为保留值MOVED（-1）那么说明table正在扩容，这里调用辅助扩容函数更新table
                tab = helpTransfer(tab, f);
          
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
              	// 这里是一个快速检查，如果对于那些不允许覆盖的操作，当检查到桶的头结点的键和待插入的键一致的话直接返回原值即可不需要插入
                // 这是一种快速失败的优化
                return fv;
          
            else {
              	// 到这里说明，f是当前桶的头结点并且不为空，也尚未被扩容，同时在不可覆盖模式下头结点和待插入节点的键不一致需要向后查找
              	// 也就是说明会有几种情况：
                // 1. 允许覆盖的情况下，并且头结点是链表节点的话，需要遍历链表查找是否有相同的键，如果有则覆盖
              	// 2. 如果没有相同的键则插入到最后，同时要计算链表长度考虑是否要树化
                // 3. 如果头结点是树节点的话，则调用红黑树方法查找
                // 4. 不允许覆盖的情况同理，可以直接从第二个节点开始查找
                V oldVal = null;
                synchronized (f) {
                  	// 获取头结点的锁，保证当前头结点是不被其它节点修改的
                    if (tabAt(tab, i) == f) {
                      	// 进行再次检查，查看当前头结点是否发生改变，比如等待锁的过程中当前链表发生了删除等操作，需要重新判断
                      	// 如果当前头结点和之前记录的不一致的话，就放弃锁重新进行循环判断插入位置
                        if (fh >= 0) {
                          	// 头节点hash值大于0表示是链表节点，那么记录链表长度
                            binCount = 1;
                          	// 遍历即可，如果有相同键则替换，否则插入到最后，同时要更新链表长度binCount
                          	// 只要插入成功会立刻break，退出循环
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                          	// 如果是红黑树，则调用红黑树的方法插入（fh == -2）
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                          	// 最后一种循环的情况（fh == -3）
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                  	// 如果当前计数值大于阈值的话考虑是否要树化（不一定树化，树化的条件是大于等于阈值64），
                    // 但是这里的阈值是8，也就是说在链表长度大于8并且小于64的时候会先尝试将数组大小翻倍而不是树化
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

## initTable

initTable是table的初始化方法，它只发生在当前table尚未被初始化即（table == null 或 table.length == 0）时，并且利用CAS操作和sizeCtl变量来保证并发安全。当table尚未被初始化时sizeCtl为0，在初始化过程中尝试使用CAS来将sizeCtl设置为-1表示当前线程开始了初始化操作，其它线程等待即可；而其它线程在判断sizeCtl=-1时会让出时间片，或者CAS失败后会重新循环判断，源码如下：

```java
		private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
      	// 如果table没有被初始化会一直循环，但是如果初始化成功那么就会自动退出
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
              	// sizeCtl小于0表示其它线程正在初始化，所以当前线程让出时间片等待即可
                Thread.yield(); // lost initialization race; just spin
          	// 尝试使用CAS获取锁，这个锁就是指将sizeCtl设置为-1，当尚未初始化时sizeCtl为0，此时CAS可能成功（如果没有竞争或成功抢到）
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                  	// 再次检查，防止table被初始化了（有可能初始化后其它线程进入while循环，此时获取的sizeCtl为正数，表示已经初始化的table的容量的阈值
                  	// 这样也能通过上面的CAS判断进入到这里，所以这里再次判断table为空才能进行初始化
                    if ((tab = table) == null || tab.length == 0) {
                      	// 如果sc=0，说明还未初始化，那么初始化为默认容量16
                      	// 否则如果指定了容量空间，那么初始化为指定的容量空间
                      	// 正是由于sizeCtl可能在前面表示初始空间的含义，所以这里需要判断table是否为null以及table长度是否为0来判空
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                      	// 新的sizeCtl，实际上就是容量的0.75倍，和默认loadFactor一致
                      	// 实际上我认为，即使构造函数中允许传入loadFactor参数，但是在构造函数实现时（JDK11），这个loadFactor只是用来标记
                      	// 初始情况下的容量，而对于中间链表满的判断还是根据0.75的阈值来进行判断的，sizeCtl存储的实际上就是当前的阈值
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

## treeifyBin

树化操作，但是不一定树化，当链表表长大于阈值8小于最小树容量64时会尝试先扩容数组，源码如下：

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
              	// 如果桶长度小于树最小容量64时尝试扩容，新大小是原来的2倍
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
              	// 如果当前的头结点非空且为链表节点时，加锁
                synchronized (b) {
                  	// 同理再次判断，防止重复扩容
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                      	// 树化过程
                        // 这里实际上是先构造了一个双向链表来辅助树化
                        // 还要注意一点，对于红黑树而言，HashMap使用了TreeNode节点直接作为红黑树的根节点存储在桶中
                        // 而ConcurrentHashMap则使用的是TreeBin来存储，这样做的目的是由于红黑树的根节点可能会发生变化，
                        // 而CHM需要对根节点加锁来保证一致性，由于根节点变化后可能导致对新的根节点需要重复加锁的问题
                        // 所以包装了一层TreeBin，只要对TreeBin加锁，那么就相当于对整个红黑树都加锁了，这样避免了
                        // 由于根节点变化可能导致的加锁问题
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                      	// 将红黑树设置到数组相应位置中，实际生成红黑树是在TreeBin的构造函数中实现的
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

## 扩容操作

接着我们来看整个ConcurrentHashMap的扩容操作，这是整个ConcurrentHashMap中最复杂的一部分。

### addCount

首先先来看`putVal`中调用了`addCount`函数实现对当前CurrentHashMap的容量计数。这里有点奇怪为什么需要用一个专门的函数来计数，这里明明可以使用`AtomicLong`或者`LongAdder`实现，可以源码中却采用了一种很奇怪的实现方式，其原因只有一个：提高效率，因为这个size在很多地方都要使用，比如添加、删除或者单纯获取size，是多个线程竞争的对象，所以CHM在设计上采用了将size分割成几部分的形式，size实际上是字段baseCount和CounterCell数组计数器的总和，这样设计的好处就是：原来是多个线程竞争一个变量size，现在每个线程可以既可以竞争baseCount也可以竞争计数器数组中的某一项，总而言之都是为了计数，从而提高并发，因为这样如果A线程使用CounterCell[0]来计数，B线程使用CounterCell[1]来计数，那么他们是互不干扰的、可以并发的，最终计算size的时候只要把CounterCell数组中所有计数和baseCount相加即可。下面来看源码：

```java
private final void addCount(long x, int check) {
    // 这里cs就是计数器数组了
    CounterCell[] cs; long b, s;
    /*
    这里有两个条件：
    1. 让cs等于CHM类中的计数器counterCells，如果CHM类中的计数器已经被初始化过了，那么满足条件
    2. 否则说明CHM类中的计数器没有被初始化过，这里尝试竞争baseCount，利用CAS将baseCount加上x
    */
    if ((cs = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 进入到这里说明：计数器数组没有被初始化，或者数组被初始化了但是竞争baseCount失败
        CounterCell c; long v; int m;
        boolean uncontended = true;
        /* 
        进一步判断：如果是计数器数组没有初始化，或者计数器数组中对应该线程的那个计数器没有被初始化，或者竞争当前计数器也失败了
        这里要明确一点，线程去寻找自己对应哪一个下标的计数器是依赖线程自己生成的随机数对数组长度取余得到的，这个随机数只会生成一次，
        后面生成的随机数就都一样了
        */
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
            // 实际初始化和添加的函数，这里的uncontended就是false了
            fullAddCount(x, uncontended);
            return;
        }
        // 这个check值就是用来检查是否需要扩容用的，如果小于等于1说明肯定不需要扩容，直接返回即可
        if (check <= 1)
            return;
        // 否则需要统计总数，就是将baseCount和各个CounterCell的计数值相加
        s = sumCount();
    }
    // 判断需不需要进行扩容，check<0只会发生在删除操作时，显然不需要进行扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 可以看到触发扩容有三个条件：1. 容量大于阈值；2. table数组非空；3.没有达到树化最小值
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            // 如果sizeCtl已经小于0了那就说明有其它线程已经开始了扩容操作，这里只要辅助就好了
            if (sc < 0) {
                //判断扩容是否结束或者并发扩容线程数是否已达最大值，如果是的话直接结束while循环
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //扩容还未结束，并且允许扩容线程加入，此时加入扩容大军中
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 如果是首个进行扩容的线程，那么将sizeCtl值改为(rs << RESIZE_STAMP_SHIFT) + 2,
            //(rs << RESIZE_STAMP_SHIFT) + 2 为首个扩容线程所设置的特定值，后面扩容时会根据线程是否为这个值来确定是否为最后一个线程
            else if (U.compareAndSetInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

下面我们接着来看当CounterCells数组的初始化过程，也就是`fullAddCount`函数：

### fullAddCount

```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    // 生成线程随机数，初始可能是0，需要初始化后再重新获取
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] cs; CounterCell c; int n; long v;
        // 第一种情况：CHM的counterCells数组已经被初始化且不为空, 那么尝试更新CounterCell
        if ((cs = counterCells) != null && (n = cs.length) > 0) {
            // 接着分类，情况1.1：如果当前数组已经被初始化，但是数组中当前线程对应的哪一个CounterCell没被初始化的话
            if ((c = cs[(n - 1) & h]) == null) {
                // 首先第一次判断CounterCells数组是否繁忙，这里没用CAS判断，是一种乐观锁
                if (cellsBusy == 0) {
                    // 乐观创建对象，没对数组加锁，这里已经把需要加的x作为默认值传入构造了
                    CounterCell r = new CounterCell(x);
                    // 到这里才尝试利用CAS获取锁
                    if (cellsBusy == 0 &&
                        U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                        // 标识当前对象是否被创建
                        boolean created = false;
                        try { 
                            CounterCell[] rs; int m, j;
                            // 再次检查数组初始化了不为空，并且当前线程对应下标的CounterCell为空
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                // 赋值并修改标记
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            // 释放锁
                            cellsBusy = 0;
                        }
                        if (created)
                            // 如果创建成功了，那么退出即可，初始化添加操作就结束了
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            // 进入下面的分支意味着当前线程对应的CounterCell对象已经被创建了，不为空
            // 情况1.2：这种情况要回到上一个函数addCount中，可以发现传入的uncounted对象意思是：当前线程对应的CounterCell对象不为空
            //         并且已经尝试过利用CAS来更新CounterCell但是失败了，所以这里将其状态改为true表明还未尝试，下面代码会重新获取
            //         一个随机数，这个随机数可能会对应到不同的CounterCell中，进行下一次尝试
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 情况1.3：对当前线程随机数对应的CountCell进行CAS更新，如果成功则返回，失败则继续判断后续的if
            //         这个情况是建立在1.2不满足的情况下进行的，也就是说可能线程随机数被重新生成了但是还没有CAS尝试，所以这里尝试一下
            else if (U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))
                break;
            // 情况1.4：如果CAS尝试也失败了，说明重新生成随机数也是失效的，这里可能存在竞争CountCell过大的问题
            //         那么这里判断如果当前的CounterCells数组被更新过了，或者当前CounterCell数组已经大于当前CPU的核心数了
            //         那么就不允许再次扩容了，这个时候剩下的所有if语句都不会被执行了，会不断重复上面的步骤：不断生成新的随机数，尝试
            //         更新CounterCell，直到更新成功为止。反之说明CounterCell被更新过需要重新尝试，或者CounterCell可以更新进入
            //         下一种情况
            else if (counterCells != cs || n >= NCPU)
                collide = false;            // At max size or stale
            // 情况1.5：执行到这里说明CounterCell可以被扩容，那么判断设置碰撞标记collide，如果下次再发生碰撞执行失败那么会执行最后一个if
            //         进行扩容。也就是说扩容操作并不是立刻就执行的，而是还会再尝试循环一次，利用CAS修改新随机数对应的CounterCell，
            //         如果还是失败了才会执行情况1.6的扩容操作
            else if (!collide)
                collide = true;
            // 情况1.6：执行扩容，扩容后不再生成新随机数而是利用当前随机数再次尝试更新CounterCell，如果当前CounterCells数组正忙或者有竞争
            //         那么还是会生成新的随机数来不断尝试CAS，直到CounterCells数组不忙并且当前线程抢到cellsBusy锁时才触发扩容
            else if (cellsBusy == 0 &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == cs) // Expand table unless stale
                        counterCells = Arrays.copyOf(cs, n << 1);
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        /* 
        第二种情况：当前CHM的countCells为空还未被初始化，那么首先要先判断当前cell是否繁忙（利用cellsBusy变量，相当于一个乐观锁）
        并且还要判断当前线程使用的这个counterCells副本是否被修改过，同时利用CAS获取锁，也就是将cellsBusy变量置1
        */
        else if (cellsBusy == 0 && counterCells == cs &&
                 U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
            // 记录是否初始化
            boolean init = false;
            // 初始化counterCells数组，默认大小是2，下面的 h & 1的1实际上就是rs数组长度-1，做的事就是取余，只是这里把值写死了
            try {
                if (counterCells == cs) {
                    CounterCell[] rs = new CounterCell[2];
                    // 这里实际上把x作为默认初值进行初始化，就完成了addCount的操作
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                // 释放锁
                cellsBusy = 0;
            }
            if (init)
                // 如果初始化成功，那么就跳出循环，值成功添加，退出即可
                break;
        }
        // 第三种情况：CounterCells数组没初始化或正忙，继续尝试使用CAS更新baseCount，若成功则退出即可
        else if (U.compareAndSetLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          
    }
}
```

这个`fullAddCount`函数非常复杂，外层包含一个for的死循环表明只有成功将这个x更新了才能退出。内部包含三种情况：

1. 当前CHM的CounterCells数组已经被初始化了并且不为空，那么就要尝试更新当前线程随机数对应的CounterCell
2. 否则说明当前CHM的CounterCells数组还没有被初始化，那么需要获取数组锁cellsBusy，获取成功的话就可以创建新的数组对象，并将当前的x值作为默认值来初始化当前线程随机数对应的CounterCell，这样就完成了更新。
3. 否则说明数组既不存在，其他线程又在操作这个数组（可能是初始化），那么当前线程就尝试用CAS更新baseCount，如果成功则退出，失败则重新循环重复全部的步骤。

但是在更新CounterCell时也会存在不断的冲突的问题，可能需要对CounterCell数组扩容、重新生成随机数等操作，也分为以下6种情况：

1. 如果当前线程随机数对应的CounterCell为空，那么尝试获取锁去初始化它，如果初始化成功，就说明需要更新的x值已经更新了，可以退出
2. 否则，如果是在对CounterCell CAS失败（用wasUncontended变量表示，第一次进入时肯定是失败的，因为在`addCount`方法里已经尝试过使用CAS赋值但是失败了才会进到`fullAddCount`函数中），那么首先尝试生成一个新的随机数再次尝试
3. 否则，如果生成的是新的随机数需要尝试，那么调用CAS进行尝试
4. 如果尝试失败了，也就是说明竞争还是存在，那就要考虑对CounterCells数据进行扩容了，但是在扩容之前要判断是否其它线程已经扩容过了（即cs != counterCells），那就会清除扩容标记（collide），先尝试生成新的随机数再次CAS设置CounterCell；或者当前CounterCells数组长度已经超过了CPU核心数（n >= NCPU），那么每次执行到这个步骤时都会清除扩容标记，不再允许扩容，而是不断生成新的随机数尝试CAS设置CounterCell，直到成功为止
5. 如果条件4不满足，即允许扩容，那么在这里会设置扩容标记，并且生成一个新的随机数，重新尝试步骤1到4，如果还是失败那么会执行步骤6进行扩容
6. 对数组进行扩容，这个进入条件非常苛刻，是在步骤5设置扩容标记后再次尝试新随机数CAS设置CounterCell还失败的情况下才会尝试扩容CounterCells数组，扩容完成后利用当前的随机数重新执行步骤1。

以上就是整个`fullAddCount`的步骤了，步骤非常麻烦，没有采用while循环+CAS的方式来不断等待给CounterCell赋值，而是每次都只尝试一下，如果失败会转而生成新随机数或者扩容，由于扩容的消耗较大，所以进入条件也比较苛刻，只有在不断重试都失败的情况下才考虑扩容操作。

### transfer

这是负责数组扩容的核心方法：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // stride就是步长，即每个线程要负责扩容多少个桶，这个桶指的是CHM中桶数组的中元素的个数
    // 单核CPU就负责1个，多核CPU负责核心数/8个，最小转移数MIN_TRANSFER_STRIDE=16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 创建新的table数组，这里不用加锁的原因就是在addCount或者tryPresize方法中，要传入null都必须是
    // 已经获取了锁的情况即成功的将sizeCtl设置为负值的那个线程才能传入null，其它线程都不能传入null
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // 给CHM的字段赋值
        nextTable = nextTab;
        // 扩容前table的大小，其实指示的是当前线程处理的最后一个桶的下标
        // 因为扩容操作允许并发，所以用这个值下一个线程要从哪个桶开始进行扩容，初始就是末尾了
        // 注意这个扩容操作是从最后一个桶开始反向进行的
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 标记是否需要向前寻找其它桶要不要帮助扩容
    boolean advance = true;
    // 标记当前线程是否完成了扩容，包括在桶数组向前寻找没有能帮助的桶时，才设置为true
    // 此时只表示当前线程完成了辅助扩容操作，而其他线程是否完成则不知道
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 这个循环中的i表示的就是需要处理的第一个桶的下标，bound表示需要处理的最后一个桶的下标即边界
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 如果允许向前搜索，那么会持续循环
        while (advance) {
            int nextIndex, nextBound;
            // 如果说当前位置i已经超过边界了，或者已经处于完成状态了，那么直接退出
            // 这里的--i>=bound实际上代替了上面for循环语句中的步进和边界条件
            // 每次处理完一个桶后都会在这里完成指针向前移动和边界判断的过程
            // 如果在边界内还会跳出while循环执行后续的代码
            if (--i >= bound || finishing)
                advance = false;
            // 将nextIndex设置为当前的边界值，如果它到数组头部，就说明所有桶都已经分配给了线程
            // 不需要再进行辅助扩容了，所以设置标记、退出循环
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 线程竞争下一个辅助扩容的起始点，本质上就是利用CAS设置transferIndex值，因为
            // 它指示了下一个需要被辅助的桶的位置，如果竞争成功则设置新的边界，并且不再向前寻找
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // 这里的nextBound就是下一个边界值，把它赋给bound表明当前的边界，取值在上面if中，
                // 如果步长大于当前剩余未处理的桶数那边界很明显就是0，否则就是上一个起始点减去步长
                bound = nextBound;
                // i是起始值，就是起始点往前1个就是开始的桶下标
                // 要注意这里i、bound是和数组下标关联的（从0开始），nextIndex、nextBound是和计数关联的（从1开始计数）
                i = nextIndex - 1;
                // 设置退出循环的标志
                advance = false;
            }
        }
        // 这个if实际上是退出的判断，i<0是上面while中第二个if将其设置为-1才成立的，当i=-1说明所有桶都已经被分配完了
        // 所以进入这个if进行退出判断
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                // 将nextTable赋值给table
                nextTable = null;
                table = nextTab;
                // 这里是个小优化，因为原来的sizeCtl表示阈值，是数组长度0.75倍，但是此时数组长度翻倍了，显然sizeCtl也会翻倍
                // 所以sizeCtl就是原数组长度的1.5被，即2*n-0.5n
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 这里尝试将sizeCtl-1,意思就是当前线程退出了
            // 首次进入transfer函数的线程会把sizeCtl设置为resizeStamp(n) << RESIZE_STAMP_SHIFT + 2
            // 后续线程会在sizeCtl的基础上+1，所以当前线程退出则将sizeCtl-1
            // 最后一个线程退出则意味着将sizeCtl - 2 = resizeStamp(n) << RESIZE_STAMP_SHIFT
            // 说明所有线程都完成了转移，这时候就可以将新的nextTable赋值给table了，整个转移工作就完成了
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    // 不是最后一个线程，退出
                    return;
                finishing = advance = true;
                // 将i赋值为n是为了能够再次进入这个if语句
                i = n; // recheck before commit
            }
        }
        // 如果当前桶是空的，那么直接尝试CAS将它设置为移动状态的标识节点fwd
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果当前桶已经是处于移动状态，那就认为移动成功
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 这里要对桶加锁，也就是说在扩容到某个桶的时候是不允许put操作并发执行的
            synchronized (f) {
                // 重新判断当前桶是否被改变，也即在这个过程中是否有put新的节点到当前桶中
                if (tabAt(tab, i) == f) {
                    // 以下的就是链表转移的逻辑了，和1.7一致
                    // 简单的说就是找到不变的那个节点，将它之后的节点挂在一起，其它节点依次处理即可
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        // 红黑树同理
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

### helpTransfer

当其它线程操作操作ConcurrentHashMap遇到ForwardingNode节点时会使用该方法来辅助转移，代码如下：

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 这里传入的tab是当前CHM的table，只有当当前处理节点是ForwaringNode时才触发辅助转移
    // 同样只有当ForwaringNode的nextTable被初始化时才能开始辅助转移，否则这个函数直接返回
    // 这种返回相当于一个自旋，因为它的调用者（例如put函数）在put失败时会死循环，会不断尝试
    // 进入这个方法来辅助转移，直到转移结束，才会继续尝试插入
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        // 下面的判断条件都是一样的
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### tryPresize

当调用put方法插入时可能会使用到的树化函数treeifyBin，它可能会调用这个函数来进行扩容：

```java
private final void tryPresize(int size) {
    // c是新的容量值
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    // 当当前阈值大于0时进行循环
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // 如果当前table为空时，在这里尝试初始化表（和initTable一致，不分析了）
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        // 如果容量太大，退出
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        // 如果当前table存在且没有被修改，那么这里作为首个线程进入扩容函数进行扩容
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (U.compareAndSetInt(this, SIZECTL, sc,
                                    (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```



好了以上就是JDK1.8以后版本中的ConcurrentHashMap的整个流程了，其中最核心的就是要理解JDK8之后，CHM取消了Segment+链表的设计，改为使用CAS+synchronized+链表+红黑树的方法来实现并发安全。里面最核心的部分就是要理解CHM支持的并发扩容机制即`transfer`函数的实现原理。