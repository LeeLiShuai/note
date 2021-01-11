

```
public class IdentityHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, java.io.Serializable, Cloneable
```

只使用==判断是否相等，不使用equals的map

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