```
public class LongAdder extends Striped64
```

高并发情况下代替AtomicLong,解决高并发环境下**AtomicLong**的自旋瓶颈问题

AtomicLong中使用volatile long value保存数值，所有的操作都针对这个变量进行，多个线程竞争同一个热点。

LongAdder就是分散热点，将value分散到一个数组中，不同线程命中数组中不同的元素，对对应的元素进行操作，分散热点。

```
//cpu核心数
static final int NCPU = Runtime.getRuntime().availableProcessors();

//保存数据的数组
transient volatile Cell[] cells;

//计数时，先尝试修改base,有竞争时采取操作cells
transient volatile long base;

//调整单元格大小和/或创建单元格时使用的自旋锁
transient volatile int cellsBusy;
```



方法：

```
//增加计数
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    //cells不为空或者修改base失败
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        //如果cells为空，或者cells长度为0，或者对应的元素为空，修改对应数组元素失败。执行longAccumulate方法初始化cells数组
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```



```
//返回累加的和，当前的计数值。遍历cells累加。没有加锁返回的是近似值
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

