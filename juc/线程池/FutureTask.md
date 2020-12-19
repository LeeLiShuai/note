```
public interface Future<V>
```

Future接口

方法：

```
boolean cancel(boolean mayInterruptIfRunning);
```

取消操作

```
boolean isCancelled();
```

判断任务是否已被取消

```
boolean isDone();
```

任务是否已完成

```
V get() throws InterruptedException, ExecutionException;
```

获取任务执行结果，阻塞至执行结束

```
V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
```

获取任务执行结果，等待一段时间，时间结束抛出异常



```
public interface RunnableFuture<V> extends Runnable, Future<V>
```

RunnableFuture，简单的继承Runnable,Future,与Future相比只多了run方法



```
public class FutureTask<V> implements RunnableFuture<V>
```

FutureTask,实现RunnableFurture接口



字段：

```
volatile int state
```

线程状态字段

```
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

线程状态的具体值。

对象初始化时为NEW状态，可能的状态转化有

 NEW -> COMPLETING -> NORMAL
 NEW -> COMPLETING -> EXCEPTIONAL
 NEW -> CANCELLED
 NEW -> INTERRUPTING -> INTERRUPTED

```
private Callable<V> callable;
```

初始化时传入的callable参数

```
private Object outcome;
```

get()返回结果

```
private volatile Thread runner;
```

执行run方法的线程，通过cas设置

```
private volatile WaitNode waiters;
```

调用get()被阻塞的线程



```
private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset;
private static final long runnerOffset;
private static final long waitersOffset;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        stateOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("state"));
        runnerOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("runner"));
        waitersOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

修改statae,runner,waiters的方法



构造函数：

```
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```

传入callable的构造函数，直接给this.callable赋值



```
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

传入runable的构造函数，将runnable包装成callable.



```
Executors:
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}
```

```
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```

RunnableAdapter实现Callable接口。内部存放一个Runable字段，重写call方法调用Runnable的run方法



```
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

get方法，判断status，如果<COMPLETING，即还没执行。则调用awaitDone;，只分析，get调用时的情况

```
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    //timed为false.则deadline为0
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
        int s = state;
        if (s > COMPLETING) { //被唤醒后
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)   //第一次循环，q==null，初始化q.queued还是false
            q = new WaitNode();
        else if (!queued)   //第二次循环，cas修改waiters.将之前的waiters设置为q的后继节点，q变成waiters
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else 							//第三次循环，线程挂起
            LockSupport.park(this);
    }
}
```

awaitDone，如果task未执行，将当前线程包装成WaitNode.头插法修改waiters.

```
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

report方法，如果是正常执行结束，返回结果，如果被取消或中断(调用cancel方法)。则抛出异常。



```
public void run() {
		//只有新建状态才能执行，将runner设置为当前线程
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
            		//调用callable的call方法
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
            		//设置返回值
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

run方法，线程启动时的方法.或直接调用此方法。先将runner设置为当前线程，然后调用callable的call方法

然后调用set设置返回值。只有在call方法执行完成后，执行set方法时才会修改state.若call方法内部有阻塞行为，会检测中断标识，可能会抛出中断异常，抛出异常后被catch到，执行setException().将state设置为EXCEPTIONAL，释放waiters,结果设置为异常值。

call方法中没有判断中断中断位则任务开始后不会被中断。



```
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```

将state设置为COMPLETING，返回值设置为v,state设置为NORMAL.调用finshCompletion，释放所有因调用get方法挂起的线程

```
private void finishCompletion() {
    //遍历waiters链表
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    //唤醒线程
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    done();
    callable = null;        // to reduce footprint
}
```

遍历waiters链表，唤醒线程



```
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

被唤醒后，get方法继续执行。调用report.正常执行情况下s=2,直接outcome.



```
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```

取消方法，传入参数判断，如果线程正在运行是否中断.

如果status是NEW,参数是false，则把state设置为CANCELLED，之后其他线程调用get会抛出CancellationException异常。

如果status是NEW,参数是false，设置state设置为INTERRUPTING，继续向下执行，将runner的中断标志设置为true，然后将state设置为INTERRUPTED。然后释放waiters。



