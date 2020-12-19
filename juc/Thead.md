Thread

线程状态：

NEW, 创建完毕，未启动状态

RUNABLE,可运行状态，在虚拟机中已经启动，等待操作系统资源

RUNNING，运行状态

BLOCKED,等待同步器状态，等待获取锁

WAITNG，等待状态，因调用object.wait,Thread.join,LockSupport.park等方法进入此状态。释放锁和资源,只有其他线程调用object.notify,object.notifyAll,THrea.join,LockSupport.unPark,才会被唤醒，去竞争锁对象

TIMED_WAITING,限时等待，因调用Thread.sleep,Object.wait(long),Thread.join(long),LockSupport.parkNanos,LockSupport.parkUntil等方法进入此状态，当时间结束后或其他线程调用object.notify,object.notifyAll,THrea.join,LockSupport.unPark,才会被唤醒，去竞争锁对象

TERMINATED,线程已执行完毕



sleep()

休眠,不会释放锁



yield()

提醒调度器此线程愿意放CPU资源，如果线程从RUNNING切换到RUNABLE状态



join()

父线程等待子线程执行完成后再执行



Object.wait()

线程进入等待，必须再同步方法或同步代码块中执行，否则会抛出异常。会释放锁



中断：

Interrupt().将中断为设置为true。如果线程处于阻塞状态，会抛出InterruptedException异常,并将中断标志位设置为false.

阻塞线程并有可能抛出InterruptedException异常的方法有：

Object.wait

Object.wait(long)

Object.wait(long,int)

Thread.sleep(long)

Thread.sleep(long,int)

Thread.join()

Thread.join(long)

Thread.join(long,int)



静态方法interrupted()，返回中断标志位是否为true，并设置为false



isInterrupted(),返回中断标志位是否为true



