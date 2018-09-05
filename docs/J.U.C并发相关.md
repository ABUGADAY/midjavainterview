# J.U.C/并发相关

## 1.如何停止一个线程
- 使用 volatile 变量终止正常运行的线程 + 抛异常法 / Return法
- 组合使用 interrupt 方法与 interruptted/isinterrupted 方法终止正在运行的线程 + 抛异常法 / Return 法
- 使用 interrupt 方法终止正在阻塞中的线程

## 2.何为线程安全的类
在线程安全性的定义中，最核心的概念就是 **正确性** 。当多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类都能表现出正确的行为，那么这个类就是线程安全的。
## 3.为什么线程通信的方法 wait() notify() notifyAll() 被定义在 Object 类里？
```java
Object lock = new Object();
synchronized (lock) {
    lock.wait();
    ...
}
```

Wait-notify 机制是在获取对象锁的前提下不同线程间的通信机制。在 Java 中，任意对象都可以当作锁来使用，由于锁对象的任意性，所以这些通信方法需要被定义在 Object 类里。

## 4.为什么 wait() notify() 和 notifyAll() 必须在同步方法或者同步 块中被调用？
wait/notify 机制是依赖于 Java 中 Synchornize 同步机制的，其目的在于确保等待线程从 Wait() 返回时能沟感知通知线程对共享变量所作出的修改。如果不再同步范围内使用，就会抛出 **java.lang.IllegalMonitorStateException** 的异常。

## 5.并发三准则
- 异常不会导致死锁现象：当线程出现异常且没有捕获处理时，JVM 会字的喔咕释放当前线程所占用的锁，因此不会由于异常导致出现死锁现象，同时还会释放 CPU；
- 锁的是对象而非引用；
- 有 wait 必有 notify；

## 6.如何确保线程安全？
在 Java 中可以有很多方法来保证线程安全，诸如：
- 通过加锁（Lock/Synchronized）保证对临界资源的同步互斥访问；
- 使用 volatile 关键字，轻量级同步机制，但不保证原子性；
- 使用不变类 和 线程安全类（原子类，并发容器，同步容器等）。

## 7. volatile 关键字在Java中有什么作用
volatile 的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新，即保证了内存的可见性，除此之外还能禁止指令重排序。此外，synchronized 关键字也可以保证内存可见性。   
指令重排序问题在并发环境下会导致线程安全问题， volatile 关键字通过禁止指令重排序来避免这一问题。而对于 synchronized 关键字，其所控制范围内的程序在执行时独占的，指令重排序问题不会对其产生任何影响，因此无论如何，其都可以保证最终的正确性。

## 8. ThreadLocal 及其引发的内存泄露
ThreadLocal 是 Java 中的一种线程绑定机制，可以为每一个使用该变量的线程都提供一个变量值的副本，并且每一个线程都可以独立地改变自己的副本，而不会与其它线程的副本发生冲突。
每个线程内部有一个 ThreadLocal.ThreadLocalMap 类型的成语阿变量 threadLocals，这个 threadLocals 存储了与该线程相关的所有 ThreadLocal 变量以及其对应的值，也就是说， ThreadLocal 变量及其对应的值集iushi该Map 中的一个 Entry ，更直白地， threadLocals 中每个 Entry 的 key 是 ThreadLocal 变量本身， 而 Value 是该 ThreadLocal 变量对应的值。
### 1.ThreadLocal 可能引起的内存泄露
下面时 ThreadLocalMap 的部分源码，我们可以看出 ThreadLocalMap 里面对 Key 的引用是弱引用。那么，就存在这样的情况： 当释放调对 threadlocal 对象的强引用后， map 里面的 value 没有被回收，但却永远不会访问到了，因此 ThreadLocal 存在着内存泄露问题。
```java
static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }
        ...
    }
```
看下面的图示，实现代表强引用，虚线代表弱引用。每个 thread 中都存在一个 map ， map 的类型时上文提到的 ThreadLocal.ThreadLocalMap ，该 map 中的 key 为一个 ThreadLocal 实例。 这个 Map 的确使用了弱引用， 不过弱引用只是针对 key ， 每个 key 都弱引用指向 ThreadLocal 对象。 一旦把 threadlocal 实例置为 null 以后，那么将没有任何强引用指向 ThreadLocal 对象，因此 ThreadLocal 对象将会被 Java GC 回收。但是，与之关联的 value 却不能回收，因为存在一条从 current thread 连接过来的强引用。 只有当前 thread 结束以后， current thread 就不会存在栈中，强引用断开，Current Thread、Map 及 value 将全部被 Java GC 回收。
![img1](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc1.jpg)

所以，得出一个结论就是：只要这个线程对象被 Java GC 回收，就不会出现内存泄露。但是如果只把 ThreadLocal 引用指向 null 而线程对象依然存在，那么此时 Value 是不会被回收的，这就发生了我们认为的内存泄露。比如，在使用线程池的时候，线程结束是不会销毁的而是会再次使用的，这种情形下就可能出现 ThreadLocal 内存泄露。
Java 为了最小化减少内存泄露的可能性和影响，在 ThreadLocal 进行 get、set 操作时会清除线程 Map 里所有 key 为 null 的 value。所以最怕的情况就是，ThreadLocal 对象设 null 了，开始发生 “内存泄露”，然后使用线程池，线程结束后被放回线程池中而不销毁，那么如果这个线程一直不被使用或者分配使用了又不再调用 get/set 方法，那么这个期间就会发生真正的内存泄露。因此，最好的做法是：在不使用该 ThreadLocal 对象时，及时调用该对象的 remove 方法去移除 ThreadLocal.ThreadLocalMap 中的对应 Entry。



## 什么是死锁（DeadLock）？如何分析和避免死锁

死锁是指两个以上的线程永远阻塞的情况，这种情况产生至少需要两个以上的线程和两个以上的资源。
分析死锁，我们需要查看Java应用程序的线程转储。我们需要找出那些状态为 BLOCKED 的线程和他们等待的资源。每个资源都有一个唯一的 id ，用这个 id 我们可以找出哪些线程已经拥有了它的对象锁。下面列举了一些 JDK 自带的死锁检测工具：
1. Jconsole： JDK 自带的图形化界面工具，主要用于对  JAVA 应用程序做性能分析和调优。
![img2](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc2.png)

2. Jstack: JDK 自带的命令行工具，主要用于线程 Dump 分析。

![img3](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc3.png)

3. VisualVM: JDK 自带的图形化界面工具，主要用于对 JAVA 应用程序做性能分析和调优。

![img4](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc4.jpg)

## 10.什么是 Java Timer 类？ 如何创建一个有特定时间间隔的任务？

Timer 是一个调度器， 可以用于安排一个任务在未来的某个特定时间执行或周期性执行。 TimerTask 是一个实现了 Runnable 接口的抽象类，我们需要去继承这个类来创建欧文自己的定时任务并使用 Timer 去安排它的执行。
```java
Timer timer = new Timer();
timer.schedule(new TimeTask(){
    public void run(){
        System.out.println("abc");
    }
}, 200000 , 1000);
```

## 11.什么是线程池？ 如何创建一个 Java 线程池？
一个线程池管理了一组工作线程，同时它还包括了一个用于防止等待执行的任务的队列。线程池可以避免线程的频繁创建与销毁，降低资源的消耗，提高系统的反应速度。 java.util.concurrent.Excutors 提供了几个 java.util.concurrent.Excutor 接口的实现用于创建线程池，其主要涉及四个角色：
-  线程池： Executor
-  工作线程： Worker 线程， Worker 的 run() 方法执行 Job 的 run() 方法
-  任务 Job： Runnable 和Callable
-  阻塞队列 BlockingQueue
![img5](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc5.png)

1. 线程池 Executor
Excutor 及其实现类是用户级的线程调度器，也是对任务执行机制的抽象，其将任务的提交与任务的执行分离开来，核心实现类包括 ThreadPoolExecutor（用来执行被提交的任务）和 ScheduledThreadPoolExecutor（可以在给定的延迟后执行任务或者周期性执行任务）。Executor 的实现继承链条为：（父接口）Excutor -> （子接口）ExecutorService -> （实现类）[ThreadPoolExecutor + SchrduledThreadPoolExecutor]。

2. 任务 Runnable/Callable
Runnable(run) 和 Callable(call) 都是对任务的抽象，但是 Callable 可以返回任务执行的结果或者抛出异常。

3. 任务执行状态 Future 
Future 是对恩物执行状态和结果的抽象，核心实现类是 furtureTask (所以它既可以作为 Runnable 被线程执行，又可以作为 Futrue 得到 Callable 的返回值)；

![img6](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc6.png)

	3.1. 使用 Callable + Future 获取执行结果
```java
ExecutorService executor = Executors.newCachedThreadPool();
Task task = new Task();
Future<Integer> result = executor.submiut(task);
System.out.println("task运行结果" + resilt.get());

calss Task implements Callable<Integer>{
    @override
    public Integer call() throws Exception{
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i = 0 ; i <100 ; i++)
        	sum += i ;
        return sum;
    }
}
```

	3.2. 使用 Callable + FutureTask 获取执行结果
```java
ExecutorService executor = Executors.newCachedThreadPool();
Task task = new Task();
FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
executor.submit(futureTask);
System.out.println("task运行结果" + futureTask.get());

class Task implements Callable<Integer>{
    @override
    public Integer call() throws Exception{
        System.out.println("子线程在进行运算");
        Thread.sleep(3000);
        int sum = 0 ;
        for(int i = 0 ; i < 100 ; i++){
            sum += i；
        return sum;
        }
    }
}
```

4. 四种常用的线程池

   4.1. FixedThreadPool

   用于创建使用固定线程数的 ThreadPool, corePoolSize = maximumPloolSize  = n（固定的含义），阻塞队列为LinkedBlockingQueue。

   ```java
   public static ExecutorService newFixedThreadPool(int nThreads){
       return new ThreadPoolExecutor(nThreads, nTHreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
   }
   ```



   4.2. SingleThreadExecutor

   用于创建一个单线程的线程池， corePoolSize = maximumPoolSize = 1,阻塞队列为 LinkedBlockingQueue。

   ```java
   public static ExecutorService newSingleThreadExecutor(){
       return new FinalizableDelegatedExecutorService(new ThreadPoolExecutor(1,1,0L,TimeUnit.MILLISECONDS,
                                       new LinkedBlockingQueue<Runnable>()));
   }
   ```



   4.3. CachedThreadPool

   用于创建一个可缓存的线程池， corePoolSize = 0,

   maximumPoolSize = Integer.MAX_VALUE,阻塞队列为SynchronousQueue(没有容量的阻塞队列，每个插入操作必须等待另一个线程对应的移除操作，反之亦然)。

   ```java
   public static ExecutorService newCachedThreadPool() {
           return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                         60L, TimeUnit.SECONDS,
                                         new SynchronousQueue<Runnable>());
       }
   ```



   4.4. ScheduledThreadPoolExecutor

   用于创建一个大小无限的线程池，此线程池支持定时以及周期性执行任务的需求

   ```java
   public ScheduledThreadPoolExecutor(int corePoolSize){
       super(corePoolSize, Integer.MAX_VALUE, 0 , TimeUnit.NANOSECONDS,
            	new DelayedWorkQueue());
   }
   ```



   4.5. 线程池的饱和策略

- Abortpolicy: 直接抛出异常，默认策略；

- CallerRunsPolicy：用调用者所在的线程来执行任务；

- DiscardOldestPolicy: 丢弃阻塞队列中最老的任务，并执行当前任务；

- DiscardPolicy： 直接丢弃任务；

  当然也可以根据应用场景实现 RejectedExecutionHandler 接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。

6. 线程池调优

- 设置最大线程数，防止线程资源耗尽；
- 使用有界队列，从而增加系统的稳定性和预警能力(饱和策略)；
- 根据任务的性质设置线程池大小：CPU 密集型任务（CPU 个数个线程），IO 密集型任务（CPU 个数两倍的线程），混合型任务（拆分）。



##  12. CAS

CAS 自旋 volatile 变量，是一种很经典的用法。

CAS Compare and Swap 即比较并交换，设计并发算法时常用到的一种技术。 CAS 有 3 个操作数，内存值 V ，旧的预期值 A ， 新值 B 。当且仅当预期值 A 和内存值 V 相同时，将内存值修改为 B ，否则什么都不做。CAS 是通过 unsafe 类的 compareAndSwap (JNI, Java Native Interface) 方法实现的，该方法包括四个参数：第一个参数是要修改的对象，第二个参数是对象中要修改变量的偏移量，第三个参数是修改之前的值，第四个参数是预想修改后的值。

CAS 虽然很高效的解决原子操作，但是 CAS 仍然存在三大问题： ABA 问题、 循环时间长开销大和只能保证一个共享变量的原子操作。

- ABA 问题：因为 CAS 需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是 A，变成了 B，又变成了 A，那么使用 CAS 进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA 问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么 A－B－A 就会变成 1A-2B－3A。
- 不适用于竞争激烈的情形中： 并发越高，失败的次数会越多，CAS 如果长时间不成功，会极大的增加 CPU 的开销。因此 CAS 不适合竞争十分频繁的场景。
- 只能保证一个共享变量的原子操作：当对一个共享变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量 i＝2,j=a，合并一下 ij=2a，然后用 CAS 来操作 ij。从 Java1.5 开始 JDK 提供了 AtomicReference 类来保证引用对象之间的原子性，因此可以把多个变量放在一个对象里来进行 CAS 操作。

##  13.AQS：队列同步器

队列同步器（AbstractQueuedSynchronizer）是用来构建锁和其他同步组建的基础框架,技术是 CAS 自旋 Colatile 变量： 它使用了一个 Volatile 成员变量表示同步状态，通过 CAS 修改该变量的值，修改成功的线程表示获取到该锁；若没有修改成功，或者发现状态 state 已经是加锁状态，则通过一个 Waiter 对象封装线程，添加到等待队列中，并刮起等待被唤醒。

同步器是实现锁的关键，子类通过继承同步器并实现它的抽象方法来管理同步状态，利用同步器实现锁的语义。特别地，锁是面向锁使用者的，它定义了使用者与锁交互的接口，隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程排队、等待与唤醒等底层操作。锁和同步器很好地隔离了锁的使用者与锁的实现者所需关注的领域。

一般来说，自定义同步器要么是独占方式，要么是恭喜那个方式，他们也只需实现 tryAcquire-tryRelease、 tryAcquireShared-tryReleaseShared 中的一种即可。但 AQS 也支持自定义同步器同时实现独占和共享两种方式，如 ReentrantReadWriteLock 。

同步器的设计是基于 **模板方法模式** 的，也就是说，使用者需要继承同步器并重写制定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

![img7](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc7.jpg)

AQS 维护了一个 volatile int state （代表共享资源）和一个 FIFO 线程等待队列（多线程争用资源被阻塞时会进入此队列）。 这里 volatile 是核心关键词，具体 volatile 的语义，在此不述 。 state 的访问方式有三种： getState()、 setState() 以及 compareAndSetState()。

AQS 定义了两种资源共享方式： Exclusive（独占，只有一个线程能执行，如 ReentrentLock） 和 Share （共享，多个线程可同时执行，如 Semaphore/CountDownLatch）。不同的自定义同步器在实现时只需要实现共享资源 state 的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS  已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively(): 该线程是否正在独占资源。只有用到 condition 才需要去实现它；

- tryAcquire(int): 独占方式。尝试获取资源，成功则返回 true，失败则返回 false；

- tryRelease(int)：独占方式。尝试释放资源，成功则返回 true ，失败则返回 false；

- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0 表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源；

- tryReleaseShared(int): 共享方式。尝试释放资源，成功则返回 true ，失败则返回 false。

  以 ReentranLock 为例， state 初始化为 0，表示威所定状态。 A 线程 lock() 时，会调用 tryAcquire() 独占该酸并将 state+1.此后，其他线程再 tryAcquire() 时就会失败，直到 A 线程 unlock() 到 state=0 （即释放锁）为止，其他线程才有机会获取该锁。当然，释放锁之前，A 线程自己时可以重复获取此锁的 （state 会累加），这就是可重入的概念。但要注意，获取多少次就要释放多少次，这样才能保证 state 是能回到零态的。

##  14. Java Concurrency API 中的 Lock 接口（Lock interface）是什么？对比同步它有什么优势？

Synchronized 是 Java 的关键字，是 Java 的内置特性，在 JVM 层面实现了对临界资源的同步互斥访问。Synchronized 的语义底层是通过一个 monitor 对象来完成的，线程执行 monitorenter/monitorexit 指令完成锁的获取与释放。而 Lock 是一个 Java 接口（API 如下图所示），是基于 JDK 层面实现的，通过这个接口可以实现同步访问，它提供 了比 synchronized 关键字更灵活、更广泛、粒度更细的锁操作，底层是由 AQS 实现的。二者之间的差异总结如下：

- 实现层面： synchronized （JVM 层面）、Lock（JDK 层面）

- 响应中断： Lock 可以让等待锁的线程响应中断，而使用 synchronized 时，等待的线程会一直等待下去，不能够响应中断；

- 立即返回： 可以让线程尝试获取锁，并在无法获取锁的时候立即返回或者等待一段时间，而 synchronized 却无法办到；

- 读写锁： Lock 可以提高多个线程进行杜操作的效率；

- 可实现公平锁： Lock 可以实现公平锁，而 sychronized 天生就是非公平锁；

- 显式获取和释放： synchronized 在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而 Lock 异常时，如果没有主动通过 unLock() 区释放锁，则很可能造成死锁现象，因此使用 Lock 时需要在 finally 块中释放锁；

  ![img8](https://raw.githubusercontent.com/ABUGADAY/midjavainterview/master/img/juc8.png)




> 引用声明：  
> 　原创作者：书呆子Rico   
> 　作者博客地址：http://blog.csdn.net/justloveyou_/
