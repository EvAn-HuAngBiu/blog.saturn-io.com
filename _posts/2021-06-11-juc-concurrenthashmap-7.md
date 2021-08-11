---
layout: post
title: ConcurrentHashMap 1.7
categories: 并发
keywords: ConcurrentHashMap, JUC
---

# ConcurrentHashMap（1.7）

ConcurrentHashMap在Java1.7及之前版本是通过Segment+HashEntry数组（桶）+链表的方法保证线程安全的，其相对于HashTable的一个优势就在于它加锁的粒度更细，它每次只给一个Segmeng加锁，所以Segment的个数（初始化后就不可修改）就代表了当前ConcurrentHashMap的并发数，如下图所示：

<img src="/images/Concurrent/ConcurrentHashMapOverallV7.png" alt="ConcurrentHashMapOverallV7" style="zoom:50%;" />

源码中使用了`concurrentLevel`来表示Segment的个数，默认是 16，也就是说 ConcurrentHashMap 有 16 个 Segments，所以理论上，这个时候，最多可以同时支持 16 个线程并发写，只要它们的操作分别分布在不同的 Segment 上。这个值可以在初始化的时候设置为其他值，但是一旦初始化以后，它是不可以扩容的。

## HashEntry

HashEntry是ConcurrentHashMap的最小存储单元，它包括四个字段：hash、key、value和next指针，需要注意的是这个类它大多数使用Unsafe类提供的放来设置属性值。源码如下：

```java
static final class HashEntry<K, V> {
  final int hash;
  final K key;
  volatile V value;
  volatile HashEntry<K, V> next;

  HashEntry(int hash, K key, V value, HashEntry<K, V> next) {
    this.hash = hash;
    this.key = key;
    this.value = value;
    this.next = next;
  }

  final void setNext(HashEntry<K, V> n) {
    // 使用Unsafe类，根据next属性在内存中的偏移地址来设置值
    UNSAFE.putOrderedObject(this, nextOffset, n);
  }

  static final sun.misc.Unsafe UNSAFE;
  static final long nextOffset;

  static {
    try {
      UNSAFE = sun.misc.Unsafe.getUnsafe();
      Class k = HashEntry.class;
      // 获取next属性在内存中的偏移地址
      nextOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("next"));
    } catch (Exception e) {
      throw new Error(e);
    }
  }
}
```

## Segment

Segment是HashEntry的上层结构，它以数组的形式被保存在ConcurrentHashMap中，同时Segment类有保存了HashEntry数组，这也是ConcurrentHashMap的整个数据组织结构。由于Segment的类定义时**继承了ReentrantLock**，所以可以看出ConcurrentHashMap加锁操作的粒度变为了Segment。

下面是源码中定义的`Segment`类的字段：

```java
static final Class Segment<K, V> extends ReentrantLock implements Serializable {
  private static final long serialVersionUID = 2249069246763182397L;
  
  static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
  
  transient volatile HashEntry<K,V>[] table;
  
  transient int count;
  
  transient int modCount;
  
  transient int threshold;
  
  final float loadFactor;
}
```

其中`MAX_SCAN_RETRIES`变量规定了后续自旋锁的最大自旋次数，单核1次，多核64次；同时要注意`count`、`modCount`变量没有`volatile`修饰，所以使用时要加锁或采用其它方式读才能保证它的可见性；同样`threshold`和`loadFactor`变量是整个Segment公用的，读取时需要注意。

然后我们逐个分析Segment类中的方法：

### scanAndLockForPut

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    // retries用来标记以下两种情况：
    // retries=-1时：要么是第一次进入还没有完成插入结点的初始化，要么是当前Entry的链表被修改要重新创建结点的情况
    // retries>=0时：表明正在自旋重试中，重试次数大于MAX_SCAN_RETRIES时就会加锁阻塞
    int retries = -1;
    while (!tryLock()) {
         HashEntry<K,V> f;
         if (retries < 0) {
             if (e == null) {
                 // 如果进入到这里说明有以下几种可能: 1.找到最后了还是找不到包含key的结点 2. 刚开始找的时候就找不到对应的结点
                 if (node == null)
                     // 这里判断node为空是为了避免重复创建
                     node = new HashEntry<K,V>(hash, key, value, null);
                 retries = 0;
             }
             else if (key.equals(e.key))
                 // 遍历时找到了我们要插入的位置（覆盖插入）
                 retries = 0;
             else
                 // 否则继续向后找
                 e = e.next;
             }
         else if (++retries > MAX_SCAN_RETRIES) {
              // 超过最大自旋次数，退出
              lock();
              break;
         }
         else if ((retries & 1) == 0 &&
                  (f = entryForHash(this, hash)) != first) {
               // 当检查到其它线程修改了当前链表时（体现在重新获取的当前桶的链表表头与刚才保存的表头不一致）
               e = first = f; // 修改first，重新从头寻找插入位置
               retries = -1;
         }
   }
   return node;
}
```

这个方法的作用是为了提前寻找需要put插入的entry的位置。当然`Segment`类中还有一个`scanAndLock`函数，它是专门用于remove和replace操作中的，其原理和这个函数基本一致，这里不再分析。下面来看插入函数put:

### put

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  // 这里会先尝试抢锁，如果抢到锁了则返回null，反之则进入scanAndLockForPut方法
  // 这里只能说可能会拿到一个已经初始化好的node结点，但只是可能，因为tryLock肯定返回null，
  // 而scanAndLockForPut方法也可能返回null
  HashEntry<K,V> node = tryLock() ? null :
       scanAndLockForPut(key, hash, value);
  V oldValue;
  try {
    HashEntry<K,V>[] tab = table;
    // 寻找当前插入node的key对应在哪个桶里（这里和HashMap一致）
    int index = (tab.length - 1) & hash;
    // 这里是利用Unsafe类来直接读取table中对应index位置上的链表首结点
    HashEntry<K,V> first = entryAt(tab, index);
    for (HashEntry<K,V> e = first;;) {
      if (e != null) {
        // e表示当前正在遍历的链表结点，非空表示当前结点有值，要么和当前的key一致，如果一致则替换旧值
        // 否则继续遍历链表的下一个结点
        K k;
        if ((k = e.key) == key ||
            (e.hash == hash && key.equals(k))) {
          oldValue = e.value;
          if (!onlyIfAbsent) {
            // 如果允许覆盖旧值的话覆盖，并更新操作计数器，用于并发时的快速失败
            e.value = value;
            ++modCount;
          }
          break;
        }
        // 否则寻找下一个
        e = e.next;
      }
      else {
        // e为空表示要么当前key在链表中找不到，那么只要把当前的key对应的节点作为头结点即可（头插法）
        // 要么就是当前链表完全为空，那么就把当前值初始化为新的链表结点即可
        // 这里判断了node是否为空，如果node不为空就说明在scanAndLockForPut函数中已经完成了上述两种判断，只要将其插入末尾即可
        // 反之则说明node结点并没有被创建，则要重新创建node节点
        if (node != null)
          // 即将当前的头结点first挂到node结点的next字段上
          node.setNext(first);
        else
          // 构造函数第四个参数就是初始化next字段的（这里用first变量来初始化）
          node = new HashEntry<K,V>(hash, key, value, first);
        int c = count + 1;
        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
          //扩容是直接重新new一个新的HashEntry数组，这个数组的容量是老数组的两倍，
          //新数组创建好后再依次将老的table中的HashEntry插入新数组中，所以这个过程是十分费时的，应尽量避免。
          //扩容完毕后，还会将这个node插入到新的数组中。
          rehash(node);
        else
          // 同样调用Unsafe类来设置table数组第index位置上的node值，这个方法里用的是UNSAFE.putOrderedObject
          // 因为它使用的是StoreStore屏障，而不是十分耗时的StoreLoad屏障
          // 给我个人感觉就是putObjectVolatile是对写入对象的写入赋予了volatile语义，但是代价是用了StoreLoad屏障 ?
          // 而putOrderedObject则是使用了StoreStore屏障保证了写入顺序的禁止重排序，但是未实现volatile语义导致更新后的不可见性 ?
          setEntryAt(tab, index, node);
        ++modCount;
        count = c;
        oldValue = null;
        break;
       }
    }
   } finally {
     unlock();
   }
   return oldValue;
}
```

### remove

remove操作的原理并不复杂，下面来看源码：

```java
final V remove(Object key, int hash, Object value) {
    // remove操作必须加锁
    if (!tryLock())
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        // 获取key下标对应Entry的首个节点e
        HashEntry<K,V> e = entryAt(tab, index);
        HashEntry<K,V> pred = null;
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            // 这里要求key和value值都命中才算找到了对应节点
            if ((k = e.key) == key || (e.hash == hash && key.equals(k))) {
                V v = e.value;
                if (value == null || value == v || value.equals(v)) {
                    // 这里的pred就是当前节点的前一个节点（这里就是链表节点的删除）
                    // 如果删除的是首节点，那么就把下一个节点设置为新的首节点
                    if (pred == null)
                        setEntryAt(tab, index, next);
                    else
                        // 反之就把前一个节点直接链接到下一个节点（就是链表删除）
                        pred.setNext(next);
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e;
            e = next;
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```



### rehash

扩容函数，将Segment中HashEntry的数组容量扩大至原来的2倍

```java
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 新容量是原来的2倍
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 计算当前节点在新数组中的下标
            int idx = e.hash & sizeMask;
            if (next == null)   //  处理链表只有一个节点的情况
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                // 这里先试图寻找一些在新数组中位置不发生变化的节点，lastIdx变量保存不发生变化的第一个节点的下标
                // 这里的原理和HashMap一致，举个例子假设HashEntry数组容量为16，那么下标为7的链表中可能包含hash值为7、14、21、28、35、42等
                // 节点，当这个HashEntry数组被扩容到32时，显然hash值为14、21、28的节点会被放置到新位置上，而hash值为35、42、49的节点则
                // 不发生任何变动还是挂在下标为7的数组上，所以下面这个for循环就是找到这种节点的开头，这里就是hash值为35的节点将它直接挂到
                // 下标为7的结点之后即可，当然上面已经处理了hash值为7的节点这种初始情况了
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 不发生变化的节点直接挂上去即可
                newTable[lastIdx] = lastRun;
                // 处理剩余节点，只需要将刚才处理的节点作为循环的末尾即可，这里是一种优化
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // 添加当前待添加的节点
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

## ConcurrentHashMap

### 常量

下面将上述两种结构融合起来，来看整个ConcurrentHashMap的源码，先来分析它的常量：

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
  implements ConcurrentMap<K, V>, Serializable {
  
  private static final long serialVersionUID = 7249069246763182397L;
  
  // 默认初始化容量
  static final int DEFAULT_INITIAL_CAPACITY = 16;
  
  // 默认负载因子
  static final float DEFAULT_LOAD_FACTOR = 0.75f;
  
  // 默认并发度，也就是默认Segment的个数，它直接决定了整个map的并发度，因为Segment是加锁的最小单位
  static final int DEFAULT_CONCURRENCY_LEVEL = 16;
  
  // 最大容量
  static final int MAXIMUM_CAPACITY = 1 << 30;
  
  // 每个Segment的最小容量
  static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
  
  // 最大分段数，即Segment数组的最大长度
  static final int MAX_SEGMENTS = 1 << 16; // slightly conservative
  
  // 默认自旋次数，超过这个次数直接加锁，防止在size方法中由于不停有线程在更新map
  // 导致无限的进行自旋影响性能，当然这种会导致ConcurrentHashMap使用了这一规则的方法
  // 如size、clear是弱一致性的。
  static final int RETRIES_BEFORE_LOCK = 2;
  
  // 用于索引segment的掩码值，key哈希码的高位用于选择segment
  final int segmentMask;
  
  // 索引segment的偏移量
  final int segmentShift;
  
  final Segment<K,V>[] segments;

  transient Set<K> keySet;
  transient Set<Map.Entry<K,V>> entrySet;
  transient Collection<V> values;
}
```

### 构造函数

构造函数分析，这里只分析参数最全的一个，其它构造函数都是通过使用默认参数来调用这个构造函数的：

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    // 非法参数筛查
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
      // 如果当前并发度超过了分段数组最大大小，那么会自动缩小并发度（并发度和Segment数组长度是一个概念）
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // 如果给出的并发度不是2的次幂，那么会寻找一个大于该并发度的2的幂，sshift表示偏移量，ssize表示最终的并发度
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // segmentShift表示，前sshift位为Segment大小，即key的hash码右移segmentShift位和下面掩码相与的值即对应的桶的位置
    this.segmentShift = 32 - sshift;
    // mask就是比如Segment大小为16，那掩码二进制就为1111（十进制15），说明只要将key的hash值与mask相与（相当于取余）就能得到当前key应该在哪个Segment里了
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 注意，默认初始化容量initialCapacity是全部Segment可以分到的总和，所以这里将总数除以Segment个数得到每个Segment的Entry数组长度
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 每个Segment中Entry数组最小长度，如果这个长度小于给定的initialCapacity那么会扩大2倍直到大于为止
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // 初始化Segment数组的首个元素（只初始化这一个），Segment中Entry数组的长度为cap
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    // 构造Segment数组，长度为ssize（即并发度）
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // 利用Unsafe设置数组第一个元素的值
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

### put

接着来看ConcurrentHashMap层面的put方法：

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    // j就是segment数组的下标，先右移拿到前32-shift位，再与mask相与（就是取余）得到下标
    int j = (hash >>> segmentShift) & segmentMask;
    // 这里判断对应下边的Segment是否被初始化，利用的是UNSAFE直接访问对应内存位置
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    // 实际调用的还是Segment的put方法，上面分析过了
    return s.put(key, hash, value, false);
}
```

### ensureSegment

这个函数的目的是保证当前访问的Segment被初始化了，如果没有初始化则初始化它。因为在构造函数里可以看出只有第一个Segment被初始化了，其它Segment都没有被初始化，这个初始化过程留到了第一次使用Segment的时候进行，源码如下：

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // 获取下标对应的内存偏移
    Segment<K,V> seg;
    // 如果没被初始化才会进入下面的if，否则直接返回被初始化后的Segment结果
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0]; // 使用下标0的Segment为参数原型
        // 读取第0个Segmen的各种参数，包括Entry数组容量、负载因子，后续就用这些参数来初始化其它Segment
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // 重新检查，防止重复创建（这里是没有获取Segment锁的）
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 利用CAS设值，如果已经被其它线程初始化或者当前线程设值成功则退出返回
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

### get

接着分析get方法：

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 检查对应下标的Segment不为空，并且对应HashEntry数组也不为空
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
          // 遍历寻找即可
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

过程不解释，非常简单，根据Segment数组位置和Entry数组位置找到对应Entry遍历链表查询即可。这里要注意的一点是这个方法没有加任何的锁，所以要考虑和put、remove操作并存时的并发安全问题。