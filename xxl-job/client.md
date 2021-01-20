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



EmbedServer初始化过程：

初始化embedServer,为admin调用提供服务。默认在9999端口,初始化一个netty服务。只接受POST请求。

/beat,心跳检测，直接返回成功

/idleBeat，是否空闲的心跳检测，如果有正在运行的任务，或者等待队列中有任务则返回失败，否则成功

/run,admin调用trigger的方法，根据拒绝策略和阻塞策略执行任务。如果没有阻塞或队列已满，则入队等待执行

/kill,终止任务，从executor中移除线程

/log,读取日志内容，从磁盘中读取

这些接口由调度器的ExecutorBizClient调用 



接收到一个run请求的流程：

netty接收到请求之后，调用ExecutorBizImpl的run方法。根据jobId获取对应的jobThread,和对应的jobHandler。

根据阻塞策略，进行的丢弃，覆盖或者串行。串行时，将任务存入自己的阻塞队列中。

jobThread本身会不停的从阻塞队列中取出请求，并调用对应的jobhandler处理请求。

处理完之后，将处理结果推送到TriggerCallbackThread，存入TriggerCallbackThread的阻塞队列中

TriggerCallbackThread从自己的阻塞队列中取出数据，向所有的执行器回调。




