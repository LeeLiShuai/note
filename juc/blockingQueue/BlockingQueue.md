

```
public interface BlockingQueue<E> extends Queue<E>
```

阻塞队列

```
boolean add(E e);
```

添加元素，容量足够返回true,容量不足抛IllegalStateException异常

```
boolean offer(E e);
```

添加元素，容量足够返回true,返回false

```
void put(E e) throws InterruptedException;
```

添加元素，容量不足时阻塞

```
boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException;
```

添加元素，容量不足时，等待一段时间

```
E take() throws InterruptedException;
```

取出元素，为空时阻塞

```
E poll(long timeout, TimeUnit unit)
    throws InterruptedException;
```

取出元素，为空时等待一段时间







```
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable 
```

数组实现的阻塞队列，一个可重入锁，两个条件Condition

字段：

```
final Object[] items;
```

存放元素的数组

```
int takeIndex;
```

下一个取出的下标

```
int putIndex;
```

下一个存入的下标

```
int count;
```

元素数量

```
final ReentrantLock lock;
```

锁

```
private final Condition notEmpty;
```

取出的条件

```
private final Condition notFull;
```

存入的条件



构造方法：

```
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}
```

传入容量的构造方法，默认非公平锁

```
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

传入容量和是否公平

```
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c)
```

传入容量，是否公平，集合的构造方法



方法：

```
public boolean add(E e) {
    return super.add(e);
}
```

添加元素，调用AbstractQueue.add()，实际还是调用自己的offer();插入失败抛出异常

```
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```

添加元素，获取锁，如果队列满了，返回false

```
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

添加元素，获取锁，可中断，如果队列满了，阻塞。然后入队

```
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```

入队，直接在putIndex插入元素，如果插入之后，putIndex超出数组便捷，则设为0(此时队列不一定满了，可能数组前部的元素被取出，且所有调用入队的地方，都以判断队列未满)。然后通知notEmpty，可以出队



```
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
```

出队，数量为0返回null,不为0调用dequeue()

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

出队，数量为0阻塞，不为0调用dequeue()

```
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return x;
}
```

出队操作，获取takeIndex对应的元素，置为空。takeIndex+1,如果和数组长度相等，置为0,通知notFull。



```
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
```

链表实现的阻塞队列

字段：

```
private final int capacity;
```

容量

```
private final AtomicInteger count = new AtomicInteger();
```

元素数量

```
transient Node<E> head;
```

头结点，空节点，存储数据

```
private transient Node<E> last;
```

尾结点

```
private final ReentrantLock takeLock = new ReentrantLock();
```

出队锁

```
private final Condition notEmpty = takeLock.newCondition();
```

非空条件

```
private final ReentrantLock putLock = new ReentrantLock();
```

入队锁

```
private final Condition notFull = putLock.newCondition();
```

非满条件



构造函数：

```
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
```

无参构造函数，容量为int最大值

```
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}
```

传入容量的构函数

```
public LinkedBlockingQueue(Collection<? extends E> c)
```

传入集合的构造函数

方法：

```
public int size() {
    return count.get();
}
```

获取元素个数，直接获取AtomicInteger的值

```
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

插入元素，继承自AbstractQueue,调用offer

```
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

不能插入null值，判断容量是否足够，获取插入锁，入队，count<容量，通知notFull

如果是第一次插入，通知notEmpty

```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
  
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```

插入元素，获取插入锁，count,容量不足时阻塞。入队，count<容量，通知notFull

如果第一次插入，通知notEmpty

```
public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

出队，继承自AbstractQueue,调用poll

```
public E poll() {
    final AtomicInteger count = this.count;
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) {
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

出队,队列为空返回null,获取取出锁，dequeue。出队后count>0时，通知notEmpty。

通知notFull

```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

出队，队列为空时阻塞，出队后count>0时，通知notEmpty。

通知notFull

```
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
```

移除head,返回head.next的值，然后head.next置为空，并设置为head

```
public E peek() {
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}
```

查询头元素，返回头几点下一个元素的值



```
public class LinkedBlockingDeque<E>
    extends AbstractQueue<E>
    implements BlockingDeque<E>, java.io.Serializable
```

链表实现的双端阻塞队列

字段：

```
transient Node<E> first;
```

头结点，存储数据

```
transient Node<E> last;
```

尾结点，存储数据

```
private transient int count;
```

节点数量

```
private final int capacity;
```

最大容量

```
final ReentrantLock lock = new ReentrantLock();
```

锁

```
private final Condition notEmpty = lock.newCondition();
```

非空条件

```
private final Condition notFull = lock.newCondition();
```

非满条件



构造函数：

```
public LinkedBlockingDeque() {
    this(Integer.MAX_VALUE);
}
```

无参构造函数，最大容量为int最大值

```
public LinkedBlockingDeque(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
}
```

传入容量的构造函数

```
public LinkedBlockingDeque(Collection<? extends E> c) 
```

传入集合的构造函数



方法：

```
public void addFirst(E e) {
    if (!offerFirst(e))
        throw new IllegalStateException("Deque full");
}
```

头部插入，调用offerFirst

```
public boolean offerFirst(E e) {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return linkFirst(node);
    } finally {
        lock.unlock();
    }
}
```

头部插入，调用linkFirst

```
public void putFirst(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        while (!linkFirst(node))
            notFull.await();
    } finally {
        lock.unlock();
    }
}
```

头部插入，插入失败阻塞，调用linkFirst

```
private boolean linkFirst(Node<E> node) {
    // assert lock.isHeldByCurrentThread();
    if (count >= capacity)
        return false;
    Node<E> f = first;
    node.next = f;
    first = node;
    if (last == null)
        last = node;
    else
        f.prev = node;
    ++count;
    notEmpty.signal();
    return true;
}
```

容量超过，返回false，头插法插入链表，++count,通知notEmpty

```
public void addLast(E e) {
    if (!offerLast(e))
        throw new IllegalStateException("Deque full");
}
```

尾部插入，调用offerLast

```
public boolean offerLast(E e) {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return linkLast(node);
    } finally {
        lock.unlock();
    }
}
```

尾部插入，调用linkLast

```
public void putLast(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    Node<E> node = new Node<E>(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        while (!linkLast(node))
            notFull.await();
    } finally {
        lock.unlock();
    }
}
```

尾部插入，插入失败阻塞,调用linkLast

```
private boolean linkLast(Node<E> node) {
    // assert lock.isHeldByCurrentThread();
    if (count >= capacity)
        return false;
    Node<E> l = last;
    node.prev = l;
    last = node;
    if (first == null)
        first = node;
    else
        l.next = node;
    ++count;
    notEmpty.signal();
    return true;
}
```

尾插法插入链表，通知notEmpty

```
public E removeFirst() {
    E x = pollFirst();
    if (x == null) throw new NoSuchElementException();
    return x;
}
```

移除头部节点，调用pollFirst

```
public E pollFirst() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return unlinkFirst();
    } finally {
        lock.unlock();
    }
}
```

移除头部节点，调用unlinkFirst

```
public E takeFirst() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E x;
        while ( (x = unlinkFirst()) == null)
            notEmpty.await();
        return x;
    } finally {
        lock.unlock();
    }
}
```

移除头部节点，返回为空则阻塞，调用unlinkFirst

```
private E unlinkFirst() {
    // assert lock.isHeldByCurrentThread();
    Node<E> f = first;
    if (f == null)
        return null;
    Node<E> n = f.next;
    E item = f.item;
    f.item = null;
    f.next = f; // help GC
    first = n;
    if (n == null)
        last = null;
    else
        n.prev = null;
    --count;
    notFull.signal();
    return item;
}
```

头节点为空返回空，删除头结点，--count,通知notFull

```
public E removeLast() {
    E x = pollLast();
    if (x == null) throw new NoSuchElementException();
    return x;
}
```

移除尾结点，调用pollLast

```
public E pollLast() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return unlinkLast();
    } finally {
        lock.unlock();
    }
}
```

移除尾结点，调用unlinkLast

```
public E takeLast() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E x;
        while ( (x = unlinkLast()) == null)
            notEmpty.await();
        return x;
    } finally {
        lock.unlock();
    }
}
```

移除尾部节点，返回为空则阻塞，调用unlinkLast

```
private E unlinkLast() {
    // assert lock.isHeldByCurrentThread();
    Node<E> l = last;
    if (l == null)
        return null;
    Node<E> p = l.prev;
    E item = l.item;
    l.item = null;
    l.prev = l; // help GC
    last = p;
    if (p == null)
        first = null;
    else
        p.next = null;
    --count;
    notFull.signal();
    return item;
}
```

尾节点为空返回空，删除尾结点，--count,通知notFull

Queue相关操作，增加元素操作调用last,删除元素操作调用first





```
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable
```

优先级阻塞队列

字段：

```
private static final int DEFAULT_INITIAL_CAPACITY = 11;
```

默认容量11

```
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

最大容量，int最大值-8

```
private transient Object[] queue;
```

保存元素的数组

```
private transient int size;
```

元素数量

```
private transient Comparator<? super E> comparator;
```

比较器

```
private final ReentrantLock lock;
```

锁

```
private final Condition notEmpty;
```

非空条件，优先队列会自动扩容，不需要非满条件

```
private transient volatile int allocationSpinLock;
```

扩容用的字段



构造函数：

```
public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
```

无参构造函数函数

```
public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}
```

传入容量的构造函数

```
public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}
```

传入容量和比较器的构造函数

```
public PriorityBlockingQueue(Collection<? extends E> c) 
```

传入集合的构造函数



方法：

```
public boolean add(E e) {
    return offer(e);
}
```

新增元素，调用offer

```
public void put(E e) {
    offer(e); // never need to block
}
```

新增元素，调用offer

```
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```

新增元素，不能插入null,获取锁，判断是否需要扩容，上浮。通知非空

```
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : // grow faster if small
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

扩容，先释放锁，因为只有offer调用此方法，offer加了锁，将扩容锁字段置为1，容量<64时，为之前的两倍+2，否则为1.5倍。数组重新赋值扩容锁字段置为0，获取lock，复制元素

siftUp,siftDown和优先队列一样

```
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

poll出队，调用dequeue.

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```

take出队，如果dequeue为空，则阻塞

```
private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0];
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}
```

dequeue,size<0，返回null,和优先队列一样的操作



```
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E>
```

延迟队列，内部是一个优先队列，只能存放实现了Delayed接口的对象，对象到期后才能从队列中取走。根据Delayed中comparTo的实现，决定堆顶是最小还是最大

字段：

```
private final transient ReentrantLock lock = new ReentrantLock();
```

锁

```
private final PriorityQueue<E> q = new PriorityQueue<E>();
```

优先队列

```
private Thread leader = null;
```

头线程,用于优化唤醒线程

```
private final Condition available = lock.newCondition();
```

条件



构造函数：

```
public DelayQueue() {}
```

无参构造函数

```
public DelayQueue(Collection<? extends E> c) {
    this.addAll(c);
}
```

传入参数的构造函数



方法：

```
public boolean add(E e) {
    return offer(e);
}
```

add，调用offer

```
public void put(E e) {
    offer(e);
}
```

put,调用offer

```
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e);
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

offer.获取锁，调用优先队列的offer，如果是入队后是堆顶元素，则leader设置为空，通知条件

```
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll();
    } finally {
        lock.unlock();
    }
}
```

出队，poll,获取锁，获取堆顶元素，如果为空，或者delay>0,返回空，否则直接返回堆顶元素

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return q.poll();
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

出队，take,获取锁，获取堆顶元素，如果为空，阻塞。否则获取delay。如果小于零，返回。>0时，如果leader不为空，阻塞。为空则设置为当前贤臣。然后阻塞delay的时间



```
static class DelayedWorkQueue extends AbstractQueue<Runnable>
    implements BlockingQueue<Runnable> 
```

ScheduledThreadPoolExecutor的内部类，延迟队列。存放元素只能是RunnableScheduledFuture



字段：

```
private static final int INITIAL_CAPACITY = 16;
```

默认容量16

```
private RunnableScheduledFuture<?>[] queue =
    new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
```

存放元素的数组

```
private final ReentrantLock lock = new ReentrantLock();
```

锁

```
private int size = 0;
```

元素个数

```
private Thread leader = null;
```

头线程

```
private final Condition available = lock.newCondition();
```

条件



构造函数：

只有默认无参构造函数



方法：

```
public boolean add(Runnable e) {
    return offer(e);
}
```

入队add,调用offer

```
public void put(Runnable e) {
    offer(e);
}
```

入队put,调用offer

```
public boolean offer(Runnable x) {
    if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length)
            grow();
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
            siftUp(i, e);
        }
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
```

offer，元素为空抛出异常，强转为RunnableScheduledFuture，获取lock,判断是否扩容，

如果是第一入队的元素，queue[0]=e,否则插入末尾，siftUp。如果插入后，堆顶元素为e，设置leader为空，通知条件。

```
public RunnableScheduledFuture<?> poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        RunnableScheduledFuture<?> first = queue[0];
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return finishPoll(first);
    } finally {
        lock.unlock();
    }
}
```

poll出队，获取锁，获取第一个元素，即栈顶元素，如果为空，或者delay>0返回空，否则返回

```
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

take,获取锁，获取堆顶元素，如果为空，阻塞，获取delay,<0直接返回，否则，如果头线程不为空，阻塞，不为空设置当前线程设置为leader，阻塞delay时间

