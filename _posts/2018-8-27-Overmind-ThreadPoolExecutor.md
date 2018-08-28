---
layout: post
title: 通过线程池创建并行任务提速任务
---

最近一年来我的全部精力都投在了Overmind的环境治理模块中，最近会抽空写一系列文章对其中涉及到的知识点由浅入深地进行梳理和回顾。

今天要分享的主题是，通过线程池创建并行任务提速任务。

### 问题背景
我们的模块中有一个功能原先的实现方式是这样的：挑选出一组合适的集群之后，调用第三方服务的接口，挨个执行“修改构建配置”、“新增实例”、“绑定服务器”操作，如下图所示:

![线程池背景说明]({{ site.baseurl }}/images/overmind_threadpool.png "线程池背景说明")

代码也很简单，把相关业务逻辑套在for循环中，挨个执行即可。

当集群的数量比较少的时候，这段代码执行起来问题还不大，但是当一个环境下包含10余个集群，每个集群都要挨个调用第三方接口去操作的时候，问题出现了：代码耗时太长。

### 并行
由于对于每个集群的处理逻辑相同，很容易就能想到，通过Java线程池来处理这类需求。

> 线程池配置，线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式。这样的处理方式让写的同学更加明确线程池的运行规则, 规避资源耗尽的风险。

上面这段话是阿里巴巴的Java开发手册里提到的，所以我们手工创建一个线程池即可。

```
@Configuration
public class ExecutorConfig {

    private static Logger logger = LoggerFactory.getLogger(ExecutorConfig.class);

    private int corePoolSize = 20;
    private int maxPoolSize = 30;

    private RejectedExecutionHandler handler = new RejectedExecutionHandler() {
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            if (!executor.isShutdown()) {
                try {
                    // 保证该task继续被尝试用阻塞式的put()到workQueue中。
                    executor.getQueue().put(r);
                } catch (InterruptedException e) {
                    logger.error("Executor rejectedExecution interrupted", e);
                }
            }
        }
    };

    // 用于部署的线程池
    @Bean
    public ThreadPoolTaskExecutor deployExecutor() {
        return getGeneralExecutor(corePoolSize, maxPoolSize, "DeployExecutor-");
    }
}
```

然后将相关业务逻辑封装成一个独立的类DeployClusterTask，由于我们需要知道单个任务的执行结果，所以可以implements Callable<String>接口，将结果以String返回即可。
```
List<String> errorMessage = new ArrayList<>();
List<Future<String>> results = new ArrayList<>();

for (int i = 0; i < appClusterList.size(); i++) {
    DeployClusterTask task = new DeployClusterTask(params...);
    Future<String> future = deployExecutor.submit(task);
    results.add(future);
}

// 自旋，获取结果
for (int i = 0; i < results.size(); i++) {
    Future<String> taskHolder = results.get(i);
    if (taskHolder.isDone()) {
        String result = taskHolder.get();
        results.remove(taskHolder);
        if (result != null) {
            errorMessage.add(result);
        }
        i--;
    }

    if (i == results.size() - 1) {
        i = -1;
    }
}
```

### 知识点
相关业务改造很快说完了，还是要记录一下常见的知识点。

1.应用与线程池的交互，线程池的内部工作过程示意图：

![线程池工作过程说明]({{ site.baseurl }}/images/overmind_threadpool2.png "线程池工作过程说明")

2.尽量避免在线程池中操作ThreadLocal

3.避免任务堆积，如果处理速度跟不上入队速度，可能占用大量系统内存，甚至造成oom

4.如果任务逻辑有问题，则会导致工作线程迟迟不能被释放，可以使用jstack排查线程栈



