字段：

```
private final Sync sync;
```

内部实现的同步类，继承AQS

主要调用AQS state相关方法。

内部定义Sync类，继承AQS,重写tryAcquireShared(int),tryReleaseShared(int)





构造函数：

```
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

传入count。内部初始化sync，设置AQS的state



方法：

```
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

内部类重写的方法

```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

等待，调用aqs的acquireSharedInterruptibly，功能和acquireShared类似，

内部先调用tryAcquireShared,如果返回值<0(当state!=0是一种成立),就调用doAcquireSharedInterruptibly,功能和doAcquireShared类似，包装一个共享节点插入队列中。

```
public void countDown() {
    sync.releaseShared(1);
}
```

计数，调用aqs的releaseShared，先调用tryReleaseShared(内部类重写的)，将state-1.

返回status是否为0.为0则调用doReleaseShared，将一个节点唤醒，下个节点被唤醒后

