interface Queue<E> extends Collection<E>

队列，一般先进先出FIFO.

方法：

```
boolean add(E e);
```

将指定元如插入队列，在不违反容量限制的情况下返回true，如果当前没有可用空间，抛出illeglasteException异常

```
boolean offer(E e);
```

往队列中添加一个元素，在不违反容量限制的情况下返回true， 如果因为空间限制，无法添加元素则，返回false；

```
E remove();
```

删除队列头元素，如果队列为空，抛出异常

```
E poll();
```

删除队列头元素，如果队列为空，返回null

```
E element();
```

返回队列头元素，不删除，为空抛出异常

```
E peek();
```

返回队列头元素，不删除，为空返回null



abstract class AbstractQueue<E> 

​			extends AbstractCollection<E>

​				implements Queue<E>】

queue的实现

```
public boolean add(E e)
```

调用offer(e)，成功返回true,失败抛出IllegalStateException异常

```
public E remove()
```

调用poll(),不为空，返回poll结果，为空抛出NoSuchElementException异常

```
public E element()
```

调用peek(),不为空返回peek结果，为空抛出NoSuchElementException异常

```
public void clear()
```

while(poll()!=null),循环调用poll,直到队列为空

```
public boolean addAll(Collection<? extends E> c)
```

c为null抛出NullPointerException异常，c==this抛出IllegalArgumentException异常

遍历c,调用add(e)





 interface Deque<E> extends Queue<E>

双向队列，可以再两端操作的队列

```
void addFirst(E e);
```

队列的头部插入元素，成功返回true，超过容量抛出异常

```
void addLast(E e);
```

队列的尾部插入元素，成功返回true，超过容量抛出异常

```
boolean offerFirst(E e);
```

队列的头部插入元素，成功返回true，超过容量时，有的子类和addFirst一样有的返回false

```
boolean offerLast(E e);
```

队列的尾部插入元素，成功返回true，超过容量时，有的子类和addFirst一样有的返回false

```
E removeFirst();
```

移除队列头部元素，队列为空时，抛出异常

```
E removeLast();
```

移除队列尾部元素，队列为空时，抛出异常

```
E pollFirst();
```

移除头部元素，为空时，返回null

```
E pollLast();
```

移除尾部元素，为空时，返回null

```
E getFirst();
```

获取头部元素，为空抛出NoSuchElementException异常

```
E getLast();
```

获取头部元素，为空抛出NoSuchElementException异常

```
E peekFirst();
```

获取头部元素，为空返回null

```
E peekLast();
```

获取尾部元素，为空返回null

```
boolean removeFirstOccurrence(Object o);
```

从头部开始遍历删除第一个与o相等的元素

```
boolean removeLastOccurrence(Object o);
```

从尾部开始遍历删除第一个与o相等的元素,如果存在就删除，返回true.不存在返回false





class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable

数组实现的队列

字段：

```
transient Object[] elements;
```

存放元素的数组

```
transient int head;
```

头元素的下标

```
transient int tail;
```

尾元素右边一个元素的下标，总是指向下一个可被插入的下标

```
private static final int MIN_INITIAL_CAPACITY = 8;
```

最小容量



私有方法：

```
private static int calculateSize(int numElements)
```

返回比numElements大的最小的2次幂

```
private void allocateElements(int numElements)
```

初始化elements数组，调用calculateSize，长度为2次幂

```
private void doubleCapacity()
```

讲队列容量扩从为当前的两倍

新建一个长度为之前两倍的数组，然后将元素拷贝过来，赋值给element

```
private <T> T[] copyElements(T[] a)
```

将队列的元素拷贝到a中



构造函数：

```
public ArrayDeque() {
    elements = new Object[16];
}
```

无参构造函数，默认容量16

```
public ArrayDeque(int numElements) {
    allocateElements(numElements);
}
```

传入容量的构造函数，大小最小为8，大于8则为大于参数的最小2次幂

```
public ArrayDeque(Collection<? extends E> c) {
    allocateElements(c.size());
    addAll(c);
}
```

传入集合参数的构造函数，先初始化，然后addAll



方法：

```
    public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        //相当于elements[head-1]=e,处理了为-1的情况，-1时变为length-1
        elements[head = (head - 1) & (elements.length - 1)] = e;
        if (head == tail)
            doubleCapacity();
    }
```

e为空抛出异常，head,tail初始化时为0.addFirst默认从数组末尾开始插入。插入完成后如果头尾相等则扩容

```
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```

扩容，r为head右侧的元素数量。先将head右侧的怨妇复制到新数组，从下标0开始，然后复制head左侧的元素，下标从r开始。维持队列的顺序，head为0，tail为原数组length

```
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```

尾部添加元素，直接element[tail],因为tail为尾元素的下一个下标，然后判断tail是否和head重合，重合则扩容

```
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
```

头部添加元素，调用addFirst,e不为null时必定返回true。

```
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

尾部添加元素，调用addLast,e不为null时必定返回true。

```
public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    E result = (E) elements[h];
    // Element is null if deque empty
    if (result == null)
        return null;
    elements[h] = null;     // Must null out slot
    head = (h + 1) & (elements.length - 1);
    return result;
}
```

头部移除，返回elements[head],然后置空，head-1

```
public E pollLast() {
    int t = (tail - 1) & (elements.length - 1);
    @SuppressWarnings("unchecked")
    E result = (E) elements[t];
    if (result == null)
        return null;
    elements[t] = null;
    tail = t;
    return result;
}
```

尾部移除，返回elements[tail-1],tail-tail-1.

```
public E removeFirst() {
    E x = pollFirst();
    if (x == null)
        throw new NoSuchElementException();
    return x;
}
```

头部移除，调用pollFirst

```
public E removeLast() {
    E x = pollLast();
    if (x == null)
        throw new NoSuchElementException();
    return x;
}
```

尾部移除，调用pollLast

```
public E getFirst() {
    @SuppressWarnings("unchecked")
    E result = (E) elements[head];
    if (result == null)
        throw new NoSuchElementException();
    return result;
}
```

获取头元素，返回elements[head],为空抛出异常

```
public E getLast() {
    @SuppressWarnings("unchecked")
    E result = (E) elements[(tail - 1) & (elements.length - 1)];
    if (result == null)
        throw new NoSuchElementException();
    return result;
}
```

获取尾元素，返回elements[tail-1],为空抛出异常

```
public E peekFirst() {
    // elements[head] is null if deque empty
    return (E) elements[head];
}
```

获取头元素，返回elements[head]，为空返回null

```
public E peekLast() {
    return (E) elements[(tail - 1) & (elements.length - 1)];
}
```

获取尾元素，为空返回null

```
public boolean removeFirstOccurrence(Object o) {
    if (o == null)
        return false;
    int mask = elements.length - 1;
    int i = head;
    Object x;
    while ( (x = elements[i]) != null) {
        if (o.equals(x)) {
            delete(i);
            return true;
        }
        i = (i + 1) & mask;
    }
    return false;
}
```

从head开始遍历，删除第一个equals的元素

```
public boolean removeLastOccurrence(Object o) {
    if (o == null)
        return false;
    int mask = elements.length - 1;
    int i = (tail - 1) & mask;
    Object x;
    while ( (x = elements[i]) != null) {
        if (o.equals(x)) {
            delete(i);
            return true;
        }
        i = (i - 1) & mask;
    }
    return false;
}
```

从tail开始遍历，删除第一个equals的元素



queue相关的方法尾进头出





```
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable
```

优先队列，小顶堆实现，数组标识堆。

left = 2*parent+1

right = 2*parent+2

Parent = (child -1)/2

每个节点都不大于两个子节点

字段：

```
private static final int DEFAULT_INITIAL_CAPACITY = 11;
```

默认容量11

```
transient Object[] queue
```

保存元素的数组

```
private int size = 0
```

元素个数

```
private final Comparator<? super E> comparator
```

比较器

```
transient int modCount = 0;
```

修改次数

构造函数：

```
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}
```

无参构造函数，默认容量11

```
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
```

传入容量的构造函数

```
public PriorityQueue(Comparator<? super E> comparator) {
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}
```

传入比较器的构造函数

```、
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

传入容量和比较器的构造函数

```
public PriorityQueue(Collection<? extends E> c)
```

传入集合的构造函数

```
public PriorityQueue(PriorityQueue<? extends E> c) {
    this.comparator = (Comparator<? super E>) c.comparator();
    initFromPriorityQueue(c);
}
```

传入自己的实例的构造函数

```
public PriorityQueue(SortedSet<? extends E> c) {
    this.comparator = (Comparator<? super E>) c.comparator();
    initElementsFromCollection(c);
}
```

传入SortedSet的构造函数

方法：

```
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        siftUp(i, e);
    return true;
}
```

添加元素，先判断是否需要扩容，如果是第一个元素，就插入到queue[0],否则调用siftUp.



```
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // Double size if small; else grow by 50%
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    // overflow-conscious code
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}
```

扩容，小于64时，每次+2，否则扩容为1.5倍



```
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```

元素上浮，k为堆末尾下标，从k开始判断与父节点的关系，如果小于则交换位置，大于等于则停止



```
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);
    return result;
}
```

从队列中取出一个元素，即堆顶元素queue[0]，然后将堆尾元素放到堆顶下沉

```
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    int half = size >>> 1;        // loop while a non-leaf
    while (k < half) {
    		//左叶子节点
        int child = (k << 1) + 1; // assume left child is least
        Object c = queue[child];
        //右叶子节点
        int right = child + 1;
        //c为左右节点中更小的那个
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        //key小于等于两个子节点
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```

下沉，k为0，对比k和k的两个子节点，与更小的那个子节点交换。直到小于两个子节点

```
private E removeAt(int i) {
    // assert i >= 0 && i < size;
    modCount++;
    int s = --size;
    if (s == i) // removed last element
    		//是对位元素
        queue[i] = null;
    else {
        E moved = (E) queue[s];
        queue[s] = null;
        //用最后一个元素替换要删除的元素然后下沉
        siftDown(i, moved);
        //下沉没有移动，说明是叶子节点，需要上浮操作
        if (queue[i] == moved) {
            siftUp(i, moved);
            if (queue[i] != moved)
                return moved;
        }
    }
    return null;
}
```

删除某个元素
