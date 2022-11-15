在《[调度中心启动流程分析](https://zhuanlan.zhihu.com/p/435419514)》这篇文章中，分析了调度中心启动流程分析，调度中心启动过程中会启动调度任务。这一篇文章主要深入调度任务的启动源码。

JobScheduleHelper类的start方法就是用来启动调度任务的，start方法会创建并启动scheduleThread和ringThread两个线程，代码如下：

```java
//代码位置：com.xxl.job.admin.core.thread.JobScheduleHelper#start
public void start(){

    ////创建并初始化scheduleThread
    scheduleThread = new Thread(new Runnable(){
        //run方法代码省略
    });
    scheduleThread.setDaemon(true);
    scheduleThread.setName("xxl-job, admin JobScheduleHelper#scheduleThread");
    scheduleThread.start();

    //创建并初始化ringThread
    ringThread = new Thread(new Runnable(){
        //run方法代码省略
    });
    ringThread.setDaemon(true);
    ringThread.setName("xxl-job, admin JobScheduleHelper#ringThread");
    ringThread.start();
}
```

接下来我们看下scheduleThread线程和ringThread线程的run方法的逻辑，在这之前，我们先看下面两个问题：

1. **xxl-job集群部署时，如何避免多个服务器同时调度任务？**
2. **定时任务是如何实现的？**

带着这两个问题分析scheduleThread线程的run方法：

### **xxl-job集群部署时，如何避免多个服务器同时调度任务？**

```java
public void run() {
    //省略代码
    //取的定时任务数量
    int preReadCount = (XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax() +     XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax()) * 20;

    //循环
    while(!scheduleThreadToStop){

        // Scan Job
        long start = System.currentTimeMillis();

        Connection conn = null;
        Boolean connAutoCommit = null;
        PreparedStatement preparedStatement = null;

        boolean preReadSuc = true;
        try {
            conn = XxlJobAdminConfig.getAdminConfig().getDataSource().getConnection();
            connAutoCommit = conn.getAutoCommit();
            //闭隐式自动提交事务
            conn.setAutoCommit(false);
            //select lock for update（显式排他锁，其他事务无法进入&无法实现for update）
            preparedStatement = conn.prepareStatement("select * from xxl_job_lock where lock_name = 'schedule_lock' for update");
            preparedStatement.execute();

            // tx start

            /**
                省略了定时调度任务的代码
            */

            // tx stop
        } catch (Exception e) {
            //省略代码
        } finally {
            //省略代码
            //提交事务
             conn.commit();
            //设置提交
            conn.setAutoCommit(connAutoCommit);
        }
        //省略代码
    }

}
```

首先我们分析下，xxl-job在集群部署时，如何避免多个服务器同时调度任务，上述run方法省略了大部分代码，但是不影响分析。xxl-job通过mysql悲观锁实现分布式锁，从而避免多个服务器同时调度任务：

- 通过setAutoCommit(false)，关闭自动提交
- 通过select lock for update语句，其他事务无法获取到锁，显示排她锁。
- 进行定时调度任务的逻辑（这部分代码省略，在下面进行分析）
- 最后在finally块中commit()提交事务，并且setAutoCommit，释放for update的排他锁。

### **定时任务是如何实现的**

```java
public void run() {

        //省略代码
        //预读取的定时任务数量
        int preReadCount = (XxlJobAdminConfig.getAdminConfig().getTriggerPoolFastMax() + XxlJobAdminConfig.getAdminConfig().getTriggerPoolSlowMax()) * 20;

        while (!scheduleThreadToStop) {

            //省略代码
            try {
                //省略加锁的代码
                // tx start
                // 1、pre read
                long nowTime = System.currentTimeMillis();
                //从数据库把5秒内要执行的任务读出
                List<XxlJobInfo> scheduleList = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleJobQuery(nowTime + PRE_READ_MS, preReadCount);
                if (scheduleList != null && scheduleList.size() > 0) {
                    // 2、push time-ring
                    for (XxlJobInfo jobInfo : scheduleList) {

                        // time-ring jump
                        //如果当前时间大于下一次触发时间加上五秒（超过下一次触发时间的五秒内的时间）
                        //下一次触发时间过期时间超过5秒
                        if (nowTime > jobInfo.getTriggerNextTime() + PRE_READ_MS) {
                            // 2.1、trigger-expire > 5s：pass && make next-trigger-time
                            logger.warn(">>>>>>>>>>> xxl-job, schedule misfire, jobId = " + jobInfo.getId());

                            // 1、misfire match
                            //默认什么也不做策略
                            MisfireStrategyEnum misfireStrategyEnum = MisfireStrategyEnum.match(jobInfo.getMisfireStrategy(), MisfireStrategyEnum.DO_NOTHING);
                            //立即执行一次策略
                            if (MisfireStrategyEnum.FIRE_ONCE_NOW == misfireStrategyEnum) {
                                // FIRE_ONCE_NOW 》 trigger
                                JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.MISFIRE, -1, null, null, null);
                                logger.debug(">>>>>>>>>>> xxl-job, schedule push trigger : jobId = " + jobInfo.getId());
                            }

                            // 2、fresh next
                            //设置下一次的触发时间
                            refreshNextValidTime(jobInfo, new Date());

                        } else if (nowTime > jobInfo.getTriggerNextTime()) {
                            // 2.2、trigger-expire < 5s：direct-trigger && make next-trigger-time
                            //触发时间过期时间小于5秒。立即执行一次
                            // 1、trigger
                            JobTriggerPoolHelper.trigger(jobInfo.getId(), TriggerTypeEnum.CRON, -1, null, null, null);
                            logger.debug(">>>>>>>>>>> xxl-job, schedule push trigger : jobId = " + jobInfo.getId());

                            // 2、fresh next
                            //设置下一次触发时间
                            refreshNextValidTime(jobInfo, new Date());

                            // next-trigger-time in 5s, pre-read again
                            //如果任务正在运行并且在当前时间的5秒内，放进时间轮
                            if (jobInfo.getTriggerStatus() == 1 && nowTime + PRE_READ_MS > jobInfo.getTriggerNextTime()) {

                                // 1、make ring second
                                //时间轮
                                int ringSecond = (int) ((jobInfo.getTriggerNextTime() / 1000) % 60);

                                // 2、push time ring
                                //加入时间轮
                                pushTimeRing(ringSecond, jobInfo.getId());

                                // 3、fresh next
                                //刷新下一次触发时间
                                refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));

                            }

                        } else {
                            // 2.3、trigger-pre-read：time-ring trigger && make next-trigger-time

                            // 1、make ring second 秒的时间轮
                            int ringSecond = (int) ((jobInfo.getTriggerNextTime() / 1000) % 60);

                            // 2、push time ring 放进时间轮
                            pushTimeRing(ringSecond, jobInfo.getId());

                            // 3、fresh next 更新下一次触发时间
                            refreshNextValidTime(jobInfo, new Date(jobInfo.getTriggerNextTime()));

                        }

                    }

                    // 3、update trigger info
                    //更新调度任务信息
                    for (XxlJobInfo jobInfo : scheduleList) {
                        XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleUpdate(jobInfo);
                    }

                } else {
                    preReadSuc = false;
                }
                // tx stop

            } catch (Exception e) {
                //省略代码
            } finally {
               //省略代码       
            }
           //省略代码
}
```

xxl_job_info表是记录定时任务的表，里面有个trigger_next_time（Long）字段，表示下一次任务被触发的时间，任务每被触发一次都要更新trigger_next_time字段，这样就知道任务何时被触发。定时任务的实现分成下面几步：

- 从数据库中读取5秒内需要执行的任务，并遍历任务。
- 如果当前时间超过下一次触发时间5秒，获取此时调度任务已经过期的调度策略的配置，默认是什么也做策略。如果配置是立即执行一次策略，那么就立即触发定时任务，否则什么也不做。最后更新下一次触发时间。
- 如果当前时间超过下一次触发时间，但并没有超过5秒，立即触发一次任务，然后更新下一次触发时间。如果任务正在运行并且更新以后的触发时间在当前时间5秒内，将任务放进时间轮，然后再次更新下一次触发时间。因为触发时间太短了所以就放进时间轮中，供下一次触发。
- 如果不是上面的两种情况，则计算时间轮，将任务放进时间轮中，最后更新下一次触发时间。
- 更新调度任务信息保存到数据库中，更新trigger_next_time字段。

上述就是任务调度的实现，从数据库中不断读出5秒内需要执行的任务，然后根据下一次触发时间来触发定时任务，其中会将定时任务放进时间轮中，ringThread线程会从时间轮中获取任务执行。那到底什么是时间轮呢？

![img](https://pic3.zhimg.com/80/v2-e8789eff2af8ecf5f2f21cf673e9b82e_1440w.webp)

上述就是时间轮的图示，将一段时间分成相等的时间，每个相等的时间关联着任务。在xxl-job中，时间论的数据结构如下：

```java
private volatile static Map<Integer, List<Integer>> ringData = new ConcurrentHashMap<>();
```

ringData时间轮的key是秒数（1-60），value是任务id列表。下面分析下ringThread是如何触发任务的，ringThread线程的run方法如下：

```java
public void run() {

        while (!ringThreadToStop) {

            // align second
            //休息一秒
            try {
                TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis() % 1000);
            } catch (InterruptedException e) {
                if (!ringThreadToStop) {
                    logger.error(e.getMessage(), e);
                }
            }

            try {
                // second data
                List<Integer> ringItemData = new ArrayList<>();
                int nowSecond = Calendar.getInstance().get(Calendar.SECOND);   // 避免处理耗时太长，跨过刻度，向前校验一个刻度；
                for (int i = 0; i < 2; i++) {
                    List<Integer> tmpData = ringData.remove((nowSecond + 60 - i) % 60);
                    if (tmpData != null) {
                        ringItemData.addAll(tmpData);
                    }
                }

                // ring trigger
                logger.debug(">>>>>>>>>>> xxl-job, time-ring beat : " + nowSecond + " = " + Arrays.asList(ringItemData));
                if (ringItemData.size() > 0) {
                    // do trigger
                    //触发定时任务
                    for (int jobId : ringItemData) {
                        // do trigger
                        JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null, null);
                    }
                    // clear
                    ringItemData.clear();
                }
            } catch (Exception e) {
                if (!ringThreadToStop) {
                    logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread error:{}", e);
                }
            }
        }
        logger.info(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread stop");
}
```

ringThread线程的run方法首先获取当前的时间（秒数），然后从时间轮内移出当前秒数前2个秒数的任务列表，遍历任务列表触发任务的执行，最后清空已经执行的任务列表。这里获取当前秒数前2个秒数的任务列表是因为避免处理时间太长导致错失了调度。

### 总结

xxl-job实现定时任务的调度，是依靠时间轮实现的，将5秒内要执行的任务从数据库中查询出来，根据当前时间与下一次触发时间的比较，决定任务是执行还是放进时间轮中，时间轮中任务根据当前的秒数被触发执行。这篇文章分析了定时任务是如何实现的，只分析到任务被触发。并没有深入任务执行的流程，下一篇文章将会分析任务执行的流程。