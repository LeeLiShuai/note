

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

ArrayList,数组实现的List

字段：

```
private static final int DEFAULT_CAPACITY = 10;
默认容量10
```

```
private static final Object[] EMPTY_ELEMENTDATA = {};
初始化时的数组为空
```

```
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
默认容量的空数组
```

```
transient Object[] elementData;
存储数据的数组
```

```
private int size;
list内元素数量
```



构造函数： 

```
无参构造函数，初始化时，数组为空数组，add元素时才将数组重新赋值为default_capacity容量的数组
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

```
传入容量的构造函数
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

传入collection的构造函数



方法：

```
添加元素，先判断数组容量是否足够，不足的扩容为1.5倍
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

```
//指定位置添加元素，判断下标是否越界，是否容量足够，然后数据向后移动一位，插入
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

```
//检验下标，然后返回数组中对应下标的元素
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```

```
//移除指定下标的元素，检验下标，获取元素，移动下标之后的数据，下标对应元素设置为null
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

```
//根据值移除元素，遍历删除
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```