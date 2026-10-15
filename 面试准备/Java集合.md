## 概念

### Java集合有哪些

Collection下的List，Set，Queue，还有Map

### fail-fast机制

https://juejin.cn/post/6879291161274482695#heading-4

为了能够提前发现并发操作导致线程安全风险。

**Java集合**中的一种**错误机制**。 在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的结构进行了修改（增加、删除），则会抛出**Concurrent Modification Exception**（并发修改异常）。

- expectedModCount是内部类迭代器的变量，modCount是集合的成员变量。

- 对集合的修改建议使用迭代器的函数，这样才不会触发并发修改异常

- forEach虽然是使用迭代器实现的，但是修改是直接调用了集合的函数，所以会触发并发修改异常。

**modCount变量**

```java
public abstract class AbstractList<E> extends AbstractCollection<E>
        implements List<E> {

    /** 记录结构性修改次数 */
    protected transient int modCount;
}
```

- 只要对列表做了“结构性修改”（插入、删除、清空、扩容等会改变元素数量或内部结构的操作），类内部的相关方法就会把 modCount加 1。

- ArrayList里的私有内部类Itr是一个非静态内部类。当迭代器创建时，它把当前的modCount 复制到expectedModCount。

- 之后的每次 `next()` / `remove()` 调用都会比较 `modCount` 与 `expectedModCount`。如果不相等，就说明集合在迭代过程中被别人改动过，迭代器会抛出 `ConcurrentModificationException`，这就是 **fail‑fast 机制** 的实现。

```java
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }
    
    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

### fail-safe

但在某些场景，我们可能想避免fail-fast机制抛出的异常，这时我们就要将ArrayList替换为使用fail-safe机制的CopyOnWriteArrayList。迭代器工作在**快照副本**或支持并发的内部结构上。即便遍历期间有增删，迭代仍继续，不抛异常，但也 **看不到（或不保证看到）最新修改**，比如利用了COW思想的读写分离。

## List

### 线程安全

#### Vector

增删改查等方法基本都加了synchronized，性能非常低下。

#### Stack

#### CopyOnWriteArrayList

读取操作是完全无需加锁的。更加厉害的是，写入操作也不会阻塞读取操作，只有写写才会互斥，互斥是通过**ReentrantLock**对修改操作加锁，相当于**只会生成一个副本**。

缺点：内存开销大，复制副本消耗，数据不一致。

Copy-On-Write写时复制：当需要修改（ add，set、remove等操作） CopyOnWriteArrayList的内容时，不会直接修改原数组，而是会先创建底层数组的副本，对副本数组进行修改，修改完之后再将修改后的数组赋值回去，这样就可以保证写操作不会影响读操作了。

### 非线程安全

**ArrayList：**初始10，1.5倍扩容

**LinkedList：**

## Set

### 线程安全

CopyOnWriteArraySet：是通过“动态数组(CopyOnWriteArrayList)”实现的，并不是散列表实现的。

ConcurrentSkipListSet

### 非线程安全

HashSet：基于HashMap，元素放在HashMap的 key 里，value 统一放一个静态常量哨兵对象（常见名字PRESENT），并不是“不存值”，而是都存同一个占位对象。

LinkedHashSet

TreeSet

## Queue / Deque

### 线程安全

**ConcurrentLinkedQueue**

**ConcurrentLinkedDeque**

#### LinkedBlockingQueue

锁是分离的，即生产用的是`putLock`，消费是`takeLock`，这样可以防止生产者和消费者线程之间的锁争夺。

#### ArrayBlockingQueue

**生产者-消费者**模型实现的并发容器

有容量限制。使用一个**ReentrantLock锁**保护环形数组，既保证线程安全，又避免多锁开销。有两个Condition实现线程间的等待和唤醒操作。ReentrantLock类结合Condition实例可以实现选择性通知。

put/take是**阻塞式**增加/删除（没有位置就等待），add/offer/remove/poll是**非阻塞式**增加/删除（没有就返回错误）。

```java
public void put(E e) throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)       // 满
            notFull.await();                // -> 等待 notFull
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)                  // 空
            notEmpty.await();               // -> 等待 notEmpty
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private void enqueue(E e) {
    items[putIndex] = e;
    if (++putIndex == items.length) putIndex = 0;
    count++;
    notEmpty.signal();                      // 唤醒可能在等的消费者
}

private E dequeue() {
    E e = items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) takeIndex = 0;
    count--;
    notFull.signal();                       // 唤醒可能在等的生产者
    return e;
}
```

**LinkedBlockingDeque**

**PriorityBlockingQueue**

#### DelayQueue

用于实现定时任务调度和缓存过期删除等场景。

要使用，首先需要继承 `Delayed` 实现 `DelayedTask`，实现 `getDelay` 方法以及优先级比较 `compareTo`。

使用了PriorityQueue，通过使用 ReentrantLock和Condition解决线程安全，确保单位时间内只有一个线程可以操作延迟队列。使用 CAS 操作实现元素的添加和移除操作。使用了并发设计模式的Leader-Follower模式，当没有任务到期时，leader等待堆顶元素的过期时间，而follower一直等待。

数据结构：`ReentrantLock lock`、`Condition available`、`volatile Thread leader`、最小堆 `q`。

take()：

1. 看堆顶剩余延时 `delay`；
2. **有 leader**：其他线程 `available.await()` 无限期等；
3. **无 leader**：当前线程设为 **leader**，`awaitNanos(delay)` 精确等到期；
4. 返回元素前把 `leader = null`；**若队列非空，`signal()` 唤醒一个等待者**当新 leader。

offer(e)：

若 `e` 成为**新的最早到期**（堆顶），则 `leader = null; available.signal()`，让被唤醒的线程接班按更短延时计时。

![image-20260823223921632](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20260823223921632.png)

**SynchronousQueue**

**LinkedTransferQueue**

### 非线程安全

ArrayDeque

PriorityQueue

LinkedList

------

## Map

### 线程安全

>  从HashTable的锁整张表，到1.7的分段锁，到1.8的桶级锁

#### HashTable

每个方法加synchronized，不允许null，初始容量11，扩容2n+1，不会树化。

#### ConcurrentHashMap1.7

分段存储，哈希值该怎么处理？高位 n 位 ，决定Segment（分段锁，降低锁粒度）。低位m位，决定段内桶索引。每个段内的table数组长度是单独扩容的。

Segment的数量怎么确定？可以传入参数，默认是16，确定之后就固定了，16就是线程并发数。

用到锁的两个地方：

- **CAS**：1. 如果相应位置的 Segment 还未初始化，则通过 CAS 进行赋值。2. 向空桶插入首结点使用CAS。

- **ReentrantLock**（段级）：“悲观”地保护涉及链表或数组结构变化的操作：插入、删除节点，rehash，clear……

其中Segment<K,V> 本身继承ReentrantLock，因此拿锁非常廉价（对象里直接有锁状态字段）。

```java
static final class Segment<K,V> extends ReentrantLock {
    volatile HashEntry<K,V>[] table;
    int count;          // 需要锁保护
    int modCount;       // 需要锁保护
    ...
}
```

#### ConcurrentHashMap1.8

> 单表 + 广泛CAS + 桶级 `synchronized`；多线程并行迁移 + ForwardingNode；高冲突树化至 O(log n)；整个Map的数量计数用 LongAdder 思路。

1. 结构去分段

- 取消 Segment：改为单一 `Node[] table`，整体结构更像 `HashMap`。
- 定位与容量：数组长度始终为 2 的幂，位运算寻址；构造器里的 `concurrencyLevel` 仅作容量估算提示，不再分段。

2. 并发控制方式变化

- get 全程无锁：靠 `volatile` 读与有序写保证可见性。
- put/replace 删除：
  - 桶为空：用 CAS（对数组槽位）直接放入首节点；
  - 桶非空：对该桶头节点加 `synchronized` 做局部加锁（粒度比 1.7 的段锁更细）。
- 哈希扰动：`spread()` 混合高低位，降低冲突。
- **CAS使用在初始化，并行迁移，计数**

3. 扩容/迁移可并行

- 以 `sizeCtl` 统一控制初始化与阈值；
- 触发扩容时，多个线程按 stride 分块并行搬迁；
- 迁移过的桶放入 `ForwardingNode`（hash = MOVED/-1） 占位，读操作遇到它会跳到 `nextTable`，无需全表停顿。

4. 冲突退化的优化（树化）

- 链表长度 ≥ `TREEIFY_THRESHOLD = 8` 时转为 红黑树（`TreeBin`/`TreeNode`），最坏查找从 O(n) 降到 O(log n)；
- 当表容量尚小（< `MIN_TREEIFY_CAPACITY = 64`）时优先扩容而不是树化；
- 链表长度降到 `UNTREEIFY_THRESHOLD = 6` 会退回链表。

5. 计数更抗争用

- 由段计数改为 `baseCount + CounterCells`（类似 `LongAdder`），减少 `size` 更新热点；
- `size()`/`mappingCount()` 为弱一致近似值（在并发下可能非瞬时精确，但非常接近）。

6. 迭代与新 API

- 迭代器/视图 弱一致（不抛 `ConcurrentModificationException`）；
- 支持 `compute\*`/`merge`/`forEach`/`search`/`reduce` 等函数式与批量操作（必要时可借助公共 ForkJoin 线程池并行），映射函数在桶锁保护下运行；
- 仍不允许 `null` 键/值。

**ConcurrentSkipListMap**

**Properties**

### 非线程安全

#### HashMap

长度是 2 的幂次方，扩容也是两倍扩，计算位置时hash%length可以用hash&(length-1)替换，位运算效率高，且让 HashMap在扩容的时候更均匀。1.8转红黑树条件，链表长度大于8且数组长度大于等于64时。初始数组长16，扩容的负载因子0.75。退化成链表条件，删除或者扩容节点数小于6。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

#### LinkedHashMap

> 多了一组双向链表指针

排序模式

accessOrde默认为 false，也就是按照插入顺序链接。若为 true，也就是按照访问顺序，可以用于实现LRU（需要重写removeEldestEntry方法）。

同样使用迭代器遍历，相比HashMap的更快，因为HashMap还要搜索下一个非空bucket。

节点源码

```java
// JDK 21 中的简化示意
abstract class HashMap<K,V> {
    static class Node<K,V> {             // 基础链表节点
        final int hash;  final K key;
        V value;         Node<K,V> next;
    }

    static final class TreeNode<K,V> extends Node<K,V> {  // 红黑树节点
        TreeNode<K,V> parent, left, right, prev;
        boolean red;
    }
}

class LinkedHashMap<K,V> extends HashMap<K,V> {

    // ① 链表桶用的节点
    static class Entry<K,V> extends Node<K,V> {
        Entry<K,V> before, after;        // ***插入 / 访问顺序链***
    }

    // ② 冲突过多而树化后的节点
    static final class LinkedTreeNode<K,V>
            extends TreeNode<K,V> {      // 保留红黑树字段 → parent/left/right…
        LinkedTreeNode<K,V> before, after; // 仍要维护顺序链
    }

    // 双向链表的表头/表尾哨兵
    transient Entry<K,V> head, tail;
}
```

#### TreeMap

#### WeakHashMap

#### IdentityHashMap