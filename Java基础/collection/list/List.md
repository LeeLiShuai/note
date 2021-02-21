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









