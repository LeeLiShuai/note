几种线程：
ExecutorRegistryThread

注册的线程，一个线程，30s执行一次，请求每个调度中心的api/registry方法，更新提交心跳时间



JobLogFileCleanThread

清理日志的线程，一个线程，一天执行一次，清理24天之前的日志



JobThread

执行任务的线程



TriggerCallbackThread

回调结果的线程



执行器初始化流程：

XxlJobConfig中配置XxlJobSpringExector



负责接收请求的类：

EmbedServer，初始化时开启一个 netty服务器。内部调用ExecutorBizImpl,处理具体的请求 



处理具体业务的类：

ExecutorBizImpl,执行器处理实现类，主要方法：

beat:处理心跳请求，直接返回success.用于调度策略为故障转移的任务执行时调用

idleBeat:处理空闲心跳请求，通过传入的jobId,调用XxlJobExecutor的方法获取对应的JobThread,如果不为空，且正在运行或队列中有数据，咋返回失败，否则返回成功 

kill:处理中断任务请求，一般由网页控制台手动触发，获取对应的JobThread,从XxlJobExecutor中移除

log:处理查看日志请求，一般由网页控制台触发，根据传入的logid,时间和行数，返回对应的日志数据

run:处理执行任务请求，自动调度或手动触发，获取对应JobThread和handLer。只分析bean模式，

读取最新的jobHandler,如果和上一个jobHandler不同的话，移除上一个handler并更新

如果jobThread不为空，根据阻塞策略是丢弃后续调度，且有任务正在执行 ，则返回错误，此次不执行 。

如果是覆盖之前调度，kill正在运行的任务 (重新注册一个就行)，重新注册一个jobThread，然后用put进jobThrad的等待队列中。



启动的父类：

XxlJobEecutor，负责初始化执行器，并记录每个任务对应的执行线程。

初始化流程在最后。

字段:

```
//调度中心的地址
private String adminAddresses;
//token
private String accessToken;
//执行器名称
private String appname;
//执行器netty服务器地址，由下面的 ip+port组成
private String address;
//ip
private String ip;
//netty监测的端口
private int port;
//日志位置
private String logPath;
//日志存放时间
private int logRetentionDays;

//记录调度中心地址
private static List<AdminBiz> adminBizList;
//自己
private EmbedServer embedServer = null;
//存放负责处理每个任务的类的方法，在XxlJobSpringExecutor初始化时调用
private static ConcurrentMap<String, IJobHandler> jobHandlerRepository = new ConcurrentHashMap<>();
//存放具体处理每个任务的线程
private static ConcurrentMap<Integer, JobThread> jobThreadRepository = new ConcurrentHashMap<>();

```



方法：

```
public void start() throws Exception {

    //初始化日志地址
    XxlJobFileAppender.initLogPath(logPath);
    //初始化调度器之地，存入列表中 
    initAdminBizList(adminAddresses, accessToken);
    //初始化日志清理线程
    JobLogFileCleanThread.getInstance().start(logRetentionDays);
    //初始化回调线程
    TriggerCallbackThread.getInstance().start();
    //初始化netty服务器
    initEmbedServer(address, ip, port, appname, accessToken);
}
```

首先初始化JobHandlerRepository,扫描所有被XxlJob注解修饰的方法，存入jobHandlerRepository中。

调用父类即XxlJobExecutor.start

初始化日志位置,

初始化adminBiz

初始化清理日志的线程JobLogFileCleanThread



TriggerCallBackTread初始化过程 ：

初始化TriggerCallBackThread,回调线程。两个线程，普通回调的triggerCallbackThread,重试请求回调的triggerRettryCallbackThread.

triggerCallbackThread:从callbackQueue中取出全部元素，执行doCallback。

​	doCallback:获取所有adminBiz,执行callback,发送请求到admin.如果成功了，记录日志。失败了则记录到错误日志中

triggerRettryCallbackThread：30s执行一次，读取失败记录日志，反序列化，执行doCallback



EmbedServer初始化过程：

初始化embedServer,为admin调用提供服务。默认在9999端口,初始化一个netty服务。只接受POST请求。

/beat,心跳检测，直接返回成功

/idleBeat，是否空闲的心跳检测，如果有正在运行的任务，或者等待队列中有任务则返回失败，否则成功

/run,admin调用trigger的方法，根据拒绝策略和阻塞策略执行任务。如果没有阻塞或队列已满，则入队等待执行

/kill,终止任务，从executor中移除线程

/log,读取日志内容，从磁盘中读取

这些接口由调度器的ExecutorBizClient调用 

初始化ExecutorRegistryThread线程，过程见第一行。



```
//每个任务第一次执行，或空循环30次，或执行器重启，对应的jobhandler变化是调用
public static JobThread registJobThread(int jobId, IJobHandler handler, String removeOldReason){
		//新建一个JobThread,然后start.存到map中，将老的thread停止
    JobThread newJobThread = new JobThread(jobId, handler);
    newJobThread.start();
    logger.info(">>>>>>>>>>> xxl-job regist JobThread success, jobId:{}, handler:{}", new Object[]{jobId, handler});

    JobThread oldJobThread = jobThreadRepository.put(jobId, newJobThread); // putIfAbsent | oh my god, map's put method return the old value!!!
    if (oldJobThread != null) {
        oldJobThread.toStop(removeOldReason);
        oldJobThread.interrupt();
    }
    return newJobThread;
}
```



具体执行任务的线程：

JobThread

字段：

```
//对应的任务的ID
private int jobId;
//执行任务的类，存放的类型是MethodJobHandler
private IJobHandler handler;
//存放ExecutorBizImpl.run推送进来的需要执行的任务
private LinkedBlockingQueue<TriggerParam> triggerQueue;
//所有推送进的任务对应的日志ID
private Set<Long> triggerLogIdSet;    // avoid repeat trigger for the same TRIGGER_LOG_ID
//是否已被停止
private volatile boolean toStop = false;
//停止的原因
private String stopReason;
//是否正在运行 
private boolean running = false;    // if running job
//线程空循环的次数，如果超过30次空循环,则移除当前JobThread
private int idleTimes = 0;       // idel times
```



方法：

```
public ReturnT<String> pushTriggerQueue(TriggerParam triggerParam) {
   // avoid repeat
   if (triggerLogIdSet.contains(triggerParam.getLogId())) {
      logger.info(">>>>>>>>>>> repeate trigger job, logId:{}", triggerParam.getLogId());
      return new ReturnT<String>(ReturnT.FAIL_CODE, "repeate trigger job, logId:" + triggerParam.getLogId());
   }

   triggerLogIdSet.add(triggerParam.getLogId());
   triggerQueue.add(triggerParam);
       return ReturnT.SUCCESS;
}
```

调度任务推送进来，首先判断这个任务的ID是否已经存在，存在返回错误。加入队列中返回成功。



```
public boolean isRunningOrHasQueue() {
    return running || triggerQueue.size()>0;
}
```

是否非空闲，判断是否正在运行或队列中元素>0



run方法流程：

调用初始化方法，当toStop为false的时候，while循环，使用poll从队列中取元素，running赋值为true，

idleTimes重置为0，把这个任务对应的日志ID从set移除。

如果任务有超时时间限制，则新建FutrueTask执行，没有时间限制则直接执行 。最终

将执行结果推给TriggerCallbackThread





处理回调的线程类:

TriggerCallbackThread

字段：

```
//自己
private static TriggerCallbackThread instance = new TriggerCallbackThread();
//存放回调信息的队列
private LinkedBlockingQueue<HandleCallbackParam> callBackQueue = new LinkedBlockingQueue<HandleCallbackParam>();
//回调的线程
private Thread triggerCallbackThread;
//回调失败重试的线程
private Thread triggerRetryCallbackThread;
//是否被停止 
private volatile boolean toStop = false;
//回调失败日志位置
private static String failCallbackFilePath = XxlJobFileAppender.getLogPath().concat(File.separator).concat("callbacklog").concat(File.separator);
//回调失败日志名称
private static String failCallbackFileName = failCallbackFilePath.concat("xxl-job-callback-{x}").concat(".log");
```



方法：

```
public static void pushCallBack(HandleCallbackParam callback){
    getInstance().callBackQueue.add(callback);
    logger.debug(">>>>>>>>>>> xxl-job, push callback request, logId:{}", callback.getLogId());
}
```

jobThread执行完成后调用 ，直接放入 队列中 



run方法执行过程：

while从队列中使用take从队列中取元素。因为take是加锁的，取不到时会await。

一直等到队列中有元素才会返回。然后从队列中取出全部元素，调用doCallback



```
private void doCallback(List<HandleCallbackParam> callbackParamList){
    boolean callbackRet = false;
    // callback, will retry if error
    for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
        try {
            ReturnT<String> callbackResult = adminBiz.callback(callbackParamList);
            if (callbackResult!=null && ReturnT.SUCCESS_CODE == callbackResult.getCode()) {
                callbackLog(callbackParamList, "<br>----------- xxl-job job callback finish.");
                callbackRet = true;
                break;
            } else {
                callbackLog(callbackParamList, "<br>----------- xxl-job job callback fail, callbackResult:" + callbackResult);
            }
        } catch (Exception e) {
            callbackLog(callbackParamList, "<br>----------- xxl-job job callback error, errorMsg:" + e.getMessage());
        }
    }
    if (!callbackRet) {
        appendFailCallbackFile(callbackParamList);
    }
}
```

遍历所有adminBiz，调用callback方法。adminBiz就是启动时初始化的用于请求调度中心的工具



