目录

## xxl-job执行器启动流程分析

在《[调度中心启动流程分析](https://zhuanlan.zhihu.com/p/435419514)》这篇文章中，分析了调度中心启动的流程，即启动了一个springboot的admin后台管理服务，通过界面管理调度器、执行任务、日志等。启动好admin服务以后，就可以启动执行器，执行器一般就是开发写的定时任务，官方给了两个执行器启动的列子，普通的以及集成spring的案例。我们这片文章只分析执行器普通的启动。集成spring也是一样的差不多的。

![img](https://pic4.zhimg.com/80/v2-812d8562ebd40603aa6cdbdebeb4d96f_1440w.webp)

我们只分析xxl-job-executor-sample-frameless，xxl-job-executor-sample-frameless的项目结构如下：

![img](https://pic4.zhimg.com/80/v2-97d443afe5ae63848731cc396985947b_1440w.webp)

```java
//代码位置：com.xxl.job.executor.sample.frameless.FramelessApplication#main
FrameLessXxlJobConfig.getInstance().initXxlJobExecutor();
```

在FramelessApplication类的main方法中，上述代码即为启动任务执行器的代码。我们深入initXxlJobExecutor方法看看做了哪些工作：

### 初始化执行器并启动

```java
public void initXxlJobExecutor() {

        // load executor prop
        //加载xxl-job-executor.properties文件
        Properties xxlJobProp = loadProperties("xxl-job-executor.properties");

        // init executor
        //创建普通的任务执行器
        xxlJobExecutor = new XxlJobSimpleExecutor();
        xxlJobExecutor.setAdminAddresses(xxlJobProp.getProperty("xxl.job.admin.addresses"));
        xxlJobExecutor.setAccessToken(xxlJobProp.getProperty("xxl.job.accessToken"));
        xxlJobExecutor.setAppname(xxlJobProp.getProperty("xxl.job.executor.appname"));
        xxlJobExecutor.setAddress(xxlJobProp.getProperty("xxl.job.executor.address"));
        xxlJobExecutor.setIp(xxlJobProp.getProperty("xxl.job.executor.ip"));
        xxlJobExecutor.setPort(Integer.valueOf(xxlJobProp.getProperty("xxl.job.executor.port")));
        xxlJobExecutor.setLogPath(xxlJobProp.getProperty("xxl.job.executor.logpath"));
        xxlJobExecutor.setLogRetentionDays(Integer.valueOf(xxlJobProp.getProperty("xxl.job.executor.logretentiondays")));

        // registry job bean
        //注册定时任务的bean
        xxlJobExecutor.setXxlJobBeanList(Arrays.asList(new SampleXxlJob()));

        // start executor
        try {
            //启动执行器
            xxlJobExecutor.start();
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
}
```

initXxlJobExecutor方法首先加载xxl-job-executor.properties文件，接着创建普通的任务执行器XxlJobSimpleExecutor，并将从xxl-job-executor.properties文件读取的值设置XxlJobSimpleExecutor的属性。如admin的地址、执行器的名字、执行器的地址、执行器的ip、执行器的ip等。设置完XxlJobSimpleExecutor的属性以后，注册定时任务的bean，即将SampleXxlJob加入到list中去，SampleXxlJob类中有一系列由@XxlJob注解修饰的方法。这些@XxlJob注解修饰的方法就是定时任务。最后就是启动执行器了，接下来我们看看执行器的启动start方法：

```java
//代码位置：com.xxl.job.core.executor.impl.XxlJobSimpleExecutor
public void start() {

        // init JobHandler Repository (for method)
        //初始化任务处理器
        initJobHandlerMethodRepository(xxlJobBeanList);

        // super start
        try {
            //调用父类的start方法
            super.start();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
}
```

initJobHandlerMethodRepository方法的作用是初始化任务处理器，xxlJobBeanList就是保存定时任务bean的list，initJobHandlerMethodRepository方法遍历xxlJobBeanList中的定时任务bean，获取用@XxlJob修饰的定时任务方法，用反射的方式创建MethodJobHandler处理器并注册处理器到jobHandlerRepository保存起来。

调用父类的start方法启动执行器，父类的start方法如下：

```java
//代码位置：com.xxl.job.core.executor.XxlJobExecutor#start
public void start() throws Exception {

        // init logpath
        //初始化日志路径
        XxlJobFileAppender.initLogPath(logPath);

        // init invoker, admin-client
        //初始化admin的客户端
        initAdminBizList(adminAddresses, accessToken);


        // init JobLogFileCleanThread
        //初始化日志清理线程
        JobLogFileCleanThread.getInstance().start(logRetentionDays);

        // init TriggerCallbackThread
        //初始化回调线程池
        TriggerCallbackThread.getInstance().start();

        // init executor-server
        //初始化执行器服务
        initEmbedServer(address, ip, port, appname, accessToken);
}
```

父类的start方法如下做了下面几件事：

- 初始化日志路径
- 初始化admin的客户端
- 初始化日志清理线程
- 初始化回调线程池
- 初始化执行器服务

### 初始化日志路径

```java
//代码位置：com.xxl.job.core.log.XxlJobFileAppender#initLogPath
public static void initLogPath(String logPath){
        // init
        if (logPath!=null && logPath.trim().length()>0) {
            logBasePath = logPath;
        }
        // mk base dir
        //创建父类目录
        File logPathDir = new File(logBasePath);
        if (!logPathDir.exists()) {
            logPathDir.mkdirs();
        }
        logBasePath = logPathDir.getPath();

        // mk glue dir
        //创建脚本代码目录
        File glueBaseDir = new File(logPathDir, "gluesource");
        if (!glueBaseDir.exists()) {
            glueBaseDir.mkdirs();
        }
        glueSrcPath = glueBaseDir.getPath();
}
```

initLogPath方法首先创建了保存日志的文件目录，然后在创建保存脚本代码的文件目录。

### 初始化admin的客户端

```java
//代码位置： com.xxl.job.core.executor.XxlJobExecutor#initAdminBizList
private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
        //遍历调度器的地址
        if (adminAddresses!=null && adminAddresses.trim().length()>0) {
            //以逗号分隔
            for (String address: adminAddresses.trim().split(",")) {
                if (address!=null && address.trim().length()>0) {

                    //初始化admin客户端
                    AdminBiz adminBiz = new AdminBizClient(address.trim(), accessToken);

                    if (adminBizList == null) {
                        adminBizList = new ArrayList<AdminBiz>();
                    }
                    //保存到list中
                    adminBizList.add(adminBiz);
                }
            }
        }
}
```

initAdminBizList方法遍历调度器的地址，即admin服务的地址，adminAddresses地址是以逗号分隔的，每一个admin地址就创建admin客户端AdminBizClient，AdminBizClient是执行器与admin服务通信的客户端，是通过http进行通信的，通过AdminBizClient执行器就可以注册执行器、删除执行器以及将定时任务的结果回调给admin保存起来。最后将创建好的AdminBizClient保存到adminBizList中。AdminBizClient实现了AdminBiz接口，AdminBiz接口的类图如下：

![img](https://pic4.zhimg.com/80/v2-dcaf8a88d696024176f729fc5ae712d3_1440w.webp)

AdminBiz接口有三个方法，registry方法用来注册执行器的，registryRemove方法用来删除执行器的，callback方法是将定时任务的执行的结果回调保存到数据库中。**AdminBizClient是admin客户端，AdminBizImpl是admin服务端，AdminBizClient的三个方法通过http请求转发给AdminBizImpl对应的是三个方法**。

### 初始化日志清理线程

启动一个线程localThread，用来清理过期的日志文件。localThread的run方法一直执行，首先获取所有的日志文件目录，日志文件形式如logPath/yyyy-MM-dd/9999.log，获取logPath/yyyy-MM-dd/目录下的所有日志文件，然后判断日志文件是否已经过期，过期时间是配置的，如果当前时间减去日志文件创建时间（yyyy-MM-dd）大于配置的日志清理天数，说明日志文件已经过期，一般配置只保存30天的日志，30天以前的日志都删除掉。

### 初始化回调线程池

启动了两个线程，一个是triggerCallbackThread回调线程，一个是triggerRetryCallbackThread重试回调线程，triggerCallbackThread线程的作用就是将定时任务执行的结果回调给admin保存在数据库中，调用AdminBizClient的callback方法回调写回admin服务的数据库中。triggerRetryCallbackThread线程是将错误的回调重新进行回调，跟triggerCallbackThread线程是一样的，只是triggerRetryCallbackThread只重新回调错误的回调。定时任务运行完成以后，将运行以后的结果保存在队列中，每次回调都是从队列中获取定时任务的结果写回admin服务，是通过http去写回到admin服务的。

### 初始化执行器服务

启动了一个netty服务器，用于执行器接收admin的http请求。主要接收admin发送的空闲检测请求、运行定时任务的请求、停止运行定时任务的请求、获取日志的请求。最后a还向dmin注册了执行器，注册执行器是调用AdminBizClient的registry方法注册的，AdminBizClient的registry方法通过http将注册请求转发给admin服务的AdminBizImpl类的registry方法，AdminBizImpl类的registry方法将注册请求保存在数据库中。

执行器服务接收admin服务的请求，交给ExecutorBiz接口处理，ExecutorBiz接口的类图如下：

![img](https://pic3.zhimg.com/80/v2-8f2df1e4655abe7226dab510b94bcabe_1440w.webp)

ExecutorBiz接口有五个方法，分别是beat（心跳检测）、idleBeat（空闲检测）、run（运行定时任务）、kill（停止运行任务）、log（获取日志）。ExecutorBiz接口有两个实现：ExecutorBizClient和ExecutorBizImpl，ExecutorBizClient是执行器客户端，ExecutorBizImpl执行器服务端。admin服务通过ExecutorBizClient类的方法通过http将请求转发给执行器服务的ExecutorBizImpl对应的方法。