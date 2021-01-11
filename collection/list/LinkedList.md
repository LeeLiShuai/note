

```
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

LinkedList,双向链表实现的List.



字段：

```
transient int size = 0;
元素数量
```

```
transient Node<E> first;
头结点
```

```
transient Node<E> last;
尾结点
```



构造函数：

```
空的构造函数
public LinkedList() {
}
```

```
使用collection初始化的构造函数
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```



方法：

```
添加元素，添加到链表末尾
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

```
指定位置添加元素，检验位置，找到对应位置的当前元素，插入到当前元素元素前面
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}
```

```
链表头部添加元素
public void addFirst(E e) {
    linkFirst(e);
}
```

```
根据下标获取元素，检验下标，然后遍历
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

```
根据下标获取对应节点，判断属于前半部分还是后半部分，然后遍历查找
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

```
获取某元素的下标，遍历查找
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```