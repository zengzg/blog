**Executor Interfaces**

三种线程管理接口，所有任务都需要提交给它们之一，才能得到执行：

 - `Executor`——启动（或执行）新任务
 - `ExecutorService`——扩展`Executor`，支持管理任务和自身生命周期
 - `ScheduledExecutorService`——扩展`ExecutorService`，支持日程（或计划任务）

其中`ExecutorService`管理任务的生命周期体现在通过`submit`方法返回的`Future`类上，`Future`定义有取消任务和获取任务结果的方法。
`ScheduledExecutorService`中的日程指的是可以延时或周期性执行任务。

![enter image description here](https://github.com/zengzg/blog/blob/master/java/images/java_concurrent_1.png?raw=true)

**Thread Pools**

上面的线程管理接口（executor）的实现有：

 - `ThreadPoolExecutor`——维护用于任务执行需要的线程池
 - `ScheduledThreadPoolExecutor`——支持日程的`ThreadPoolExecutor`

上面所示的*Executor*可以通过`Executors`工厂类来创建配置。

![enter image description here](https://github.com/zengzg/blog/blob/master/java/images/java_concurrent_2.png?raw=true)

 **Fork/Join**

为了更好地利用现代计算机的多核架构，提供Fork/Join型线程池：`ForkJoinPool`。如果任务可以分解为更小的多个子任务，并允许子任务**并行运行**，最终将子任务结果合并便为原任务结果。对于这种情况就可以使用`ForkJoinPool`来实现。该线程池提供了一种在多线程下实现分治算法的方案。在类`Arrays`中的`parallelSort()`方法正是其应用场景。
运行在`ForkJoinPool`中的任务由特殊的类来描述，并在特殊的线程中运行：

 - `ForkJoinTask`——可分解任务
 - `ForkJoinWorkerThread`——运行`ForkJoinTask`任务的线程

![enter image description here](https://github.com/zengzg/blog/blob/master/java/images/java_concurrent_3.png?raw=true)

> Written with [StackEdit](https://stackedit.io/).
