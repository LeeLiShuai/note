public interface List<E> extends Collection<E>

List

方法：

```
int size();
```

返回集合内元素的数量

```
boolean isEmpty();
```

集合是否为空

```
boolean contains(Object o);
```

集合是否包含某一元素

```
Iterator<E> iterator();
```

获取迭代器

```
Object[] toArray();
```

转化为数组

```
<T> T[] toArray(T[] a);
```

转化为特定类型的数组

```
boolean add(E e);
```

添加元素

```
boolean remove(Object o);
```

移除元素

```
boolean containsAll(Collection<?> c);
```

是否包含另一集合内的全部元素

```
boolean addAll(Collection<? extends E> c);
```

添加另一集合的全部元素

```
boolean addAll(int index, Collection<? extends E> c);
```

从某一下标开始添加另一个的的元素

```
boolean removeAll(Collection<?> c);
```

移除另一元素内的全部元素

```
boolean retainAll(Collection<?> c);
```

是否包含另一集合的全部元素

```
E get(int index);
```

根据下标获取元素

```
E set(int index, E element);
```

设置某一下标的元素

```
void add(int index, E element);
```

某一下标开始添加元素

```
E remove(int index);
```

删除某一下标的元素

```
int indexOf(Object o);
```

返回第一个euqals(o)的下标

```
int lastIndexOf(Object o);
```

返回最后一个euqals(o)的下标



```
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>
```

abstractList

```
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

Vecotr，线程安全的List实现，几乎全部方法都有synchronized

```
public class Stack<E> extends Vector<E>
```

Stack，继承Vector,使用Vector的方法实现栈的操作





```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

数组实现List,非线程安全

字段：

```
private static final int DEFAULT_CAPACITY = 10;
```

默认初始容量10

```
private static final Object[] EMPTY_ELEMENTDATA = {};
```

空数组，传入容量为0时使用

```
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

空数组，未向List中插入元素是指向空数组

```
transient Object[] elementData;
```

存放数组元素

```
private int size;
```

元素数量

```
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

数组最大容量



构造函数：

```
public ArrayList(int initialCapacity)
```

传入容量的构造函数，容量大于0新建容量为initialCapacity的数组，等于0时elementData指向EMPTY_ELEMENTDATA,小于0时抛出IllegalArgumentException异常

```
public ArrayList()
```

无参构造函数，elements指向DEFAULTCAPACITY_EMPTY_ELEMENTDATA

```
public ArrayList(Collection<? extends E> c)
```

传入集合的构造函数，c转为数组，如果length大于0，复制元素，否则elemetData指向EMPTY_ELEMENTDATA



主要方法：

```
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

添加元素，先判断容量是否足够，然后直接在数组最后插入元素

```
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

size<10时，传入参数为10.否则参数为size+1.扩容为1.5倍，如果大于Integer.MAX-8，则设置为Integer.max-8

```
public void add(int index, E element) {
    rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

指定位置添加元素，检测index是否<0或大于size,判断是否需要扩容，index及之后的元素向后移动一位，然后index插入element.





```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

LinkedList,双向链表实现的List.



```
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

写时复制，每次写入都新建一个数组，在新数组上操作，然后指向新数组

字段：

```
final transient ReentrantLock lock = new ReentrantLock();
```

可重入锁

```
private transient volatile Object[] array;
```

保存元素的数组



构造函数：

```
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```

无参构造函数，设置array为容量为0的数组

```
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        if (c.getClass() != ArrayList.class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```

传入集合的构造函数，c转化为数组，然后复制元素

```
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```

传入数组的元素，复制元素



方法：

```
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

添加元素，申请锁，新建length+1的数组，插入元素到数组末尾，array指向新数组

```
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException("Index: "+index+
                                                ", Size: "+len);
        Object[] newElements;
        int numMoved = len - index;
        if (numMoved == 0)
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            newElements = new Object[len + 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        newElements[index] = element;
        setArray(newElements);
    } finally {
        lock.unlock();
    }
}
```

指定位置添加元素，找到位置，分成两段复制

```
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

删除指定位置的元素，申请锁，获取对应元素，复制其余元素

```
public E get(int index) {
    return get(getArray(), index);
}
```

获取指定下标的元素，直接根据下标取



COllections内部类

```
static class SynchronizedList<E>
    extends SynchronizedCollection<E>
    implements List<E>
```

线程安全的List

字段：

```
final List<E> list;
```



构造函数：

```
SynchronizedList(List<E> list) {
    super(list);
    this.list = list;
}
```

传入list的构造函数

```
SynchronizedList(List<E> list, Object mutex) {
    super(list, mutex);
    this.list = list;
}
```

传入list和锁对象的构造函数

方法使用同步代码块synchronized (mutex)



