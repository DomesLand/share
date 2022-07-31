# Android 后台任务



# 前言

1. 为什么需要后台任务

   - 执行耗时任务(从服务器拉取数据或将数据保存到本地数据库时)
   - 执行复杂计算，比如计算你爸爸的哥哥的叔叔的女儿的孩子应该怎么称呼你？等等

2. 在主线程执行这些任务的后果

   - 抛出`NetworkOnMainThreadException`
   - 界面响应卡顿，jankframe
   - App Not Responding(ANR) 

3. Android执行后台任务的方式

   - **线程**

     线程是最直接的解决方式，其它的方式都是对线程的封装。我们可以创建工作线程执行任务，从而不阻塞主线程

     ![](../../../../imgs/thread-lifecycle-diagram-in-java.jpg)

   - **线程池**

     线程池是一种基于池化思想来管理线程的工具。线程过多会带来额外的开销，包括创建销毁线程的开销、调度线程的开销，降低来计算机的整体性能。线程池维护多个线程，等待监督管理者分配可并发执行的任务。
   
     线程池的运行机制：
   
     ![](../../../../imgs/1713ee01cd341705tplv-t2oaga2asx-zoom-in-crop-mark3024000.webp)
   
     https://img-blog.csdnimg.cn/fbc235bc9d254b33950ff79846cc9a3e.png
   
     
   
     ThreadPoolExecutor
   
     ```java
         public ThreadPoolExecutor(int corePoolSize,
                                   int maximumPoolSize,
                                   long keepAliveTime,
                                   TimeUnit unit,
                                   BlockingQueue<Runnable> workQueue,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler)
     ```
   
     
   
     corePoolSize: 核心线程数，线程池创建线程时，如果当前线程总数小于核心线程数，新建的是核心线程；超过时新建的是非核心线程，核心线程默认情况下会一直存活(设置来allowCoreThreadTimeOut为true会被销毁掉)
   
     maximumPoolSize: 线程总数最大值，线程总数为核心线程数加非核心线程数
   
     keepAliveTime: 非核心线程空闲时超时时长(设置来allowCoreThreadTimeOut为true时，也会作用于核心线程数)
   
     workQueue: 任务队列，维护着等待执行的Runnable对象。当核心线程数满时会添加到该队列中等待处理。

     > SynchronousQueue: 接收到任务时，直接提交线程处理。如果核心线程数满则创建新线程执行任务，为了避免因线程数达到最大值而无法创建线程，使用该队列时要设置线程总数最大值为无限大
     >
     > LinkedBlockingQueue: 因为该队列没有最大值限制，所以所有超过核心线程数的任务都会被添加到队列中
     >
     > ArrayBlockingQueue: 有长度限制
     >
     > DelayQueue: 接收到任务时先入队，到达了指定的延时时间后才执行任务
   
     > 泛型参数是`Runnable`类型，但是也支持添加`Callable`，是因为ThreadPoolExecutor的父类中将`Callable`转换成了`Runnable`的子类`FutureTask`
   
     ThreadFactory: 定义线程创建的方式
   
     RejectedExecutionHandler: 提交任务数超过最大线程总数与workQueue之和时的执行策略
   
     > AbortPolicy: 抛出异常，默认
     >
     > CallerRunsPolicy: 在调用这所在线程重试任务
     >
     > DiscardOldestPolicy: 丢弃任务队列中最久的任务
     >
     > DisardPolicy: 丢弃当前任务

     处理流程
   
     ```mermaid
     graph LR;
     a(提交任务) --> b{核心线程是否已满?}
     b --否-->c[创建线程执行任务]
     b --是-->d{任务队列是否已满?}
     d --否-->e[将任务存储在队列中]
     d --是--> f{线程池是否已满?}
     f --否--> g[创建线程执行任务]
     f --是--> h[按照策略处理无法执行的任务]
     ```
   
     ```java
         public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
             return new ThreadPoolExecutor(nThreads, nThreads,
                                           0L, TimeUnit.MILLISECONDS,
                                           new LinkedBlockingQueue<Runnable>(),
                                           threadFactory);
         }
     ```
   
     > 这是一种固定大小的线程池，核心线程数和最大线程数都是外部设置的值，且任务队列是一个无界队列。
   
     线程池状态
   
     源码
   
     选择合适的线程池：
   
     1. CPU密集型
   
        CPU使用率高，开过多的线程会增加CPU调度次数，一般为核心数+1
   
     2. IO密集型
   
        CPU使用率不高，主要在存储的写入，一般为核心数的2倍
   
     缺点：
   
     - 对于复杂任务支持不够好
     - 需要考虑Activity,Fragment等的生命周期
     - 如果在Activity中直接使用匿名线程类，会引起内存泄漏
     - 需要使用Handler进行线程间交互
   
   - **AsyncTask**
   
     优点：
   
     - 直接重写其中的方法就可以使用它
     - `AsyncTask.execute`是静态方法，更方便进行使用
     - 提供毁掉用于任务状态的更新通知
   
     缺点：
   
     - 执行单一任务，执行关联任务时就会产生回调地狱
     - 任务默认是串行执行的，需要并行的话要调用`executeOnExecutor`
     - 一个AsyncTask的实例只能执行一次任务，如果要再执行一次就需要创建新的实例
   
   - **Service**
   
   - **IntentService**
   
     IntentService是一个包含工作线程的Service,将会在任务完成后终止
   
     优点：
   
     - 简单易用
     - 支持在不同的进程运行
     - App关掉之后也可以执行任务
     - 优先级高，不易被杀死
   
     缺点：
   
     - 任务保存在队列中，串行执行
     - 默认无法与主线程进行交互
   
   - **Loader**
   
     优点：
   
     - 通过回调简化线程管理
     - 可以监听数据变化
     - Android提供了多种实现方式
   
     缺点：
   
     - 依赖于Activity或Fragment的生命周期
     - 不保证任务完成
   
     **JobService**
   
     优点：
   
     - 可以高性能的处理批次任务
     - 可以设置任务执行约束
   
     缺点：
   
     - 需要满足约束条件才启动，可能不适合对时间苛刻的任务
   
   - **WorkManager**
   
   - **协程**