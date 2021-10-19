---
layout: post
title: ConcurrentLinkedQueue
categories: 并发
keywords: ConcurrentLinkedQueue, JUC
---

### JUC集合：ConcurrentLinkedQueue

ConcurrentLinkedQueue是一个没有使用锁的并发链表数据结构，它内部通过大量使用CAS+自旋的方式实现了线程安全的添加、删除操作。其内部表示结点的核心类为`Node`，就是通过大量使用CAS的方式实现线程安全，同时再使用直接内存操作，使得整体的效率提升。CLQ内部使用了自定义的Node结点来完成CAS设值的。

> 本文引用了https://blog.csdn.net/qq_38293564/article/details/80798310以及https://www.pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentLinkedQueue.html的相关内容和图片

#### Node

```java
static final class Node<E> {
    volatile E item;
    volatile Node<E> next;

    // CAS赋值
    Node(E item) {
        ITEM.set(this, item);
    }

    // 初始化一个空节点
    Node() {}

    void appendRelaxed(Node<E> next) {
        NEXT.set(this, next);
    }

    boolean casItem(E cmp, E val) {
        return ITEM.compareAndSet(this, cmp, val);
    }
}
```

可以看到Node类采用的都是VarHandle赋值或者CAS赋值，这样的好处就是效率高，允许失败，并发量也大。相比使用锁或者同步机制而已，它的效率绝对是更高的。

#### 属性和构造函数

CLQ类只维护了两个属性，一个是头结点，另一个是尾结点（忽略如版本号这些属性以及VarHandle属性）：

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {
  transient volatile Node<E> head;
  private transient volatile Node<E> tail;
}
```

而CLQ的构造函数也很简单，它支持无参构造函数初始化一个空的头结点；也支持从任意集合初始化一个链表：

```java
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>();
}

public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        Node<E> newNode = new Node<E>(Objects.requireNonNull(e));
        if (h == null)
          	// 如果是头结点，那么让头尾指针同时指向这个结点
            h = t = newNode;
        else
          	// 否则挂到尾结点之后即可
            t.appendRelaxed(t = newNode);
    }
    if (h == null)
      	// 如果给出的是一个空集合，那么初始化一个空结点即可
        h = t = new Node<E>();
    head = h;
    tail = t;
}
```

#### 元素添加

CLQ的元素添加比较复杂，这是因为它有以下几个特点：

1. 不允许null值加入队列，因为CLQ删除结点时就是通过将value设置为null实现的
2. 这里的head和tail不一定指向真正的头结点和尾结点，为了提高效率，头尾结点的设置有滞后性

接下来我们先来看源码，再画图来分析整个入队过程：

```java
public boolean offer(E e) {
    // 创建新结点
    final Node<E> newNode = new Node<E>(Objects.requireNonNull(e));
    // 自旋，直到结点成功插入位置
    for (Node<E> t = tail, p = t;;) {
        // 这里t表示当前的尾结点（尾结点可能发生变化，这里保存旧的尾结点）
        // p用来指示t是否是真正的尾结点，如果不是真正的尾结点，那么p会向后移动去寻找真正的尾结点
        Node<E> q = p.next;
        //
        if (q == null) {
            // q是p的下一个结点，如果q为空，说明t就是尾结点
            // 那么直接尝试将新结点挂到p之后即可
            if (NEXT.compareAndSet(p, null, newNode)) {
                // p等于t说明t就是真正的尾结点，此时不需要更新尾结点，从而提高效率
                // p不等于t说明t不是真正的尾结点，因为t在循环中始终表示旧的尾结点
                // p表示寻找的真正的尾结点，如果二者不相同，说明上一次加入队尾没有更新尾结点
                // 那么这里要尝试更新尾结点
                if (p != t) // hop two nodes at a time; failure is OK
                    // 这个更新操作可以失败，因为运行到这里添加结点的任务已经完成了，这里CAS失败意味着
                    // 有其它线程更新了尾结点，那么其它线程必然在当前结点之后，所以这里失败无所谓
                    // 其它线程会更新到最新的状态的
                    TAIL.weakCompareAndSet(this, t, newNode);
                // 更新成功则退出
                return true;
            }
            // 否则说明设置尾结点CAS竞争失败，需要自旋重新尝试
        }
        // 这个p等于q的条件说明，当前结点p和它的下一个结点q是一样的，也就是存在自我连接
        // 出现这个情况只有可能存在于删除时，头结点链接到了自己来辅助GC
        // 如果出现这个情况那么需要重新寻找头结点，这个新的头结点之后才是有效的结点
        // 并从这个头结点出发去寻找尾结点
        // 这种情况可能发生在队列只有一个结点时，那个结点被删除导致的，此时插入的新结点就会遇到这个问题
        else if (p == q)
            // 这里先判断尾结点有没有发生更新，如果发生了更新，那么有可能这种自连接已经被清除
            // 否则就要从当前头结点去找有效的尾结点了
            p = (t != (t = tail)) ? t : head;
        else
            // 如果也不存在自连接，也不是队尾，那么就要去寻找真正的队尾了
            // 如果当前指针p已经和保存的尾结点t不一样了但是还是运行到了这里就说明可能存在竞争，即第一个if失败或内部CAS失败
            // 那么这个时候尾结点可能已经被其它线程更新了，这里要判断一下，如果被更新了那么就要取新尾结点尝试插入
            // 否则的情况不可能发生，只有可能前面的 p != t失败，说明p指向的尾结点不是真正的尾结点，而q才是（跳数一定为1）
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

这个过程相当复杂，原因就在于CLQ尾结点tail指示的不一定是真正的尾结点这个问题，但是设计上要么tail是尾结点要么tail.next是尾结点，即跳数只能为1。所以在整个流程中，如果tail是尾结点，那么就尝试直接插入，这个时候不修改tail；否则的话要判断是不是自连接，自连接是因为对头结点删除时，可以通过自连接告诉其它结点这个头结点已经失效了，并且也可以辅助GC，如果是头结点那么就考虑重新寻找头结点（如果此时尾结点变了，说明这件事已经由其它线程做完了，只要去取新的尾结点重复这个过程即可）；否则的话说明存在一个跳跃，那么让p后移一位即可，如果出现跳跃两次的问题，那么就要考虑是不是其它线程修改了尾结点，还是一样重新取尾结点即可，否则就取下一个结点作为尾结点重新循环即可。

上面的说法很复杂，我们用一个例子来说明这件事：

![CLQ-Offer-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/CLQ-Offer-Process.png)

很明显可以看到的是，tail总是隔一个更新一次，这样做是为了避免反复CAS更新尾结点带来的效率损失，但是这样也增加了整个流程的复杂度。这个例子没有体现 p == q的情况，这个情况，我们放到删除元素时再讨论。

#### 元素删除

元素的删除操作和添加类似，head结点的更新也存在滞后性：并不是每次删除时都更新head节点，当head节点里有元素时，直接弹出head节点里的元素，而不会更新head节点。只有当head节点里没有元素时，删除操作才会更新head节点。采用这种方式也是为了减少使用CAS更新head节点的消耗，从而提高出队效率。下面先来看整个流程的示意图：

![CLQ-Poll-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/CLQ-Poll-Process.png)

再来看源码：

```java
public E poll() {
    // 自旋，当头结点发生变化时重启外部循环
    restartFromHead: for (;;) {
        // 内部自旋
        for (Node<E> h = head, p = h, q;; p = q) {
            final E item;
            // 如果当前头结点没有被删除并且成功将当前头结点设置为空即删除状态
            if ((item = p.item) != null && p.casItem(item, null)) {
                // 同理如果头结点和真正的头结点已经不一样了，那么要更新头结点
                // 原始的头结点是h，如果当前头结点p的下一个结点非空，那么就让下一个结点作为头结点
                // 否则说明没有其它节点了，就让p作为一个空的头结点
                // ((q = p.next) != null) ? q : p 可以改写成 (p.next != null) ? p.next : p 更好理解
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            } else if ((q = p.next) == null) {
                // 如果头结点元素为空或者CAS竞争失败有其他线程删除了元素，那么要这里判断一下队列是否还有元素
                // 如果没有则把头结点后移一位，这里h和p是有可能一样的，如果只剩空头结点的时候尝试删除就是一样的
                updateHead(h, p);
                return null;
            } else if (p == q)
                // 最后如果当前结点p和它的下一个结点一样，说明被删除了，需要重新寻找头结点重新开始
                continue restartFromHead;
          	// 最后在内层for循环里更新p，令p=q相当于p=p.next
        }
    }
}
```

删除的流程也很简单，需要判断当前头结点head到底是不是真正的头结点，有可能head的item是null，表示的就是被删除的结点，此时真正的头结点就是head.next。如果发现真正的头结点和head不一致，那么就需要CAS更新头结点。同时如果当前队列为空，那么就要考虑更新头结点，这里来看更新头结点的方法`updateHead`：

```java
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && HEAD.compareAndSet(this, h, p))
        NEXT.setRelease(h, h);
}
```

很简单，先判断要更新的头结点和旧的头结点是否一样，如果一样才会更新；然后用CAS更新head结点为p；最后给旧头结点h的next结点设置一个自连接。所以上面元素添加和删除可能出现的 `p == q` 原因都来自这里的自连接设置。

我们考虑一下这种情况：

![CLQ-Special-Process](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Concurrent/CLQ-Special-Process.png)

当前head指向被删除的结点，tail指向倒数第二个结点，也就是head和tail同时出现滞后性的这种情况。这时候如果我们尝试去删除头结点，那么会导致头结点移动到当前真实头结点的下一个结点，也就是head指针会越过tail指针，因为在`poll`方法中没有对tail指针做过任何修改。此时会调用`updateHead(h, ((q = p.next) != null) ? q : p);`代码，也就是`updateHead(h, q)`。这个时候原始头结点h的next结点即当前tail指示的结点就多了一个自连接。所以为了解决这种问题，需要在`offer`中判断一下 `p == q`。

同样，在并发删除时，如果有两个线程同时获取到了一样的head指针，在一个完成删除后也会出现上述情况，所以需要进行 `p == q`的判断。

#### 元素获取

`Queue`接口中定义了从队列头获取元素的方法`peek`，这个方法和上面删除的最大区别就是它只获取头元素，而不删除。其源码如下：

```java
public E peek() {
    restartFromHead: for (;;) {
        for (Node<E> h = head, p = h, q;; p = q) {
            final E item;
            // 如果当前head有值，或者当前head已经是最后一个节点了即队列空
            if ((item = p.item) != null || (q = p.next) == null) {
                // 这里可以尝试更新head结点到非空结点
                // 如果head结点就是队列头结点，那么不需要更新，此时 h == p
                // 否则才更新head到最靠前的结点
                updateHead(h, p);
                return item;
            }
            else if (p == q)
                // head结点被修改过了，重新获取head结点
                continue restartFromHead;
        }
    }
}
```

几乎一模一样，就是少了删除结点即将结点值设置为null的语句。但是无论如何，只要头结点指向的不是真实的头结点，那么都会尝试更新头结点。

#### 获取表长

获取表长需要遍历整个链表，同时也不是线程安全的：

```java
public int size() {
    restartFromHead: for (;;) {
        int count = 0;
        for (Node<E> p = first(); p != null;) {
            if (p.item != null)
                if (++count == Integer.MAX_VALUE)
                    break;  // @see Collection.size()
            if (p == (p = p.next))
                // 如果头结点发生变化，需要重新统计
                continue restartFromHead;
        }
        return count;
    }
}
```

可以看到，时间复杂度是O(n)，原因就是为了避免出现头结点变化导致的长度问题。但是由于这种统计没有加锁，所以仍然会存在线程不安全的问题，如果在统计时发生一些不会导致自连接的添加和删除，那么是不会从头开始统计的。

#### 指定元素删除

指定元素删除方法`remove`和`poll`的区别就在于，它删除的结点位置可能不一定在头部，可能是队列内部的某个结点。这种结点被删除之后就需要重新调整队列链表的前后连接关系，其源码如下：

```java
public boolean remove(Object o) {
    if (o == null) return false;
    restartFromHead: for (;;) {
        // p是当前结点，pred是当前结点的前驱结点
        for (Node<E> p = head, pred = null; p != null; ) {
            // q是当前结点的后继结点
            Node<E> q = p.next;
            final E item;
            // 如果当前结点非空，那么就需要判断是不是这个结点
            // 如果是那么需要删除这个结点，否则继续往下找
            if ((item = p.item) != null) {
                if (o.equals(item) && p.casItem(item, null)) {
                    skipDeadNodes(pred, p, p, q);
                    return true;
                }
                pred = p; p = q; continue;
            }
            // 如果当前结点为空，说明这个结点已经被删除了，那么需要移除这个结点
            for (Node<E> c = p;; q = p.next) {
                // q是当前结点的下一个结点，如果下一个结点是空节点或者有效结点
                // 那么直接链接到下一个结点即可，否则需要继续寻找下一个有效结点或末尾
                if (q == null || q.item != null) {
                    pred = skipDeadNodes(pred, c, p, q); p = q; break;
                }
                // 处理并发添加问题
                if (p == (p = q)) continue restartFromHead;
            }
        }
        // 如果没找到对应结点，即当前结点 p == null，退出了上面循环
        return false;
    }
}
```

还是比较清晰的，如果是一个有效结点，那么就需要判断和目标结点是否一致，如果一致则CAS删除（赋值为null），然后利用`skipDeadNodes`跳过无效结点；如果是无效结点，那么就需要找到下一个有效结点或者队尾，调整链接关系。其中`skipDeadNodes`源码如下：

```java
// pred: 是上一个已知的存活结点
// c   : 第一个无效结点
// p   : 最后一个无效结点 （如果只有一个无效结点，那么可以 c = p）
// q   : 下一个有效结点，可以为空，实际上和 p.next 一致
private Node<E> skipDeadNodes(Node<E> pred, Node<E> c, Node<E> p, Node<E> q) {
    // 如果下一个结点是空节点
    if (q == null) {
        // 如果只有一个无效结点，那么不做任何修改，防止tail指针出现问题
        if (c == p) return pred;
        // 否则将下一个有效结点设置为最后一个无效结点，原因一样防止tail指针出现问题
        q = p;
    }
    // 如果CAS设置下一个有效结点成功，则返回pred
    // 否则如果CAS失败或者pred被删除，那么返回最后一个无效结点p
    return (tryCasSuccessor(pred, c, q)
            && (pred == null || ITEM.get(pred) != null))
        ? pred : p;
}
```

逻辑很简单，就是要注意一点，并不是所有无效结点都能删除，如果无效结点处于队尾，那么就不能删除，防止tail指针游离。

#### HOPS(延迟更新的策略)的设计

通过上面对offer和poll方法的分析，我们发现tail和head是延迟更新的，两者更新触发时机为：

- `tail更新触发时机`：当tail指向的节点的下一个节点不为null的时候，会执行定位队列真正的队尾节点的操作，找到队尾节点后完成插入之后才会通过casTail进行tail更新；当tail指向的节点的下一个节点为null的时候，只插入节点不更新tail。
- `head更新触发时机`：当head指向的节点的item域为null的时候，会执行定位队列真正的队头节点的操作，找到队头节点后完成删除之后才会通过updateHead进行head更新；当head指向的节点的item域不为null的时候，只删除节点不更新head。

并且在更新操作时，源码中会有注释为：`hop two nodes at a time`。所以这种延迟更新的策略就被叫做HOPS的大概原因是这个，从上面更新时的状态图可以看出，head和tail的更新是“跳着的”即中间总是间隔了一个。那么这样设计的意图是什么呢?

如果让tail永远作为队列的队尾节点，实现的代码量会更少，而且逻辑更易懂。但是，这样做有一个缺点，如果大量的入队操作，每次都要执行CAS进行tail的更新，汇总起来对性能也会是大大的损耗。如果能减少CAS更新的操作，无疑可以大大提升入队的操作效率，所以doug lea大师每间隔1次(tail和队尾节点的距离为1)进行才利用CAS更新tail。对head的更新也是同样的道理，虽然，这样设计会多出在循环中定位队尾节点，但总体来说读的操作效率要远远高于写的性能，因此，多出来的在循环中定位尾节点的操作的性能损耗相对而言是很小的。

#### ConcurrentLinkedQueue适合的场景

ConcurrentLinkedQueue通过无锁来做到了更高的并发量，是个高性能的队列，但是使用场景相对不如阻塞队列常见，毕竟取数据也要不停的去循环，不如阻塞的逻辑好设计，但是在并发量特别大的情况下，是个不错的选择，性能上好很多，而且这个队列的设计也是特别费力，尤其的使用的改良算法和对哨兵的处理。整体的思路都是比较严谨的，这个也是使用了无锁造成的，我们自己使用无锁的条件的话，这个队列是个不错的参考。