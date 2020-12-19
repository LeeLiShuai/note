```
public class ThreadPoolExecutor extends AbstractExecutorService
```

继承自AbstractExecutorService,AbstractExecutorService只是实现了submit方法。

将传入的runnable方法封装为FutureTask，然后调用execute方法，并返回task.



ThreadPoolExecutor的内部类：

```
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
```

Worker,具体执行线程的类

字段：

```
final Thread thread;
```

执行任务的线程

```
Runnable firstTask;
```

传入线程池的runnable,用于worker直接调用runnable的存方法

```
volatile long completedTasks;
```

已完成任务数量



构造函数：

```
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```



其他方法：

```
public void run() {
    runWorker(this);
}
```

run方法

```
public void lock()        { acquire(1); }
```

获取锁，调用aqs.acquire方法

```
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

重写aqs的tryAcquire

```
public void unlock()      { release(1); }
```

释放锁，调用aqs.release

```
protected boolean tryRelease(int unused) {
    setExclusiveOwnerThread(null);
    setState(0);
    return true;
}
```

重写aqs的tryRelease



几种拒绝策略



字段：

