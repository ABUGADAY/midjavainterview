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
![img1](https://github.com/ABUGADAY/midjavainterview/tree/master/img/juc1.jpg)

所以，得出一个结论就是：只要这个线程对象被 Java GC 回收，就不会出现内存泄露。但是如果只把 ThreadLocal 引用指向 null 而线程对象依然存在，那么此时 Value 是不会被回收的，这就发生了我们认为的内存泄露。比如，在使用线程池的时候，线程结束是不会销毁的而是会再次使用的，这种情形下就可能出现 ThreadLocal 内存泄露。
Java 为了最小化减少内存泄露的可能性和影响，在 ThreadLocal 进行 get、set 操作时会清除线程 Map 里所有 key 为 null 的 value。所以最怕的情况就是，ThreadLocal 对象设 null 了，开始发生 “内存泄露”，然后使用线程池，线程结束后被放回线程池中而不销毁，那么如果这个线程一直不被使用或者分配使用了又不再调用 get/set 方法，那么这个期间就会发生真正的内存泄露。因此，最好的做法是：在不使用该 ThreadLocal 对象时，及时调用该对象的 remove 方法去移除 ThreadLocal.ThreadLocalMap 中的对应 Entry。

> 引用声明：
>　原创作者：书呆子Rico 
>　作者博客地址：http://blog.csdn.net/justloveyou_/
