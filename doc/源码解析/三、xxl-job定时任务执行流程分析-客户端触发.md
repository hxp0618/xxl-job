在《[xxl-job定时任务触发实现分析](https://zhuanlan.zhihu.com/p/436447196)》这篇文章中，分析如何实现定时任务的触发，是通过时间轮的机制触发定时任务的。而这篇文章是深入分析定时任务的被触发以后的执行流程是怎么样的。**定时任务执行流程分析分为两部分：客户端触发和服务端执行**。**客户端触发是记录触发的日志、准备触发参数触发远程服务器的执行。服务端执行是具体执行定时任务的，并将任务处理器处理的结果返回客户端。**这篇文章先分析客户端的触发。

### trigger方法准备触发任务

客户端的触发调用XxlJobTrigger类的trigger方法。该方法如下：

```java
//代码位置： com.xxl.job.admin.core.trigger.XxlJobTrigger#trigger
public static void trigger(int jobId,
                               TriggerTypeEnum triggerType,
                               int failRetryCount,
                               String executorShardingParam,
                               String executorParam,
                               String addressList) {

        // load data
        //从数据库中获取任务
        XxlJobInfo jobInfo = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().loadById(jobId);
        if (jobInfo == null) {
            logger.warn(">>>>>>>>>>>> trigger fail, jobId invalid，jobId={}", jobId);
            return;
        }
        //设置执行参数
        if (executorParam != null) {
            jobInfo.setExecutorParam(executorParam);
        }
        //重试次数
        int finalFailRetryCount = failRetryCount>=0?failRetryCount:jobInfo.getExecutorFailRetryCount();
        //任务调度组
        XxlJobGroup group = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().load(jobInfo.getJobGroup());

        // cover addressList
        //如果地址不为空，覆盖原来的地址列表
        if (addressList!=null && addressList.trim().length()>0) {
            group.setAddressType(1);
            group.setAddressList(addressList.trim());
        }

        // sharding param
        int[] shardingParam = null;
        //executorShardingParam不等于null
        if (executorShardingParam!=null){
            String[] shardingArr = executorShardingParam.split("/");
            if (shardingArr.length==2 && isNumeric(shardingArr[0]) && isNumeric(shardingArr[1])) {
                shardingParam = new int[2];
                shardingParam[0] = Integer.valueOf(shardingArr[0]);
                shardingParam[1] = Integer.valueOf(shardingArr[1]);
            }
        }
        //分片广播
        if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null)
                && group.getRegistryList()!=null && !group.getRegistryList().isEmpty()
                && shardingParam==null) {
            //并行处理
            for (int i = 0; i < group.getRegistryList().size(); i++) {
                //处理触发
                processTrigger(group, jobInfo, finalFailRetryCount, triggerType, i, group.getRegistryList().size());
            }
        } else {
            if (shardingParam == null) {
                shardingParam = new int[]{0, 1};
            }
            //处理触发
            processTrigger(group, jobInfo, finalFailRetryCount, triggerType, shardingParam[0], shardingParam[1]);
        }

}
```

trigger方法的逻辑如下：

- 根据任务id从数据库中获取执行的任务
- 根据任务组名字从数据库中获取任务组，如果地址不为空，覆盖原来的地址列表，设置触发类型为手动触发。
- 判断路由策略，如果是分片广播，遍历地址列表，触发所有的机器，否则只触发一台机器。分片广播是要触发所有的机器并行处理任务。

### processTrigger触发任务

接下来我们继续深入processTrigger方法处理触发的逻辑：

```java
//代码位置： com.xxl.job.admin.core.trigger.XxlJobTrigger#processTrigger
private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, TriggerTypeEnum triggerType, int index, int total){

        // param
        //执行阻塞策略
        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(jobInfo.getExecutorBlockStrategy(), ExecutorBlockStrategyEnum.SERIAL_EXECUTION);  // block strategy
        //路由策略
        ExecutorRouteStrategyEnum executorRouteStrategyEnum = ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null);    // route strategy
        //分片广播
        String shardingParam = (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==executorRouteStrategyEnum)?String.valueOf(index).concat("/").concat(String.valueOf(total)):null;

        // 1、save log-id
        //保存日志
        XxlJobLog jobLog = new XxlJobLog();
        //省略设置日志的代码

        // 2、init trigger-param 初始化触发参数
        TriggerParam triggerParam = new TriggerParam();
        //省略设置触发参数的代码

        // 3、init address 初始化地址
        String address = null;
        ReturnT<String> routeAddressResult = null;
        if (group.getRegistryList()!=null && !group.getRegistryList().isEmpty()) {
            if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST == executorRouteStrategyEnum) {
                if (index < group.getRegistryList().size()) {
                    address = group.getRegistryList().get(index);
                } else {
                    address = group.getRegistryList().get(0);
                }
            } else {
                //路由获取地址
                routeAddressResult = executorRouteStrategyEnum.getRouter().route(triggerParam, group.getRegistryList());
                if (routeAddressResult.getCode() == ReturnT.SUCCESS_CODE) {
                    address = routeAddressResult.getContent();
                }
            }
        } else {
            routeAddressResult = new ReturnT<String>(ReturnT.FAIL_CODE, I18nUtil.getString("jobconf_trigger_address_empty"));
        }

        // 4、trigger remote executor 触发远程执行器
        ReturnT<String> triggerResult = null;
        if (address != null) {
            triggerResult = runExecutor(triggerParam, address);
        } else {
            triggerResult = new ReturnT<String>(ReturnT.FAIL_CODE, null);
        }

        // 5、collection trigger info 触发信息
        StringBuffer triggerMsgSb = new StringBuffer();
        triggerMsgSb.append(I18nUtil.getString("jobconf_trigger_type")).append("：").append(triggerType.getTitle());
        //省略设置触发信息的代码

        // 6、save log trigger-info 保存触发日志
        jobLog.setExecutorAddress(address);
        jobLog.setExecutorHandler(jobInfo.getExecutorHandler());
        jobLog.setExecutorParam(jobInfo.getExecutorParam());
        jobLog.setExecutorShardingParam(shardingParam);
        jobLog.setExecutorFailRetryCount(finalFailRetryCount);
        //jobLog.setTriggerTime();
        jobLog.setTriggerCode(triggerResult.getCode());
        jobLog.setTriggerMsg(triggerMsgSb.toString());
        XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().updateTriggerInfo(jobLog);

        logger.debug(">>>>>>>>>>> xxl-job trigger end, jobId:{}", jobLog.getId());
}
```

processTrigger方法的逻辑主要如下：

- 获取执行阻塞策略
- 获取路由策略
- 保存任务日志
- 初始化触发参数
- 初始化执行器的地址：如果路由策略是分片广播，执行地址就为第index的地址，否则从通过路由获取执行地址。
- 触发远程执行器，即触发远程的定时任务
- 设置触发信息并保存触发日志

上述逻辑都不复杂，重点关注下触发远程执行器的runExecutor方法:

```java
//代码位置：com.xxl.job.admin.core.trigger.XxlJobTrigger#runExecutor
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
        ReturnT<String> runResult = null;
        try {
            //获取执行器
            ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
            //执行任务
            runResult = executorBiz.run(triggerParam);
        } catch (Exception e) {
            logger.error(">>>>>>>>>>> xxl-job trigger error, please check if the executor[{}] is running.", address, e);
            runResult = new ReturnT<String>(ReturnT.FAIL_CODE, ThrowableUtil.toString(e));
        }

        //返回执行结果
        StringBuffer runResultSB = new StringBuffer(I18nUtil.getString("jobconf_trigger_run") + "：");
        runResultSB.append("<br>address：").append(address);
        runResultSB.append("<br>code：").append(runResult.getCode());
        runResultSB.append("<br>msg：").append(runResult.getMsg());

        runResult.setMsg(runResultSB.toString());
        return runResult;
}
```

runExecutor方法通过 XxlJobScheduler.getExecutorBiz方法获取执行器ExecutorBiz，然后调用执行器ExecutorBiz的run方法执行任务。getExecutorBiz方法首先通过地址从executorBizRepository（map）获取ExecutorBiz，如果获取的ExecutorBiz不为null，则直接返回，否则，创建一个ExecutorBizClient保存在executorBizRepository中，然后将创建的ExecutorBizClient返回。ExecutorBiz的类图如下所示：

![img](https://pic1.zhimg.com/80/v2-678b8decc4cb5e4bd106cdeceae05168_1440w.webp)

ExecutorBiz接口有两个实现，分别是ExecutorBizClient（执行器客户端）、ExecutorBizImpl（执行器服务端），ExecutorBizClien类就是客户端操作任务的类，ExecutorBizImpl就是服务端操作任务的类。ExecutorBiz接口有beat（心跳检测）、idleBeat（空闲检测）、run（执行任务）、kill（停止任务）、log（打印日志）这些方法。我们看看ExecutorBizClien的run方法：

```java
//代码位置：com.xxl.job.core.biz.client.ExecutorBizClient#run
public ReturnT<String> run(TriggerParam triggerParam) {
        return XxlJobRemotingUtil.postBody(addressUrl + "run", accessToken, timeout, triggerParam, String.class);
}
```

ExecutorBizClien的run方法比较简单，就是调用http请求发送触发参数触发服务端的任务执行，然后将结果返回给客户端。请求的地址为addressUrl + "run"，当客户端发送请求以后，ExecutorBizImpl的run方法将会接收请求处理，然后将处理的结果返回，这篇文章就讲到这里，服务端执行定时任务放到下一篇文章进行讲解。

### 总结

客户端触发任务执行，首先从数据库中查询出需要执行的任务，然后做好任务执行的准备，如日志的记录、触发参数的初始化、获取执行的地址等，然后发送http请求给服务端执行任务，服务器将处理任务的结果返回给客户端。**客户端触发任务执行，是通过http请求触发任务执行，如果请求丢失，那么就会错过任务的执行。**