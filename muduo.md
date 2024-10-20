# linux下muduo网络库的简要概括

总的理念就是one loop per thread，让每一个线程管理自己被分配的连接的相关事务，属于其他线程的事务就追加到其他线程代办事务，每一个线程往复执行epoll返回的事件和代办事务。这样就避免每次使用资源前都要加锁，只在向其他线程添加连接，追加事务时加锁，大大提高并发效率。

[我的现代c++重写](https://github.com/Gidi233/Gd_rpc/tree/%E5%9F%BA%E7%A1%80%E7%BD%91%E7%BB%9C%E5%BA%93)

以下是各部分解析：

## 日志部分

多线程下多生产者单消费者模型。

### 生产者

用宏以c++流风格来构造整个日志消息对象，析构时将消息传递给后台一个锁实现的线程安全的缓存。

### 消费者

通过 `std::vector<std::unique_ptr<Buffer>>;`动态管理空间，因为日志要有时序性，多线程难以高效写入同一文件，只单开一个线程来将所有缓存写入文件。

## 网络部分

### eventloop

管理epoll，在当前EventLoop事件循环处理channel的回调事件，和其他线程添加的回调操作 。

对于线程数 >=2 的情况 IO线程     mainloop(mainReactor) 主要工作： accept接收连接将返回的connfd回调到TcpServer::newConnection，通过轮询将TcpConnection对象分配给subloop处理。

**有意思的技巧：**

1.one loop per thread

一个最简单的多线程服务器，mainloop负责接受请求连接，将回调通过生产者消费者模型实现的线程安全的队列写入subloop中，但是通过锁和条件变量竞争获取事件，相当于将所有的线程都“阻塞”在了队列，逐个去取任务，处理时也要对对象上锁。

而muduo通过EventLoop都绑定了一个线程，充份利用了多核CPU的能力，每一个核的线程负责循环监听一组文件描述符的集合，运行相关连接的一切事务。涉及其他loop管理的连接就通过注册追加到其他loop的待办事件避免了竞态，然后通过wakeup()机制使用eventfd创建的wakeupFd_ 通知其他loop（线程）。

```
int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
```

关键就是处理事务时用线程id与loop绑定线程id比较，将事件限制到对应loop，来避免了上锁。

```
void EventLoop::runInLoop(Functor cb) {
  if (isInLoopThread())  // 在当前EventLoop
  {
    cb();
  } else  // 非当前EventLoop
  {
    queueInLoop(cb);
  }
}
```

2.减少竞态。以交换的方式减少了锁的临界区范围 提升效率 同时避免了死锁 如果执行functor()在临界区内 且functor()中调用queueInLoop()就会产生死锁。


```cpp 
  {

    std::unique_lock<std::mutex> lock(mutex_);

    functors.swap(pending_functors_);  

  }
//逐个执行functor
```

---

### EPollpoller

封装了epoll,管理channel

### channel

封装了建立了连接的fd和其感兴趣的event。tcpconnect、Acceptor给channel设置相应的回调函数 poller给channel通知感兴趣的事件发生了   channel会回调相应的回调函数

**有意思的技巧：**

1.其中TcpConnection中注册了Chnanel对应的回调函数，所以Channel不能在TcpConnection对象生命周期终止后响应。

为了满足“如果对象还活着，就调用其成员函数，否则忽略之”的语意，在channel的tie中保留对TcpConnection的weak_ptr引用，在每次回调时先判断TcpConnection对象是否还在。

```
void Channel::tie(const std::shared_ptr<void>& obj) {
  tie_ = obj;
  tied_ = true;
}

void Channel::handleEvent(util::Timestamp receiveTime) {
  if (tied_) {
    // 如果活着就调用
    std::shared_ptr<void> guard = tie_.lock();
    if (guard) {
      handleEventWithGuard(receiveTime);
    }
    // 否则忽略，说明Channel的TcpConnection对象已经不存在了
  } else {
    handleEventWithGuard(receiveTime);
  }
}
```

---

### eventLoop_thread

创建线程启动eventloop。

### eventLoop_threadPool

线程池启动线程，结束时没有额外操作，都用unique_ptr<eventLoop_thread>管理了，在eventLoop_thread里做好了线程回收。

---

### Acceptor

封装socket一系列系统调用，运行在baseloop，把accept注册给channel的可读事件。

运行过程： TcpServer::start() 构建Acceptor并listen() 如果有新用户连接执行回调(accept => connfd => 构建tcpconnect，将绑定、监听channel事件传递给 subloop并唤醒)

---

### tcpconnection

管理channel，封装读取输入行为、将注册在tcpserver的用户回调复制到自己的handle，再注册给channel。

**有意思的技巧**

1.通过enable_shared_from_this延长生命周期，因为需要把tcpconnection的成员函数加入eventloop的待办事件、注册为channel的回调。所以不能只有tcpserver拥有tcpconnection的share_ptr，不然一旦发生错误，服务器主动断开连接，往事件循环中追加完摧毁连接事件后，整个tcpconnection资源就被释放了。eventloop执行pending事件，以及此时临界状态下还没取消监听的fd发生了可读事件执行的回调，都会直接段错误。所以必须用share_ptr管理传入的this参数。

```
class TcpConnection :  public std::enable_shared_from_this<TcpConnection>
类中使用：connectionCallback_(shared_from_this());
```

2.用atomic管理state状态变化，  因为可能别的连接（另一个线程）在交互时发现该连接（或拥有该连接的上层结构）状态异常，通过shutdown改变状态，发生竞态，需要atomic（CAS）原子管理状态。

---

发送数据 应用写的快 而内核发送数据慢，所以当一次不能把所有数据流发完时，就需要把未发送部分写入缓冲区，然后给channel注册EPOLLOUT事件，Poller发现tcp的发送缓冲区有空间后会调用channel对应注册的writeCallback回调方法，就是TcpConnection设置的handleWrite回调， 把发送缓冲区outputBuffer_的内容全部发送完成


#### buffer

可动态扩展的缓冲区，因为网络可能随时中断，于用户态上的协议可能只读取了分包的部分，所以接口分为拿数据和更新下标。

**有意思的技巧**

`ssize_t readv(int fd, const struct iovec *iov, int iovcnt);`

因为Buffer缓冲区是有大小的， 但是从fd上读取数据的时候 却不知道tcp数据的最终大小，用readv只用一次系统调用拿到所有数据，避免在循环中read，多次切换用户态内核态的消耗。


```
  // 栈额外空间，用于从套接字往出读时，当buffer_暂时不够用时暂存数据，待buffer_重新分配足够空间后，在把数据交换给buffer_。
  char extrabuf[65536] = {0};

  // 使用iovec分配两个连续的缓冲区
  struct iovec vec[2];
  const size_t writable = writableBytes();

  // 第一块缓冲区，指向可写空间
  vec[0].iov_base = begin() + writerIndex_;
  vec[0].iov_len = writable;
  // 第二块缓冲区，指向栈空间
  vec[1].iov_base = extrabuf;
  vec[1].iov_len = sizeof(extrabuf);
  // 如果第一个缓冲区能接收所有数据，那就只会用一个缓冲区 而不使用栈空间extrabuf[65536]的内容
```

---

### tcpserver

管理TcpConnection，注册用户自定义回调

建立连接：TcpServer => Acceptor => 有一个新用户连接，通过accept函数拿到connfd=> TcpConnection设置回调 => 设置到Channel =>发生读事件 Poller响应 => Channel回调    

主动更改、删除监听事件：tcpserver=>TcpConnection=>channel记录自己所在循环=>eventloop=>poller

接收到epollhub：channel::handle(closeCallback\_) => tcpconnection::handleclose(closecallback_) => tcpserver::removeConnection将事件TcpConnection::connectDestroyed加到其所在loop	connectDestroyed => channel => eventloop => poller移出记录的channel

---

### TcpClient中的connector

异步的方式封装了socket的connet相关过程

过程：根据connect返回的错误码选择重试，或将fd的读、异常事件注册到epoll，连接处理完成后，触发写事件处理将连接传递给Tcpclient（这异步就算连接走本机回环地址都有用，我写测试文件时，最开始没检测连接是否成功，发起连接后直接发，总是少发几百条）
