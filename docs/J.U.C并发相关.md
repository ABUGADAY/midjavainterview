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


> 引用声明：
>　原创作者：书呆子Rico 
>　作者博客地址：http://blog.csdn.net/justloveyou_/