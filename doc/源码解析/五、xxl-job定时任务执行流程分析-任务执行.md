在《[xxl-job定时任务执行流程分析-客户端触发](https://zhuanlan.zhihu.com/p/436886892)》和《[xxl-job定时任务执行流程分析-服务端执行](https://zhuanlan.zhihu.com/p/437377126)》这两篇文章分别介绍了xxl-job定时任务的客户端触发和服务端执行的流程，而这篇文章深入到定时任务是如何执行的，在服务端执行的流程中，将任务交给任务线程池JobThread执行，JobThread的run方法主要做了几件事：

- 处理器的初始化
- 任务的执行
- 销毁清理工作

### 处理器的初始化

```java
//代码位置：com.xxl.job.core.thread.JobThread#run
handler.init();
```

处理器的初始化比较简单，调用IJobHandler的init方法，IJobHandler是接口类型，有三种方法，分别是init（初始化方法）、execute（执行方法）、destroy（销毁）方法。IJobHandler接口将在下面具体分析。

### 任务的执行

```java
//代码位置:com.xxl.job.core.thread.JobThread#run
while(!toStop){
    //1、从队列中触发参数
    triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);
    //2、如果存在执行超时时间并大于0，则在规定的时间异步执行，否则立即执行
    if (triggerParam.getExecutorTimeout() > 0) {
        FutureTask<Boolean> futureTask = new FutureTask<Boolean>(new Callable<Boolean>() {
            @Override
            public Boolean call() throws Exception {

                // init job context
                XxlJobContext.setXxlJobContext(xxlJobContext);
                //处理器执行方法
                handler.execute();
                return true;
              }
         });
         futureThread = new Thread(futureTask);
         futureThread.start();

        //等待结果
         Boolean tempResult = futureTask.get(triggerParam.getExecutorTimeout(), TimeUnit.SECONDS);
    }else{
        // just execute 立即执行
        handler.execute();
    }

}
```

上述将任务执行的代码省略了很多，只将核心的代码抽取出来。任务的执行是不断执行的，只有当任务停止了（toStop设置为ture），才跳出while循环。首先从triggerQueue队列中弹出触发参数，如果存在执行超时时间并大于0，则在规定的时间异步调用handler的execute方法执行任务，否则立即调用handler的execute方法执行任务。

### 销毁清理工作

```java
//代码位置:com.xxl.job.core.thread.JobThread#run  
//如果任务停止了，需要将队列中所有的触发删除（所有定时任务删除）
while(triggerQueue !=null && triggerQueue.size()>0){
     //从队列中获取触发参数
    TriggerParam triggerParam = triggerQueue.poll();
}

//执行处理器的销毁方法
try {
    handler.destroy();
} catch (Throwable e) {
    logger.error(e.getMessage(), e);
}
```

如果定时任务停止了，将队列中所有的触发参数都删除，最后执行处理器handler的销毁方法。

### 执行方法

上述定时任务执行过程调用处理器的init（初始化方法）、execute（执行方法）、destroy（销毁）方法，这些方法是由IJobHandler抽象类的实现类实现的，IJobHandler抽象类的类图如下所示：

![img](https://pic1.zhimg.com/80/v2-7823758399cb44f630ecb6f14ffe404c_1440w.webp)

IJobHandler抽象类有三个子类，GlueJobHandler是执行groovy的处理器，MethodJobHandler是执行java的处理器，ScriptJobHandler是执行脚本（Pyhotn、PHP、NodeJS、Shell等）的处理器。

MethodJobHandler是处理java定时任务的方法，当我们用java开发了定时任务方法，然后用@XxlJob注解修饰方法，就可以调度该定时任务方法了。我们看看MethodJobHandler的execute方法是如何执行定时任务方法的。

```java
//代码位置：com.xxl.job.core.handler.impl.MethodJobHandler#execute
public void execute() throws Exception {
        Class<?>[] paramTypes = method.getParameterTypes();
        if (paramTypes.length > 0) {
            method.invoke(target, new Object[paramTypes.length]);       // method-param can not be primitive-types
        } else {
            method.invoke(target);
        }
}
```

MethodJobHandler的execute方法利用反射，获取定时任务的method，然后利用invoke执行定时任务方法。

GlueJobHandler是执行groovy的处理器，在admin界面的idea界面上写好groovy保存在数据库，会调用GlueJobHandler类的execute方法执行，groovy是一种基于JVM的开发语言，*groovy* 代码能够与 Java 代码很好地结合，也能用于扩展现有代码。GlueFactory类的loadNewInstance方法将写好的groovy加载解析为写好的groovy代码，并返回IJobHandler，然后将IJobHandler传进GlueJobHandler构造器中新建GlueJobHandler对象。如下:

```java
IJobHandler originJobHandler = GlueFactory.getInstance().loadNewInstance(triggerParam.getGlueSource());
jobHandler = new GlueJobHandler(originJobHandler, triggerParam.getGlueUpdatetime());
```

我们看看GlueJobHandler的execute方法：

```java
public void execute() throws Exception {
        XxlJobHelper.log("----------- glue.version:"+ glueUpdatetime +" -----------");
        jobHandler.execute();
}
```

GlueJobHandler的execute方法就是调用GlueFactory工厂类创建的IJobHandler的execute方法。

最后来看看MethodJobHandler的execute方法：

```java
//代码位置：com.xxl.job.core.handler.impl.ScriptJobHandler#execute
public void execute() throws Exception {

     //1、获取执行脚本的命令
      String cmd = glueType.getCmd();
     //2、保存脚本到文件
      String scriptFileName = XxlJobFileAppender.getGlueSrcPath()
                .concat(File.separator)
                .concat(String.valueOf(jobId))
                .concat("_")
                .concat(String.valueOf(glueUpdatetime))
                .concat(glueType.getSuffix());
        File scriptFile = new File(scriptFileName);
        if (!scriptFile.exists()) {
            //将脚本写入文件中保存
            ScriptUtil.markScriptFile(scriptFileName, gluesource);
        }

    //3、执行脚本
    int exitValue = ScriptUtil.execToFile(cmd, scriptFileName, logFileName, scriptParams);

    //省略代码
}
```

上述MethodJobHandler的execute方法有些不重要的代码被省略。主要有几个重要的流程：**获取脚本执行命令、保存脚本命令到文件、执行脚本**。

ScriptJobHandler处理器有一个GlueTypeEnum属性，获取脚本执行命令实际就是获取GlueTypeEnum的cmd属性，GlueTypeEnum枚举如下：

![img](https://pic4.zhimg.com/80/v2-2e79b37febb5db9120c3f76d70a44b13_1440w.webp)

保存脚本到文件利用 ScriptUtil的markScriptFile方法，将脚本定时任务代码保存在名字为jobId_glueUpdatetime_suffix中，jobId为任务id，glueUpdatetime为脚本更新时间、suffix为脚本后缀，如采用python写定时任务时，保存在类似666-123456789.py文件中。

执行脚本ScriptUtil的execToFile方法，execToFile方法是利用**Runtime.getRuntime().exec()方法在java程序里运行脚本程序**。Runtime.getRuntime().exec()方法会将执行命令发送给操作系统，然后等待操作系统运行程序的结果返回，如Runtime.getRuntime().exec()方法给操作系统发送pyhton hello.py命令，这样就会执行python脚本，并等待python脚本的运行结果的返回。