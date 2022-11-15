现在的服务大部分都是微服务，在分布式环境中，服务在多台服务器上部署。xxl-job为了防止定时任务在同一时间内多台服务运行定时任务，利用数据库的悲观锁保证同一时间内只有一台服务运行定时任务，在运行定时任务之前首先获取到锁（select lock for update），然后才运行定时任务，当任务运行完成时，释放悲观锁，其他服务就可以去尝试获取锁而执行定时任务。

上述是xxl-job在分布式环境中如何保证同一时间只有一台服务运行定时任务，那么如何从多台服务器中选出一台服务来运行定时任务，这就涉及到xxl-job执行器路由选择的问题，接下来分析xxl-job是如何选择执行器的。执行器路由抽象类ExecutorRouter的route方法是选择服务器地址的，决定哪一台服务器执行定时任务，它有几个子类，如下图所示。

![img](https://pic4.zhimg.com/80/v2-7e6065950e57b087918007c590fef043_1440w.webp)

- **ExecutorRouteBusyover**：忙碌转移路由，从执行器地址列表查找心跳正常的执行器地址。
- **ExecutorRouteFailover**：故障转移路由，查找心跳正常的执行器地址。
- **ExecutorRouteLast**：执行器地址列表的最后一个地址。
- **ExecutorRouteFirst**：执行器地址列表的第一个地址。
- **ExecutorRouteConsistentHash**：哈希一致性路由，通过哈希一致性算法选择执行器地址
- **ExecutorRouteLFU**：最不经常使用路由，使用频率最低的执行器地址。
- **ExecutorRouteLRU**：最近最久未使用路由，选择最近最久未被使用的执行器地址。
- **ExecutorRouteRandom**：随机路由，随机选择一个执行器地址。
- **ExecutorRouteRound**：轮询路由，轮询选择一个执行器地址。

接下来，我们具体分析下上述路由的route方法。

### ExecutorRouteBusyover

```java
//代码位置：com.xxl.job.admin.core.route.strategy.ExecutorRouteBusyover#route
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        StringBuffer idleBeatResultSB = new StringBuffer();
        //遍历执行器地址
        for (String address : addressList) {
            // beat
            ReturnT<String> idleBeatResult = null;
            try {
                ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
                //空闲检测，当任务线程没有在执行，返回true
                idleBeatResult = executorBiz.idleBeat(new IdleBeatParam(triggerParam.getJobId()));
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                idleBeatResult = new ReturnT<String>(ReturnT.FAIL_CODE, ""+e );
            }
            idleBeatResultSB.append( (idleBeatResultSB.length()>0)?"<br><br>":"")
                    .append(I18nUtil.getString("jobconf_idleBeat") + "：")
                    .append("<br>address：").append(address)
                    .append("<br>code：").append(idleBeatResult.getCode())
                    .append("<br>msg：").append(idleBeatResult.getMsg());

            // beat success
            //如果空闲检测成功，则返回执行器地址
            if (idleBeatResult.getCode() == ReturnT.SUCCESS_CODE) {
                idleBeatResult.setMsg(idleBeatResultSB.toString());
                idleBeatResult.setContent(address);
                return idleBeatResult;
            }
        }

        return new ReturnT<String>(ReturnT.FAIL_CODE, idleBeatResultSB.toString());
}
```

ExecutorRouteBusyover是忙碌转移路由器，route方法首先遍历执行器地址列表，然后对执行器地址进行空闲检测，当任务线程没有在执行定时任务时，将返回空闲检测成功，将该执行器地址返回。

### ExecutorRouteFailover

```java
//代码位置：com.xxl.job.admin.core.route.strategy.ExecutorRouteFailover#route
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {

        StringBuffer beatResultSB = new StringBuffer();
        for (String address : addressList) {
            // beat
            ReturnT<String> beatResult = null;
            try {
                ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
                //心跳检测
                beatResult = executorBiz.beat();
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                beatResult = new ReturnT<String>(ReturnT.FAIL_CODE, ""+e );
            }
            beatResultSB.append( (beatResultSB.length()>0)?"<br><br>":"")
                    .append(I18nUtil.getString("jobconf_beat") + "：")
                    .append("<br>address：").append(address)
                    .append("<br>code：").append(beatResult.getCode())
                    .append("<br>msg：").append(beatResult.getMsg());

            // beat success
            //心跳正常，返回执行器地址
            if (beatResult.getCode() == ReturnT.SUCCESS_CODE) {

                beatResult.setMsg(beatResultSB.toString());
                beatResult.setContent(address);
                return beatResult;
            }
        }
        return new ReturnT<String>(ReturnT.FAIL_CODE, beatResultSB.toString());

}
```

ExecutorRouteFailover是失败转移路由，route方法遍历执行器地址，然后发送心跳给执行器服务，如果心跳正常，则成功返回该执行器地址，否则返回失败码。

### ExecutorRouteLast、ExecutorRouteFirst

ExecutorRouteLast、ExecutorRouteFirst路由的route方法代码就不贴了，ExecutorRouteFirst的route方法将执行器列表的第一个执行器地址返回，ExecutorRouteLast的route方法将执行器列表的最后一个执行器地址返回。

### ExecutorRouteConsistentHash

```java
public String hashJob(int jobId, List<String> addressList) {

        // ------A1------A2-------A3------
        // -----------J1------------------
        TreeMap<Long, String> addressRing = new TreeMap<Long, String>();
        for (String address: addressList) {
            //将虚拟节点与执行器地址关联起来
            for (int i = 0; i < VIRTUAL_NODE_NUM; i++) {
                long addressHash = hash("SHARD-" + address + "-NODE-" + i);
                addressRing.put(addressHash, address);
            }
        }

        //将jobid进行md5 hash
        long jobHash = hash(String.valueOf(jobId));
        //获取大于等于 jobHash的虚拟节点与执行器地址对应关系
        SortedMap<Long, String> lastRing = addressRing.tailMap(jobHash);
        //如果不为空，返回第一个key对应的执行器地址
        if (!lastRing.isEmpty()) {
            return lastRing.get(lastRing.firstKey());
        }
        //第一个执行器地址
        return addressRing.firstEntry().getValue();
}

    @Override
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = hashJob(triggerParam.getJobId(), addressList);
        return new ReturnT<String>(address);
}
```

hashJob方法首先遍历执行器地址列表，对每一个执行器地址生成100个虚拟节点与执行器地址对应。然后对任务id进行md5 hash，根据hash值从虚拟节点与执行器地址对应关系获取对应的执行器地址返回。

### ExecutorRouteRandom

```java
//代码位置：com.xxl.job.admin.core.route.strategy.ExecutorRouteRandom#route
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = addressList.get(localRandom.nextInt(addressList.size()));
        return new ReturnT<String>(address);
 }
```

ExecutorRouteRandom的route方法从执行器地址列表随机返回一个执行器地址。

### ExecutorRouteRound

```java
//代码位置：com.xxl.job.admin.core.route.strategy.ExecutorRouteRound#route
public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = addressList.get(count(triggerParam.getJobId())%addressList.size());
        return new ReturnT<String>(address);
}

private static int count(int jobId) {
        // cache clear
        //每一天都清理一次缓存
        if (System.currentTimeMillis() > CACHE_VALID_TIME) {
            routeCountEachJob.clear();
            //当前时间加上一天时间（1000*60*60*24）
            CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
        }

        //从缓存中获取
        AtomicInteger count = routeCountEachJob.get(jobId);
        //不存在或者大于100百万次数
        if (count == null || count.get() > 1000000) {
            // 初始化时主动Random一次，缓解首次压力
            count = new AtomicInteger(new Random().nextInt(100));
        } else {
            // count++
            count.addAndGet(1);
        }
        routeCountEachJob.put(jobId, count);
        return count.get();
}
```

ExecutorRouteRound的route方法从执行器地址列表中轮询获取一个执行器地址返回。count方法就是轮询的次数，将轮询的次数对执行器地址列表取余得到执行器地址在执行器地址列表中索引下标。

### ExecutorRouteLFU、ExecutorRouteLRU

最不经常使用路由、最近最久未使用路由分别使用HashMap、LinkedHashMap实现，具体代码也不贴 ，主要是因为有点懒，写到这里就不想写了，想去睡觉了。