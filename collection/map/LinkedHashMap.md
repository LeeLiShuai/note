

```
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

LinkedHashMap,继承自HashMap，Entry多了before,after字段

实现了afterNodeAccess,afterNodeInsertion,afterNodeRemoval几个方法，在元素插入移除访问时执行一些操作。

可以按照插入数据遍历

可以实现LRU