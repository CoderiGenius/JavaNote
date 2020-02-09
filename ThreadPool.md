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
#### 拒绝策略-handler
所谓拒绝策略，就是当线程池满了、队列也满了的时候，我们对任务采取的措施。或者丢弃、或者执行、或者其他...

jdk自带4种拒绝策略，我们来看看。

- CallerRunsPolicy // 在调用者线程执行
- AbortPolicy // 直接抛出RejectedExecutionException异常
- DiscardPolicy // 任务直接丢弃，不做任何处理
- DiscardOldestPolicy // 丢弃队列里最旧的那个任务，再尝试执行当前任务

这四种策略各有优劣，比较常用的是DiscardPolicy，但是这种策略有一个弊端就是任务执行的轨迹不会被记录下来。所以，我们往往需要实现自定义的拒绝策略， 通过实现RejectedExecutionHandler接口的方式。

### 提交任务的几种方式
- execute()用于提交不需要返回结果的任务，我们看一个例子。
- submit()用于提交一个需要返回果的任务。该方法返回一个Future对象，通过调用这个对象的get()方法，我们就能获得返回结果。get()方法会一直阻塞，直到返回结果返回。另外，我们也可以使用它的重载方法get(long timeout, TimeUnit unit)，这个方法也会阻塞，但是在超时时间内仍然没有返回结果时，将抛出异常TimeoutException。

### 关闭线程池
- shutdown()会将线程池状态置为SHUTDOWN，不再接受新的任务，同时会等待线程池中已有的任务执行完成再结束。
- shutdownNow()会将线程池状态置为SHUTDOWN，对所有线程执行interrupt()操作，清空队列，并将队列中的任务返回回来。
### 如何正确配置线程池的参数
- 任务的性质：CPU密集型、IO密集型和混杂型
- 任务的优先级：高中低
- 任务执行的时间：长中短
- 任务的依赖性：是否依赖数据库或者其他系统资源
### 线程池监控
首先，ThreadPoolExecutor自带了一些方法。
- long getTaskCount()，获取已经执行或正在执行的任务数
- long getCompletedTaskCount()，获取已经执行的任务数
- int getLargestPoolSize()，获取线程池曾经创建过的最大线程数，根据这个参数，我们可以知道线程池是否满过
- int getPoolSize()，获取线程池线程数
- int getActiveCount()，获取活跃线程数（正在执行任务的线程数）

例子：
```
ThreadPoolExecutor  threadPoolExecutor=(ThreadPoolExecutor) executor;
threadPoolExecutor.getPoolSize();
```

其次，ThreadPoolExecutor留给我们自行处理的方法有3个，它在ThreadPoolExecutor中为空实现（也就是什么都不做）。

- protected void beforeExecute(Thread t, Runnable r) // 任务执行前被调用
- protected void afterExecute(Runnable r, Throwable t) // 任务执行后被调用
- protected void terminated() // 线程池结束后被调用

例子：
```
public class ThreadPoolTest {
    public static void main(String[] args) {
        ExecutorService executor = new ThreadPoolExecutor(1, 1, 1, TimeUnit.SECONDS, new ArrayBlockingQueue<>(1)) {
            @Override protected void beforeExecute(Thread t, Runnable r) {
                System.out.println("beforeExecute is called");
            }
            @Override protected void afterExecute(Runnable r, Throwable t) {
                System.out.println("afterExecute is called");
            }
            @Override protected void terminated() {
                System.out.println("terminated is called");
            }
        };

        executor.submit(() -> System.out.println("this is a task"));
        executor.shutdown();
    }
}
```
### 一个特殊的问题
在使用submit()的时候一定要注意它的返回对象Future，为了避免任务执行异常被吞掉的问题，我们需要调用Future.get()方法。另外，使用execute()将不会出现这种问题。
