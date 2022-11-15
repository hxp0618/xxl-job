在《[xxl-job定时任务执行流程分析-客户端触发](https://zhuanlan.zhihu.com/p/436886892)》这篇文章中，分析了客户端是如何触发服务端执行定时任务的，客户端通过http发送请求给服务端，服务端接收触发请求进行处理，将处理结果返回给客户端。这篇文章分析服务端时如何执行定时任务的。

执行器启动时，会初始化一个EmbedServer类，该类的start方法会启动netty服务器。netty服务器会接收客户端发送过来的http请求，当接收到触发请求（请求路径是/run）会交给EmbedServer类的process方法处理，process方法将会调用ExecutorBizImpl的run方法处理客户端发送的触发请求，process方法接收触发请求的代码如下图所示:

![img](https://pic3.zhimg.com/80/v2-ad11f9206fb464cc2d59b29c83751f86_1440w.webp)

ExecutorBizImpl的run方法处理流程大致如下：

- **加载任务处理器与任务执行线程，校验任务处理器与任务执行线程。**
- **执行阻塞策略**
- **注册任务**
- **保存触发参数到缓存**

### 加载任务处理器与任务执行线程，校验任务处理器与任务执行线程

```java
//代码位置：com.xxl.job.core.biz.impl.ExecutorBizImpl#run
public ReturnT<String> run(TriggerParam triggerParam) {
        // load old：jobHandler + jobThread
        //加载旧的任务处理器和任务线程
        JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
        IJobHandler jobHandler = jobThread!=null?jobThread.getHandler():null;
        String removeOldReason = null;

        // valid：jobHandler + jobThread
        GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
        if (GlueTypeEnum.BEAN == glueTypeEnum) {

            // new jobhandler 从缓存中加载任务处理器，根据处理器名字
            IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());

            // valid old jobThread 如果新的任务处理器与旧的任务处理器不同，将旧的任务处理器以及旧的任务线程gc
            if (jobThread!=null && jobHandler != newJobHandler) {
                // change handler, need kill old thread
                removeOldReason = "change jobhandler or glue type, and terminate the old job thread.";

                jobThread = null;
                jobHandler = null;
            }

            // valid handler
            if (jobHandler == null) {
                jobHandler = newJobHandler;
                if (jobHandler == null) {
                    return new ReturnT<String>(ReturnT.FAIL_CODE, "job handler [" + triggerParam.getExecutorHandler() + "] not found.");
                }
            }

        } else if (GlueTypeEnum.GLUE_GROOVY == glueTypeEnum) {
            //GLUE(Java)
            // valid old jobThread 校验GlueJobHandler
            if (jobThread != null &&
                    !(jobThread.getHandler() instanceof GlueJobHandler
                        && ((GlueJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
                // change handler or gluesource updated, need kill old thread
                removeOldReason = "change job source or glue type, and terminate the old job thread.";

                jobThread = null;
                jobHandler = null;
            }

            // valid handler
            if (jobHandler == null) {
                try {
                    IJobHandler originJobHandler = GlueFactory.getInstance().loadNewInstance(triggerParam.getGlueSource());
                    jobHandler = new GlueJobHandler(originJobHandler, triggerParam.getGlueUpdatetime());
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                    return new ReturnT<String>(ReturnT.FAIL_CODE, e.getMessage());
                }
            }
        } else if (glueTypeEnum!=null && glueTypeEnum.isScript()) {
            //脚本
            // valid old jobThread
            if (jobThread != null &&
                    !(jobThread.getHandler() instanceof ScriptJobHandler
                            && ((ScriptJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
                // change script or gluesource updated, need kill old thread
                removeOldReason = "change job source or glue type, and terminate the old job thread.";

                jobThread = null;
                jobHandler = null;
            }

            // valid handler
            if (jobHandler == null) {
                jobHandler = new ScriptJobHandler(triggerParam.getJobId(), triggerParam.getGlueUpdatetime(), triggerParam.getGlueSource(), GlueTypeEnum.match(triggerParam.getGlueType()));
            }
        } else {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "glueType[" + triggerParam.getGlueType() + "] is not valid.");
        }
}
```

run方法首先根据任务id从缓存jobThreadRepository（map）中获取任务执行线程jobThread，任务执行线程jobThread保存着任务处理器jobHandler，然后进行校验任务执行线程以及任务处理器。在了解校验过程之前，我们先了解下xxl-job定时任务的种类，xxll0job支持java、groovy、脚本（Shell、Python、PHP、NodeJs、PowerShell）的定时任务。

接下来检验任务执行线程以及任务处理器，就是按照Java、groovy、脚本分别进行校验。

当任务的种类是java时,根据任务处理器的名字从jobHandlerRepository（map）中获取任务处理器，如果新的任务处理器与旧的任务处理器不同，将旧的任务处理器以及旧的任务线程设置为null，等待被java虚拟机gc掉，这样做的目的是，如果已经重新设置了新的任务执行线程和任务处理器，那么就旧的gc掉，不至于一直存在内存中。

如果任务的种类是groovy时，判断任务执行线程不等于null、任务处理器已经更改和groovy的代码被更新了，那么就将旧的任务执行线程和任务执行器设置为null，等待被gc，如果任务处理器还是为null，那么新创建GlueJobHandler任务处理器。

如果是任务的种类是脚本类型，判断任务执行线程不等于null、任务处理器已经更改和脚本的代码被更新了，那么就将旧的任务执行线程和任务执行器设置为null，等待被gc，如果任务处理器还是为null，那么新创建ScriptJobHandler任务处理器。

### 执行阻塞策略

```java
//代码位置：com.xxl.job.core.biz.impl.ExecutorBizImpl#run
public ReturnT<String> run(TriggerParam triggerParam) {
    //省略代码
    //加载任务处理器与任务执行线程，校验任务处理器与任务执行线程

      if (jobThread != null) {
            //阻塞策略
            ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
            //丢弃
            if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) {
                // discard when running 如果任务正在执行，直接返回结果，不再往下执行任务
                if (jobThread.isRunningOrHasQueue()) {
                    return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
                }
            } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
                //覆盖之前的
                // kill running jobThread
                if (jobThread.isRunningOrHasQueue()) {
                    removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();

                    jobThread = null;
                }
            } else {
                // just queue trigger
            }
       }

    //代码省略

}
```

**xxl-job有三种阻塞策略，分别为SERIAL_EXECUTION（并行）、DISCARD_LATER（丢弃）、COVER_EARLY（覆盖之前的）**。当阻塞策略为丢弃，则判断该执行线程是否正在执行，如果是则直接返回结果，不再往下执行任务了。当阻塞策略为覆盖之前的，则判断执行线程是否正在执行，如果是则杀掉原来的执行线程。如果阻塞策略是并行，则不做什么。

### 注册任务

```java
//代码位置：com.xxl.job.core.biz.impl.ExecutorBizImpl#run
public ReturnT<String> run(TriggerParam triggerParam) {
    //省略代码
    //加载任务处理器与任务执行线程，校验任务处理器与任务执行线程
    //执行阻塞策略

    //如果任务线程等于null，注册任务线程并启动线程
    if (jobThread == null) {
         jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
    }

    //省略代码
}

//代码位置：com.xxl.job.core.executor.XxlJobExecutor#registJobThread
public static JobThread registJobThread(int jobId, IJobHandler handler, String removeOldReason){
        JobThread newJobThread = new JobThread(jobId, handler);
        newJobThread.start();
        logger.info(">>>>>>>>>>> xxl-job regist JobThread success, jobId:{}, handler:{}", new Object[]{jobId, handler});

        JobThread oldJobThread = jobThreadRepository.put(jobId, newJobThread);  // putIfAbsent | oh my god, map's put method return the old value!!!
        if (oldJobThread != null) {
            oldJobThread.toStop(removeOldReason);
            oldJobThread.interrupt();
        }

        return newJobThread;
}
```

如果任务线程等于null，注册任务线程并启动线程。registJobThread方法首先新建一个任务线程，并调用newJobThread的start方法启动任务线程。然后加入jobThreadRepository进行缓存，当旧的oldJobThread不等于null，则停止掉旧的任务线程。

### 保存触发参数到缓存

```java
//代码位置：
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

pushTriggerQueue方法判断任务id是否已经存在triggerLogIdSet中，如果存在就直接返回结果，如果不存在就添加到triggerLogIdSet中，然后将触发参数保存在triggerQueue队列中。

### 总结

以上就是服务端执行定时任务的流程分析，加载任务处理器与任务执行线程，校验任务处理器与任务执行线程、执行阻塞策略、注册任务、保存触发参数到缓存。但是任务的执行还没有分析到，在注册任务中，会调用newJobThread的start方法启动任务线程执行任务，我们将在下一篇中具体分析下任务执行的源码。