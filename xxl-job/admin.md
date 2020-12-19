几种线程

JobFailMonitorHelper

负责处理执行失败重试的线程



JobLogReportHelper

负责报表数据生成的线程



JobRegistryMonitorHelper

负责维护执行器实例列表的线程



JobScheduleHelper

负责从数据库获取任务的线程



JobTriggerPoolHelper

负责执行调度的线程池



调度中心启动流程：

1.XxlJobAdminConfig初始化时，新建xxlJobScheduler,并初始化

```
private XxlJobScheduler xxlJobScheduler;

@Override
public void afterPropertiesSet() throws Exception {
    adminConfig = this;
    xxlJobScheduler = new XxlJobScheduler();
    xxlJobScheduler.init();
}
```

2.XxlJobScheduler.init()

```
public void init() throws Exception {
    // 国际化
    initI18n();

    // 执行器注册，一个线程,30秒钟执行一次。从xxl_job_group获取所有自动注册的执行器，
    如果不为空，移除xxl_job_registry中90秒没有更新的具体执行器实例。
    获取xxl_job_registry中所有存活的执行器实例，更新xxl_job_group的address_list
    JobRegistryMonitorHelper.getInstance().start();

		//失败任务重试，一个线程，10秒钟执行一次。从xxl_job_log中查询1000条最新的失败的记录，
    遍历，更新log的alarm_status为-1，锁定状态。如果log中的重试次数>0,就调用
    JobTriggerPoolHelper.trigger重试一次。没有设置报警邮箱，设置为无需警告，
    有报警邮箱的，发送成功设置为警告成功，失败设置为警告失败。
    JobFailMonitorHelper.getInstance().start();
    
		//运行丢失，一个线程，1分钟执行一次，扫描执行中的任务log,如果执行器一下已下线，且执行超过10min，就设置为失败
    JobLosedMonitorHelper.getInstance().start();
        
    
    // 执行调度的线程，两个线程池，一个执行耗时短的线程，一个执行耗时长的线程。
    某个任务执行耗时>500ms,就累计一次次数，累计超过十次，就使用slow线程池。
    累计次数的map每分钟清空一次
    JobTriggerPoolHelper.toStart();

    //报表线程，处理首页报表数据，一个线程。一分钟执行一次，统计三天内的数据，计算成功，失败，正在执行的数量
    JobLogReportHelper.getInstance().start();

    //从数据库获取job的线程，两个线程，从数据库读取任务的线程scheduleThread，执行间隔短的任务的线程ringThread。
    scheduleThread,每次最多从数据库中取五秒内内要执行的(xxl.job.triggerpool.fast.max+xxl.job.triggerpool.fast.slow)*20个任务，默认是(200+200)*20
    条任务。
    遍历这些任务，如果(执行时间+5s)小于当前时间，即错过了上次执行，如果调度过期策略是立即执行一次，更新上次执行时间下次执行时间；
    如果过了执行时间，但是还没超过5s，就调用JobTriggerPoolHelper.addTrigger(),将任务添加到执行线程池，并更新时间，如果更新后的下次执行时间还在五秒钟之内，即执行间隔小于5s,则将其推给ringThread，然后再次更新时间。
    如果还没到下次执行时间,则推给ringThead,跟新下次执行时间。
    更新所有任务信息
    计算执行时间，如果小于1s,且之前任务列表为空，休眠4-5s,如果不为空，休眠0-1s
    
    ringThread,获取当前秒数，取当前描述两秒内的数据(当前20s,取20,21的数据).
    如果有数据调用JobTriggerPoolHelper.addTrigger()，休眠0-1秒。
    JobScheduleHelper.getInstance().start();

    logger.info(">>>>>>>>> init xxl-job admin success.");
}
```



XxlJobTrigger

具体负责调用执行器的类

方法：

```
public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount, String executorShardingParam, String executorParam)
```

传入，任务id,任务类型(corn表达式，手动触发，失败重试，子任务)，剩余重试次数(>0使用，否则使用job的数据)，executorShardingParam(分片任务使用)，executorParam(执行参数,不为空时使用，为空时使用job的数据)

流程：根据任务ID获取任务数据，执行器信息，调用processTrigger



```
private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, TriggerTypeEnum triggerType, int index, int total)
```

传入，执行器数据，任务数据，重试次数,任务类型，分片下标，分片总数

流程：查询路由策略，阻塞策略，新建日志。初始化trigger参数。根据路由策略，获取执行器地址信息；

调用runExecutor，记录日志信息



```
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address)
```

传入，trigger参数，执行器地址

调用XxlJobScheduler.getExecutorBiz获取ExecutorBiz，执行ExecutorBiz.run

```
public static ExecutorBiz getExecutorBiz(String address) throws Exception {
    // valid
    if (address==null || address.trim().length()==0) {
        return null;
    }

    // load-cache
    address = address.trim();
    ExecutorBiz executorBiz = executorBizRepository.get(address);
    if (executorBiz != null) {
        return executorBiz;
    }

    // set-cache
    XxlRpcReferenceBean referenceBean = new XxlRpcReferenceBean();
    referenceBean.setClient(NettyHttpClient.class);
    referenceBean.setSerializer(HessianSerializer.class);
    referenceBean.setCallType(CallType.SYNC);
    referenceBean.setLoadBalance(LoadBalance.ROUND);
    referenceBean.setIface(ExecutorBiz.class);
    referenceBean.setVersion(null);
    referenceBean.setTimeout(3000);
    referenceBean.setAddress(address);
    referenceBean.setAccessToken(XxlJobAdminConfig.getAdminConfig().getAccessToken());
    referenceBean.setInvokeCallback(null);
    referenceBean.setInvokerFactory(null);

    executorBiz = (ExecutorBiz) referenceBean.getObject();

    executorBizRepository.put(address, executorBiz);
    return executorBiz;
}
```

从map中根据地址获取对应的executorBiz，如果没有就新建，然后存入map并返回。

然后执行executorBizClient.run,向执行器发送http请求。



路由策略：

FIRST:第一个，返回执行器列表里的第一个

LAST:最后一个，返回执行器列表里的最后一个

ROUND:轮询，使用concurrentMap存放任务和任务执行的次数，一天清空一次。用count对列表长度取模

RANDOM:随机，从列表中随机选择一个

CONSISTENT_HASH:一致性hash,虚拟节点100*list.size个，addreddList存入treeMap，

对每个address计算一百个hash点。对任务id进行hash运算，取出比jobHash大的所有节点.返回第一个。

LEAST_FREQUENTLY_USED：LFU,使用频率最低的节点。使用concurrentMap存放任务id和任务在每个执行器执行的次数,一天清空一次。  初始化时或某个执行器执行这个任务一天内已超过一百万次，则中心赋个随机值(list.size内)。取出对应的map,根据执行次数排序。取第一个元素。

LEAST_RECENTLY_USED:LRU,最久未使用，使用concurrentMap存放任务id和任务在每个执行器地址和地址,一天清空一次。每个任务对应一个LinkedHashMap，初始化时指定accessOrder为true,设置访问顺序为get/put时的顺序。取出第一个key。取出后这个第一张被放到最后。一轮过后才会再次使用这个执行器。

FAILOVER:故障转移，遍历列表，进行心跳检测，直到找到一个心跳检测正常的。

BUSYOVER:忙碌转移，遍历列表，进行空闲心跳检测，直到找到一个空闲心跳检测正常的

SHARDING_BROADCAST：分片广播，任务分片





