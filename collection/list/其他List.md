```
private static class ArrayList<E> extends AbstractList<E>
    implements RandomAccess, java.io.Serializable
```

Arrays.ArrayList



字段：

```
private final E[] a;
保存数据的数组
```



构造函数：

```
根据数组初始化的构造函数
ArrayList(E[] array) {
    a = Objects.requireNonNull(array);
}
```

不能新增元素，只能替换







```
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

Vecotr，线程安全的List实现，几乎全部方法都有synchronized



```
public class Stack<E> extends Vector<E>
```

Stack，继承Vector,使用Vector的方法实现栈的操作 



Collections内部类

```
static class SynchronizedList<E>
    extends SynchronizedCollection<E>
    implements List<E>
```

线程安全的List

字段：

```
final List<E> list;
```



构造函数：

```
SynchronizedList(List<E> list) {
    super(list);
    this.list = list;
}
```

传入list的构造函数

```
SynchronizedList(List<E> list, Object mutex) {
    super(list, mutex);
    this.list = list;
}
```

传入list和锁对象的构造函数

方法使用同步代码块synchronized (mutex)





