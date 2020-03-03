## AtomicInteger
- 原子类相比于普通的锁，粒度更细、效率更高(除了高度竞争的情况下)
- 阻塞同步和非阻塞同步都是实现线程安全的两个保障手段
- 非阻塞同步对于阻塞同步而言主要解决了阻塞同步中线程阻塞和唤醒带来的性能问题
### 阻塞同步（悲观锁）
- synchronized关键字和可重入锁ReentrantLock是两种最为常用的互斥同步手段。
#### synchronized
- 经过编译之后会在同步块前后分别形成monitorenter和monitorexit这两个字解码命令
- 这两个命令都想需要一个reference类型的参数来指明要锁和解锁的对象
  - 如果synchronized明确指定了对象参数，那么就是这个对象的reference，
  - 如果没有指定，那么根据synchronized修饰的是实例方法还是类方法去取对应的对象实例或Class对象来作为锁对象。
- 在执行monitorenter命令时，首先尝试获取对象的锁，如果成功获取，就把锁的计数器加一；
- 相应的，执行monitorexit会将锁的计数器减一。如果获取对象失败，该线程就进入阻塞状态，直到对象锁被另一个线程释放为止。

- 注意一些情况：
  - synchronized同步块对同一线程来说是可重入的，不会出现自己把自己锁死的问题；
  - 同步块在已进入线程执行完之前，会阻塞后面线程的进入；
  - Java线程是映射到操作系统的原生线程上的，如果要阻塞和唤醒一个线程，都要操作系统来完成，这需要从用户态转到核心态，会消耗很多处理器时间，因此synchronized是一个重量级的操作。
#### 可重入锁ReentrantLock：
- 在用法上，ReentrantLock和synchronized很相似都具备一样的线程重入特性，但前者表现为API层面的互斥锁，后则表现为原生语法层面的互斥锁。不过，ReentrantLock比synchronized增加了一些高级功能：

- **等待可中断**：是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以放弃等待去做其他事情。
- **公平锁**：是指多个线程在等待同一个锁时，必须按照申请锁的顺序来依次获得锁。synchronized中的锁是非公平的，ReentrantLock默认也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。
- **锁绑定多个条件**：是指一个ReentrantLock对象可以同时绑定多个Condition对象，只需多次调用newCondition()方法即可。
- 若要使用上面的三种功能，ReentrantLock是很好的选择。但一般情况下使用synchronized就可以了。JDK1.5之前多线程环境下ReentrantLock要比synchronized效率高，然而JDK1.6引入锁优化之后，两者效率已经很接近。
### 非阻塞同步(乐观锁)
- 什么叫做非阻塞同步呢？
    - 在并发环境下，某个线程对共享变量先进行操作
    - 如果没有其他线程争用共享数据那操作就成功；
    - 如果存在数据的争用冲突，那就才去补偿措施，比如不断的重试机制，直到成功为止
    - 因为这种乐观的并发策略不需要把线程挂起，也就把这种同步操作称为非阻塞同步
    - （操作和冲突检测具备原子性）
### CAS指令（Compare-And-Swap比较并交换）
![](https://img-blog.csdnimg.cn/20190111101332407.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZhbnJlbnhpYW5n,size_16,color_FFFFFF,t_70)
### 再返回来看AtomicInteger.incrementAndGet()方法
```
/**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }

```
incrementAndGet()方法在一个无限循环体内，不断尝试将一个比当前值大1的新值赋给自己，如果失败则说明在执行"获取-设置"操作的时已经被其它线程修改过了，于是便再次进入循环下一次操作，直到成功为止。这个便是AtomicInteger原子性的"诀窍"了
#### 再往下看它的compareAndSet方法:
```
/**
    * Atomically sets the value to the given updated value
    * if the current value {@code ==} the expected value.
    *
    * @param expect the expected value
    * @param update the new value
    * @return true if successful. False return indicates that
    * the actual value was not equal to the expected value.
    */
   public final boolean compareAndSet(int expect, int update) {
       return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
   }

```
可以看到，compareAndSet()调用的就是Unsafe.compareAndSwapInt()方法，即Unsafe类的CAS操作

## 锁优化：
#### 自旋锁与自适应自旋：
- 因为线程阻塞和唤醒要消耗大量处理器时间，所以在一些情况下，可以让要等待的线程“稍等一下”，但不放弃处理器，看看持有锁的线程是否会马上释放锁。为了让线程占有处理器等待，只需让线程执行一个忙循环（自旋），这就是自旋锁。

- 自旋锁不能代替阻塞，因为它是占用处理器时间的，如果锁被占用的时间很短，自旋锁效果会很好，但如果锁被占用时间很长，那自旋线程就会白白消耗处理器资源。所以自旋锁一般会指定自选次数，默认10次。

- **自适应自旋锁** 是自选时间时间不固定，而是由前一次在同一个锁上的自旋线程的自旋时间以及锁的拥有者状态来决定。如果前一次的自旋线程刚刚成功获得锁，那么虚拟机认为这次也会容易获得锁，进而允许自旋线程多自旋几次比如100次；而如果对于某个锁自旋很少成功过，那么以后的线程可能直接忽略掉自旋过程。
#### 锁清除：
- 锁清除是指虚拟机即时编译器在运行时，会将代码上要求同步，但被检测到实际上不可能出现共享数据竞争的锁进行清除。锁清除的主要判定依据来源于逃逸分析的数据支持。

#### 锁粗化：
很多情况下，总是推荐将同步代码块的范围限制得越小越好。但在一些情况下，如果一系列连续操作都对同一个对象反复加锁和解锁，甚至加锁解锁关系出现在循环体中，那么也会消耗性能。如果虚拟机探测到有这样的操作，就会把加锁同步的范围扩展（粗化）到整个操作序列之外。

#### 轻量级锁：
“轻量级”是相对于使用系统互斥量实现的传统锁而言的，因此传统的锁机制就是重量级锁。强调一点是：轻量级锁不是为了取代重量级锁的，而是在没有多线程竞争的前提下，减少传统重量级锁使用操作系统互斥量产生的性能消耗。

轻量级锁提升系统同步性能的依据是“对于绝大多数锁，在整个同步周期内是不存在竞争的”，这是一个经验数据。但如果存在竞争，除了传统锁互斥量的开销外，还额外发生了CAS操作，因此会更慢。

#### 偏向锁：
如果说轻量级锁是在无竞争的情况下使用CAS操作消除同步使用的互斥量，那么偏向锁就是在无竞争的情况下把整个同步都消除掉。偏向锁的意思是这个锁会偏向于第一个获取它的线程，如果在接下来的执行过程中，该锁没有被其他线程获取，则持有偏向锁的线程将永远不需要再进行同步。当有另一个线程去尝试获取这个锁时，偏向模式结束。
## volatile
