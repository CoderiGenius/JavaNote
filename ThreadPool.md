# 线程池与源码阅读
> 本质上是一种对象池，用于管理线程资源。
在任务执行前，需要从线程池中拿出线程来执行。
在任务执行完成之后，需要把线程放回线程池。
通过线程的这种反复利用机制，可以有效地避免直接创建线程所带来的坏处。

## 好处
- 降低资源的消耗。线程本身是一种资源，创建和销毁线程会有CPU开销；创建的线程也会占用一定的内存。
- 提高任务执行的响应速度。任务执行时，可以不必等到线程创建完之后再执行。
- 提高线程的可管理性。线程不能无限制地创建，需要进行统一的分配、调优和监控。

## 不使用线程池有哪些坏处
- 频繁的线程创建和销毁会占用更多的CPU和内存
- 频繁的线程创建和销毁会对GC产生比较大的压力
- 线程太多，线程切换带来的开销将不可忽视
- 线程太少，多核CPU得不到充分利用，是一种浪费

## 线程池处理流程
![](https://upload-images.jianshu.io/upload_images/845143-19328763889448ab.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
![](https://upload-images.jianshu.io/upload_images/845143-b510ac8252bea486.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

## 手动创建线程池
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler);
```
### 参数含义
- corePoolSize 核心线程数量
- maximumpoolsize 线程池中最大的线程数量
- keepaliveTime 当除了核心线程之外的线程处于空闲状态，这些空闲的非核心线程的存活时间
- unit 这个是keepalivetime的单位，可以是毫秒、秒、分钟、小时和天，等等
- workQueue 等待队列，线程池中的线程数超过核心线程数时，任务将放在等待队列，它是一个BlockingQueue类型的对象
- threadFactory，线程工厂，我们可以使用它来创建一个线程
- handler，拒绝策略，当线程池和等待队列都满了之后，需要通过该对象的回调函数进行回调处理
#### 等待队列-workQueue
等待队列是BlockingQueue类型的，理论上只要是它的子类，我们都可以用来作为等待队列。
同时，jdk内部自带一些阻塞队列，我们来看看大概有哪些。
##### 阻塞队列与普通队列的区别：
当队列是空的时，从队列中获取元素的操作将会被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。试图从空的阻塞队列中获取元素的线程将会被阻塞，直到其他的线程往空的队列插入新的元素。同样，试图往已满的阻塞队列中添加新元素的线程同样也会被阻塞，直到其他的线程使队列重新变得空闲起来，如从队列中移除一个或者多个元素，或者完全清空队列.

- ArrayBlockingQueue，队列是有界的，基于数组实现的阻塞队列
- LinkedBlockingQueue，队列可以有界，也可以无界。基于链表实现的阻塞队列
- SynchronousQueue，不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作将一直处于阻塞状态。该队列也是Executors.newCachedThreadPool()的默认队列
- PriorityBlockingQueue，带优先级的无界阻塞队列
通常情况下，我们需要指定阻塞队列的上界（比如1024）。另外，如果执行的任务很多，我们可能需要将任务进行分类，然后将不同分类的任务放到不同的线程池中执行。
#### 线程工厂-threadFactory
ThreadFactory是一个接口，只有一个方法。既然是线程工厂，那么我们就可以用它生产一个线程对象。来看看这个接口的定义。
```
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```
##### 默认的线程工厂
```
static class DefaultThreadFactory implements ThreadFactory {
    //原子计数类
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    //线程组
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        //获取线程组
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```
