# Netty相关

## 1. BIO、NIO、和AIO的区别

- BIO：一个连接一个线程，客户端有连接请求时服务器就需要启动一个线程进行处理。线程开销大。
- 伪异步IO：将请求连接放入线程池，一对多，但线程还是很宝贵的资源。
- NIO：一个请求一个线程，但客户端发送的连接请求都会注册到多路复用路由器上，多路复用路由器轮询到有I/O请求时才启动一个线程进行处理。
- AIO：一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理。
- BIO是面向流的，NIO是面向缓冲区的；BIO是面向各种阻塞的，而NIO是非阻塞的；BIO的Stream是单向的，而NIO的channel是双向的。
- NIO的特点：事件驱动型、单线程处理多任务、非阻塞I/O，I/O读写不再阻塞，而是返回0、基于block的传输比基于流的传输更高效、更高级的IO函数zero-copy、IO多路复用大大提高 了Java网络应用的可伸缩性和实用性。基于Reactor线程模型。
- 在Reacotr模式中，事件分发器等待某个事件或者可应用某个操作的状态发生，时间分发器就把这个事件传给事先注册的事件处理函数或者回调函数，由后者来作实际的读写操作。如在Reactor中实现度：注册 读就绪事件和相应的事件处理器、事件分发器等待事件；事件到来，激活分发器，分发器调用事件对应的处理器、事件处理器完成实际的读操作，处理读到的数据，注册新的事件，然后返还控制权。



## 2.NIO的组成

- Buffer： 与Channel进行交互，数据是从Channel读入缓冲区，从缓冲区写入Channel中的
  - flip方法：反转此缓冲区，将position给limit，然后将position置为0，其实就是切换读写模式
  - clear方法：清楚此缓冲区，将position置为0，把capacity的值给limit
  - rewind方法：重绕此缓冲区，将position置为0
- DirectByteBuffer可减少一次系统空间到用户空间的拷贝。但Buffer创建和销毁的成本更高，不可空，通常会用内存池来提高性能。直接缓冲区主要分配给那些易受基础系统的本机I/O操作影响的大型、持久的缓冲区。如果数据量比较小的中小应用情况下，可以考虑使用heapBuffer，由JVM进行管理。
- Channel：表示IO源与目标打开的连接，是双向的，但不能直接访问数据，只能与Buffer进行交互。通过源码可知，FileChannel的read方法和write方法都导致数据复制了两次！
- Selector可使一个单独的线程管理多个Channel，open方法向多路复用器注册通道，可以监听的事件类型：读、写、连接、accept。注册事件后会产生一个SelectionKey：它表示SelectableChannel和Selector之间的注册关系，wakeup方法：使尚未返回的第一个选择操作立即返回，唤醒的原因是：注册了新的channel或者事件；channel关闭，取消注册；优先级更高的事件触发（如定时器事件），希望及时处理。
- Selector在Linux的实现类是EPollSelectorImpl，委托给EpollArrayWrapper实现，其中三个native方法是对epoll的封装，而EPollSelectorImpl.implregister方法，通过调用epoll_ctl向epoll实例中注册事件，还将注册的文件描述符（fd）与SeletionKey的对应关系添加到fdToKey中，这个map维护了文件描述符与SeletionKey的映射。
- fdToKey有时会变得非常大，因为注册到Selector上的Channel非常多（百万连接）；过期或实效的Channel没有及时关闭。fdToKey总是串行读取的，而读取是在Select方法中进行的，该方法非线程安全的。
- Pipe：两个线程之间的单项数据连接，数据会被写到sink通道，从source通道读取
- NIO的服务端建立过程：Selector.open():打开一个Selector；ServerSocketChannel.open()：创建服务器的Channel;bind()：绑定到某个端口上，并配备非阻塞模式；register():注册Channel和关注的事件到Selector上；select()轮询拿到已经就绪的事件



