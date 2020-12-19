```
public interface Map<K,V>
```

存放键值对的数据结构

```
int size();
```

键值对数量

```
boolean isEmpty();
```

是否为空

```
boolean containsKey(Object key);
```

是否包含某个键

```
boolean containsValue(Object value);
```

是否包含某个值

```
V get(Object key);
```

根据键获取

```
V put(K key, V value);
```

新增

```
V remove(Object key);
```

移除

```
void putAll(Map<? extends K, ? extends V> m);
```

新增另一个map

```
void clear();
```

清空

```
Set<K> keySet();
```

获取键的集合

```
Collection<V> values();
```

获取value的集合

```
Set<Map.Entry<K, V>> entrySet();
```

获取键值对结合

```
interface Entry<K,V>
```

内部接口

方法：

```
K getKey();
```

获取键

```
V getValue();
```

获取值

```
V setValue(V value);
```

设置值

```
boolean equals(Object o);
```

equals



```
public abstract class AbstractMap<K,V> implements Map<K,V>
```

abstractMap



```
public class IdentityHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, java.io.Serializable, Cloneable
```

只使用==判断是否相等，不适用equals的map

使用一个数组存储建和值，偶数存键，奇数存值，k=i,v=i+1

字段：

```
private static final int DEFAULT_CAPACITY = 32;
```

初始容量

```
private static final int MINIMUM_CAPACITY = 4;
```

最小容量

```
private static final int MAXIMUM_CAPACITY = 1 << 29;
```

最大容量的1/2

```
transient Object[] table;
```

存放元素

```
int size;
```

键值对数量

```
transient int modCount;
```

修改次数

```
static final Object NULL_KEY = new Object();
```

null键



构造函数：

```
public IdentityHashMap() {
    init(DEFAULT_CAPACITY);
    // table = new Object[2 * initCapacity];
}
```

无参构造函数table指向2*DEFAULT_CAPACITY长度的数组，64

```
public IdentityHashMap(int expectedMaxSize) {
    if (expectedMaxSize < 0)
        throw new IllegalArgumentException("expectedMaxSize is negative: "
                                           + expectedMaxSize);
    init(capacity(expectedMaxSize));
}
```

```
private static int capacity(int expectedMaxSize) {
    // assert expectedMaxSize >= 0;
    return
        (expectedMaxSize > MAXIMUM_CAPACITY / 3) ? MAXIMUM_CAPACITY :
        (expectedMaxSize <= 2 * MINIMUM_CAPACITY / 3) ? MINIMUM_CAPACITY :
        Integer.highestOneBit(expectedMaxSize + (expectedMaxSize << 1));
}
```

传入容量的构造函数，如果大于最大容量的1/3,则设置为最大容量*2，否则，判断是否小于最小容量的2/3，小于设置为8，否则返回大于3*倍expectedMaxSize的最小二次幂

```
public IdentityHashMap(Map<? extends K, ? extends V> m) {
    // Allow for a bit of growth
    this((int) ((1 + m.size()) * 1.1));
    putAll(m);
}
```

传入map的构造函数



方法：

```
private static int hash(Object x, int length) {
    int h = System.identityHashCode(x);
    // Multiply by -127, and left-shift to use least bit as part of hash
    return ((h << 1) - (h << 8)) & (length - 1);
}
```

计算hash值，左移一位-左移八位然后取模得到的一定是偶数

```
private static int nextKeyIndex(int i, int len) {
    return (i + 2 < len ? i + 2 : 0);
}
```

hash冲突时，取下一个key

```
public V put(K key, V value) {
		//判断是否为空，空值取NULL_KEY
    final Object k = maskNull(key);
    retryAfterResize: for (;;) {
        final Object[] tab = table;
        final int len = tab.length;
        //计算hash值
        int i = hash(k, len);
        //根据hash值判断key的地址是否相同，由于存在hash冲突，需要遍历查找，找到了直接替换
        for (Object item; (item = tab[i]) != null;
             i = nextKeyIndex(i, len)) {
            if (item == k) {
                @SuppressWarnings("unchecked")
                    V oldValue = (V) tab[i + 1];
                tab[i + 1] = value;
                return oldValue;
            }
        }
				//未找到相同的key，判断插入此元素后容量是否超过数组的1/3，超过则扩容，重新计算需要插入的位置
        final int s = size + 1;
        // Use optimized form of 3 * s.
        // Next capacity is len, 2 * current capacity.
        if (s + (s << 1) > len && resize(len))
            continue retryAfterResize;
				//插入元素
        modCount++;
        tab[i] = k;
        tab[i + 1] = value;
        size = s;
        return null;
    }
}
```

插入数据，计算hash值，找到对应位置，判断key地址是否相等，不相等+2继续判断。

容量超过1/3则扩容，重新计算地址，插入元素

```
private boolean resize(int newCapacity) {
    // assert (newCapacity & -newCapacity) == newCapacity; // power of 2
    int newLength = newCapacity * 2;
    Object[] oldTable = table;
    int oldLength = oldTable.length;
    if (oldLength == 2 * MAXIMUM_CAPACITY) { // can't expand any further
        if (size == MAXIMUM_CAPACITY - 1)
            throw new IllegalStateException("Capacity exhausted.");
        return false;
    }
    if (oldLength >= newLength)
        return false;

    Object[] newTable = new Object[newLength];

    for (int j = 0; j < oldLength; j += 2) {
        Object key = oldTable[j];
        if (key != null) {
            Object value = oldTable[j+1];
            oldTable[j] = null;
            oldTable[j+1] = null;
            int i = hash(key, newLength);
            while (newTable[i] != null)
                i = nextKeyIndex(i, newLength);
            newTable[i] = key;
            newTable[i + 1] = value;
        }
    }
    table = newTable;
    return true;
}
```

扩容，容量变为之前的两倍。遍历老数组，计算新的位置，并插入。

```
public V get(Object key) {
    Object k = maskNull(key);
    Object[] tab = table;
    int len = tab.length;
    int i = hash(k, len);
    while (true) {
        Object item = tab[i];
        if (item == k)
            return (V) tab[i + 1];
        if (item == null)
            return null;
        i = nextKeyIndex(i, len);
    }
}
```

根据key获取value,计算hash值，判断key地址是否相等，不相等+2.遇到空值(即不存在)返回null

```
public V remove(Object key) {
    Object k = maskNull(key);
    Object[] tab = table;
    int len = tab.length;
    int i = hash(k, len);

    while (true) {
        Object item = tab[i];
        if (item == k) {
            modCount++;
            size--;
            @SuppressWarnings("unchecked")
                V oldValue = (V) tab[i + 1];
            tab[i + 1] = null;
            tab[i] = null;
            closeDeletion(i);
            return oldValue;
        }
        if (item == null)
            return null;
        i = nextKeyIndex(i, len);
    }
}
```

移除元素，计算hash，找到地址相同的key，置为空。并将位置之后的相同hash值得键值对前移



```
public class EnumMap<K extends Enum<K>, V> extends AbstractMap<K, V>
    implements java.io.Serializable, Cloneable
```

key为枚举类型的map



```
public class WeakHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V> 
```

节点为弱引用的map，一次gc之后会把弱引用全部清除。

ReferenceQueue<Object>记录被gc的key。在大部分方法中根据queue删除table中的元素





```
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
```

线程安全的map,方法都被synchronized修饰，value不能为空，为空抛出空指针异常

```
public Hashtable() {
    this(11, 0.75f);
}
```

默认初始容量11



```
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
```

添加元素，值不能为空，根据hashCode计算下标，获取对应entry.遍历链表。如果hash值相等且equals则替换，否则插入新节点。

```
private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();

        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```

先判断是否扩容。赋值对应下标元素

```
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

根据hashCode获取下标，遍历判断

```
public synchronized V remove(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            modCount++;
            if (prev != null) {
                prev.next = e.next;
            } else {
                tab[index] = e.next;
            }
            count--;
            V oldValue = e.value;
            e.value = null;
            return oldValue;
        }
    }
    return null;
}
```

删除，根据hashCode获取下标，遍历判断，删除节点





public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable

HashMap

字段：

```
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

默认容量

```
static final int MAXIMUM_CAPACITY = 1 << 30;
```

最大容量

```
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

默认加载因子

```
static final int TREEIFY_THRESHOLD = 8;
```

树化阈值

```
static final int UNTREEIFY_THRESHOLD = 6;
```

树退化阈值

```
static final int MIN_TREEIFY_CAPACITY = 64;
```

最小树化容量

```
transient Node<K,V>[] table;
```

保存元素的数组

```
transient Set<Map.Entry<K,V>> entrySet;
```

```
int threshold;
```

阈值

```
final float loadFactor;
```

加载因子



构造函数：

```
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

无参构造函数



```
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

传入容量的构造函数



```
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

传入容量，加载因子的构造函数，构造新对象当时未初始化table



方法：

```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //初始化table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //计算hash值后，对应的位置没有元素，直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
    		//有元素，遍历链表或树
        Node<K,V> e; K k;
        //第一个节点符合条件，直接替换
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
            //树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
        		//遍历链表
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

put ，第一次put时，table未初始化，调用resize初始化table。



```
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

get,根据hashCode得到下标，遍历节点

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

扩容，数组容量变为之前的两倍，对于原数组中每一个链表来说，重新计算hash值之后，会分出两条链表。一条是原hash值前一位为0的，hash计算之后下标不变，一条是原hash值前一位为1的，hash计算之后为之前的length+index。





```
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

LinkedHashMap,继承自HashMap，Entry多了before,after字段

