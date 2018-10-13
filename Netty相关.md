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

## 3.Netty的特点

- 一个高性能、异步事件驱动的NIO框架，它提供了TCP、UDP和文件传输的支持
- 使用更高效的socket底层 ，对epoll空轮询引起的cpu占用飙升在内部进行进行了处理，避免了直接使用NIO的陷阱，简化了NIO的处理方式
- 采用了多种decoder/encoder支持，对TCP粘包/分包进行自动化处理
- 可使用接受/处理线程池，提高连接效率，对重连、心跳检测的简单支持
- 可配置IO线程数、TCP参数，TCP接受和发送缓冲区使用直接内存代替堆内存，通过内存池的方式循环利用ByteBuf
- 通过引用计数器及时及时申请释放不再引用的对象，降低了GC频率
- 使用单线程串行化的方式，高效的Reactor线程 模型
- 大量使用了volitale、使用了CAS和原子类、线程安全类的使用、读写锁的使用

## 4.Netty的线程模型

- Netty通过Reactor模型基于多路复用器接收并处理用户请求，内部实现了两个线程池，boss线程池和work线程池；其中boss线程池的线程负责处理请求的accept事件，当接受到accpet事件的请求时，把对应的socket封装到一个NioSocketChannel中，并交给work线程池，其中work线程池负责请求的read和write事件，由对应的Handler处理。
- 但线程模型：所有I/O操作都由一个线程完成，即多路复用、事件分发和处理都是由一个Reactor线程上完成的。既要接收客户端的连接请求，向服务器发起连接 ，又要发送/读取请求或应答/响应消息。一个NIO线程同时处理成百上前的链路，性能上无法支撑，速度慢，若线程进入死循环，整个程序不可用，对于高负载、大并发的应用场景不合适。
- 多线程模型：有一个NIO线程(Acceptor)只负责监听服务端，接收客户端的TCP连接请求；NIO线程负责网络IO的操作，即消息的读取、解码、编码和发送；1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程，这是为了防止发生并发操作问题。但在并发百万客户端连接或需要安全认证时，一个Accptor线程可能会存在性能不足问题。
- 主从多线程模型：Acceptor线程用于绑定监听端口，接收客户端连接，将SocketChannel从朱线程池的Reator线程的多路复用器上移除，重新注册到Sub线程池的线程上，用于处理I/O的读写等操作，从而保证mainReacotr只负责介入认证、握手等操作；

## 5.TCP粘包/拆包的原因及解决方法

- TCP是以流的方式来处理数据，一个完整的宝可能会被TCP拆分成多个包进行发送，也可能把小的封装成一个大的数据包发送
- TCP粘包/分包的原因：
  - 应用程序写入的字节大小大于套接字发送缓冲区的大小，会发生拆包现象，而应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上 ，这将会发生粘包现象；
  - 进行MSS大小的TCP分段，当TCP报文长度-TCP头部长度>MSS的时候会发生拆包
  - 以太网帧的payload(净荷)大于MTU(1500字节)进行ip分片
- 解决方法
  - 消息定长：FixedLengthFrameDecoder类
  - 包尾增加特殊字符分割：行分割符LineBasedFrameDecoder 或自定义分隔符类：DelimiterBasedDecoder
  - 将消息分为消息头和消息体：lengthFieldBasedFrameDecoder类。分为有头部的拆包与粘包、字段长度在前且有头部的拆包和粘包、多扩展头部的拆包与粘包

## 6.了解哪几种序列化协议

- 序列化(编码)是将对象序列化为i二进制形式(字节数组)，主要用于网络传输、数据持久化等；而反序列化(解码)则是将从网络、磁盘等读取的字节数组还原成原始对象，以便完成远程调用
- 





