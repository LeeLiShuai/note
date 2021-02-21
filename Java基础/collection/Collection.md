Collection<E> extends Iterable<E>

集合

```
int size();
```

集合内元素个数

```
boolean isEmpty();
```

集合是否为空

```
boolean contains(Object o);
```

集合是否包含某元素

```
Iterator<E> iterator();
```

获取集合的迭代器

```
Object[] toArray();
```

集合转化为Object数组

```
<T> T[] toArray(T[] a);
```

集合转化为指定类型的数组

```
boolean add(E e);
```

添加元素

```
boolean remove(Object o);
```

删除元素

```
boolean containsAll(Collection<?> c);
```

是否包含所有元素

```
boolean addAll(Collection<? extends E> c);
```

添加所有元素

```
boolean removeAll(Collection<?> c);
```

移除所有元素

```
void clear();
```

清空集合

//Java8新增默认方法





abstract AbstractCollection<E> implements Collection<E>

Collection的实现,实现了部分方法

```
public boolean isEmpty()
```

rturn size() ==0

```
public boolean contains(Object o)
```

调用iterator()获取迭代器，遍历集合调用equals方法判断是否相等，相等return true，遍历结束return fasle

```
public Object[] toArray()
```

新建length为size()的Object数组，获取迭代器，遍历数组，copy元素。

```
public boolean add(E e)
```

抛出UnsupportedOperationException异常

```
public boolean remove(Object o) 
```

获取迭代器，遍历判断是否equals,相等调用it.remove()，return true,遍历结束return false.

```
public boolean containsAll(Collection<?> c)
```

遍历c,内部调用contains(e),不包含返回false,遍历结束返回true.

```
public boolean addAll(Collection<? extends E> c) 
```

遍历c,内部调用add.

```
public boolean removeAll(Collection<?> c)
```

遍历c,内部调用remove

```
public boolean retainAll(Collection<?> c)
```

遍历c,内部调用contains,如果包含就remove

```
public void clear()
```

获取迭代器，it.remove()

