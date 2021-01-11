```
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable
```

并发的map

内部类：

```
//普通链表节点，valatile修饰的value和执行的下一个节点
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

		//key和value相等则相等
    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**
     * Virtualized support for map.get(); overridden in subclasses.
     * 根据hashcode和key查找元素
     */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```



```
static final class TreeBin<K,V> extends Node<K,V>
```

树节点，连接着红黑树的根节点。封装了操作红黑树的方法



```
static final class TreeNode<K,V> extends Node<K,V>
```

红黑树里面的节点



```
static final class ForwardingNode<K,V> extends Node<K,V>
```

扩容用到的节点，hashcode为-1



```
static final class ReservationNode<K,V> extends Node<K,V>
```

保留节点，部分方法会用到，hashcode为-3



字段：

```
//保存数据的数组
transient volatile Node<K,V>[] table;
```

```
//扩容后的新数组
private transient volatile Node<K,V>[] nextTable;
```

```
/**
 * 控制table的初始化和扩容.
 * 0  : 初始默认值
 * -1 : 有线程正在进行table的初始化
 * >0 : table初始化时使用的容量，或初始化/扩容完成后的threshold
 * -(1 + nThreads) : 记录正在执行扩容任务的线程数
 */
private transient volatile int sizeCtl;
```

```
//计数基值,当没有线程竞争时，计数将加到该变量上。类似于LongAdder的base变量
private transient volatile long baseCount;
```

```
//计数数组，出现并发冲突时使用。类似于LongAdder的cells数组
private transient volatile CounterCell[] counterCells;
```

```
//扩容时需要用到的一个下标变量.
private transient volatile int transferIndex;
```

```
//自旋标识位，用于CounterCell[]扩容时使用。类似于LongAdder的cellsBusy变量
private transient volatile int cellsBusy;
```



方法：

```
//插入一个元素
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
		//key和value都不能为空
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //table为空，初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //对应位置没有元素，直接进行cas设置值，成功后直接跳出循环
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //如果节点的hash值为-3，说明正在进行扩容，协助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //已初始化，对应位置有值，不再扩容，则插入链表末尾或者红黑树中
        else {
            V oldVal = null;
            //锁住头结点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                		//hash值大于零，正常节点。遍历链表替换或插入末尾
                    if (fh >= 0) {
                        binCount = 1;
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
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }//处理树节点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }//binCount用于统计链表中有多少节点，是否要树化
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                //如果是替换，返回值，不增加count
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //元素数量+1
    addCount(1L, binCount);
    return null;
}
```



```
//查找方法
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    //如果存在元素
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //第一个就是
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //特殊节点，调用对应的find方法查询
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```



```
//返回元素数量
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```



扩容的几个入口

链表树化，插入元素后addCount