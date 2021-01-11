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






