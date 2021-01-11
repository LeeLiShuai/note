

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

