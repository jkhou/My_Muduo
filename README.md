# My_Muduo
**简介**:基于muduo网络库和Ｃ++11实现的网络库。

## 整体架构
- **Event**：事件
- **Reactor**： 反应堆
- **Demultiplex**：多路事件分发器
- **EventHandler**：事件处理器

在muduo中，其**调用关系**大致如下:
- 将事件及其处理方法注册到reactor，reactor中存储了连接套接字connfd以及其感兴趣的事件event
- reactor向其所对应的demultiplex去注册相应的connfd+事件，启动反应堆
- 当demultiplex检测到connfd上有事件发生，就会返回相应事件
- reactor根据事件去调用eventhandler处理程序

![git-command.jpg](https://img-blog.csdnimg.cn/20210214113852361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NoZW5taW5neHVlSVQ=,size_16,color_FFFFFF,t_70)

### 事件分发
- 在事件分发方面，muduo采用类似与nginx负载均衡的思想，**Ｍain Reactor**通过Acceptor处理新的客户端连接，并依照轮询算法将新的连接分派给各个**Sub Reactor**,之后每个Sub Reactor都负责一个连接的IO操作、业务逻辑等处理。

### 事件循环
- 在muduo中，EventLoop就对应于图中的Reactor，而每个Reactor都执行在一个线程上，所以实现了**one loop per thread**的设计;
- 在每个EventLoop中，都包含一个Poller和多个Channel,Poller对应于上图中的Demutiplex，Channel对应于Event并存放在ＣhannelList中；
- EventLoop和Thread等构成了EventLoopTread类，之后的EventLoopTreadPoll用**线程池**的方法存放各个事件循环线程。

![git-command.jpg](https://img-blog.csdn.net/20150731164408996)

- 1、EventLoop是整个模式的核心，它来管理所有事件。one loop per thread说明一个线程只能有一个EventLoop。它封装了eventfd和timerfd，用来唤醒等待在poll上的线程；eventfd是其他线程唤醒当前线程使用，把任务添加到EventLoop的任务队列后，如果不是EventLoop的owner线程，则要唤醒它来执行任务；timerfd用来实现定时。

- 2、Poller是个虚基类，真正调用的时PollPoller或EPollPoller。用来实现IO的复用，事件都注册到Poller中。

- 3、Channel和fd一一对应，虽然它不拥有fd，fd需要响应哪些事件都保存在Channel中，且有对应的回调函数。

- 4、TcpConnection是保存已经建立的连接，它的生命周期模式，因此采用shared_ptr来管理。它负责数据的发送和接收，负责socket fd的关闭。

- 5、Acceptor主要负责连接的建立，用在服务端。当建立连接后，它会把连接的控制权转交给TcpConnection。

- 6、TcpServer是服务端，有Acceptor，用Map保存了当前已经连接的TcpConnection。

- 7、Connector是负责发起连接的一方，用在客户端。发起连接比接收连接要难，非阻塞发起连接更难，要处理各种错误，还要考虑连接失败后如何处理。当连接成功后它会把控制权交给TcpConnection。

- 8、TcpClient是客户端，封装了Connector。

