# RPC设计

往简单了说就是，制定一个协议，将内部参数（方法服务名、msgid等等）与用户交互信息 序列化后发送，接收是反过来的相同逻辑。

[前置网络库](https://blog.csdn.net/qq_74050480/article/details/137730499)

以下是我通过实现PB的service相关接口的实现。

## 协议类ProtocolCodec

包括了协议头：Header长（4字节）->HeaderInfo(包括msg_id_、version_、service_name_、method_name_、message_type_、error_code_、extendInfo_len_、payload_len_)->extendInfo(用于可拓展，长度为extendInfo_len)->payload（实际的用户数据 长度payload_len）。



message_type包括 Request（id++）、Reponse（id++）、Cancel、Heartbeat。（待实现RequestStream,ReponseStream、Metric）



完成了消息发送前的组装处理和接收消息后的反序列化解析（包括错误处理），具体说是协议头，message的序列化直接调用一个PB的函数就完成了。

用户的发送前的序列化很直接，接收后执行用户注册的Closure 回调进行异步的业务处理。

服务端触发读操作后执行此处的回调，反序列化后交给`BaseRpcChannel`的回调处理(注册了所注册服务执行完后的回调，回到这里发送)。

最开始想弄个基类支持PB、Json两种序列化方式，把通过元编程注册自动完成c++类型与json类型都看了，但需要实现protobuf::message类型，有点复杂 放弃了。



## rpcChannelCli

注册连接成功的回调

维护随调用自增的msg_id,map<id,pair<response,closure>>等信息创建时把onRpcHeader回调注册给`ProtocolCodec`，在其反序列化后根据根据消息去找，用户注册的业务处理回调执行。

实现了google::protobuf::RpcChannel的CallMethod接口，将回调保存等到收到回复后调用。

用该channel创建stub_来进行用户的调用。



## RpcServer

创建tcpserver，存储注册的service。

注册连接成功的回调，设置读事件的回调（到ProtocolCodec），把BaseRpcChannel的生命周期交给TcpConnection管理。

维护map<const net::TcpConnectionPtr&,pair<RpcController,closure(留着置空，或返回取消成功)>>）

创建时把onRpcHeader回调注册给`ProtocolCodec`，在其反序列化后根据协议做内部处理。

用户使用时先RegisterService(service)注册所有的service，再run。

收到消息后根据协议头的信息找到具体执行的method，调用service->CallMethod执行具体注册的函数，同时注册用于序列化发送的回调。

## RpcController

用于在用户与框架间的信息交互，包括传递跟踪 RPC 调用的状态，处理错误、超时、重试和取消（主动待做：往RpcController里放一个channel(eventfd)，与msgid绑定，回调为向服务端发送cancel msg，用户主动取消就写入）。



## 时间轮

利用时间轮高效完成调⽤端的实现高效的心跳、超时与重试机制，用Task封装定时任务，用rotations（轮转几次后触发）来支持无限时长的定时。



## 被动取消（超时）

客户端设置超时事件，回调发送cancel。

服务端接收后，把任务加到别的eventloop，需要用户在耗时部分检查RpcController状态（类似go的select ctx.done()），接收取消后更改RpcController状态，更改closure回调（原序列化发送逻辑）。设置错误码，返回给客户端。

客户端在取消后 回调删除原先channel保存的业务处理closure。

或在协议里添加超时时间，服务端设置超时回调。




# 集成ZooKeeper
## 客户端负载均衡

维护从注册中心获取地址的所有连接

每次调用都选择不同的连接

[实现](https://aibenlin.com/algorithm/2019/07/22/algorithm-weight.html)

该算法除了有权重weight，还引入了另一个变量current_weight，在每一次遍历中会把current_weight加上weight的值，并选择current_weight最大的元素，对于被选择的元素，再把current_weight减去所有权重之和。



## 心跳超时断连

在服务端的连接回调新添定时事件（60s），在读事件回调中刷新该Task，触发就断连。

客户端设置无限重复的定时事件（10s）,触发就发一个Heartbeat类型的Message，任何接收都会重置定时事件。

客户端在多次心跳未被响应后，把该连接放入异常连接（不会负载均衡到该连接），等收到全部心跳后，在加回正常连接

若被挂断后，定时指数回退进行重连。（zookeeper没有更新的情况）



## 异常重试

发起重试、负载均衡选择节点的时候，去掉重试之前出现过问题的那个节点

任意连接收到正确回复后，其他连接发送cancel msg，移除回调

业务逻辑幂等，或在业务上做到强一致性，判断该操作id是否已进行

## 思考

连接到ZooKeeper 的节点数量特别多，对 ZooKeeper 读写特别频繁

 ZooKeeper 的⼀⼤特点就是强⼀致性

RPC 框架依赖的注册中⼼的服务数据的⼀致性其实并不需要满⾜ CP，只要满⾜ AP 即可
# 扩展

## 安全

rpc用于内部调用，保证让服务调⽤⽅能拿到真实的服务提供⽅ IP 地址集合，且服务提供⽅可以管控调⽤⾃⼰的应⽤就够了



## 限流

服务器在读回调前，检测当前压力 直接返回限流异常，客户端重试，把当前连接权重降低，或直接熔断。

//平滑限流的滑动窗⼝、漏⽃算法以及令牌桶算法等等。其中令牌桶算法最为常⽤

注册、发现时加⼀个分组参数，⾮核⼼应⽤不要跟核⼼应⽤分在同⼀个组，核⼼应⽤之间应该做好隔离，实现流量隔离？？



## 优雅关闭

当服务提供⽅正在关闭，如果这之后还收到了新的业务请求，服务提供⽅直接返回⼀个特定的异常给调⽤⽅

在开始关闭时改变所有回调为发送类型为Shutdown的msg ，主动向所有的连接发送Shutdown的msg，等待客户端主动断开连接，再关闭服务器？



## 启动预热

客户端定时任务逐步增加新连接的权重值至默认值

调用方 拉取 路由策略（特定IP、负载中参数） 来灰度发布 

# GRPC

创建server时，会创建ServerCompletionQueue（无锁队列，用于异步操作），交由server（SyncRequestThreadManager），用于管理连接、读写事件，mainloop中执行PollForWork（事件被触发后执行closure）、DoWork(执行RPC)，没看到这俩是怎么交互的。

创建ServerCompletionQueue时，会根据环境确认运行时实际会调用的函数，赋给vtable等，其中声明了很多内部的函数接口（不懂为啥不用纯虚函数），工厂模式。

service构造时会addmethod，Server调用RegisterService时，会把这些method放到sync_req_mgrs_里的每一个SyncRequestThreadManager

serverd的AddListeningPort会调用Chttp2ServerAddPort创建tcp_server，封装在了grpc_tcp_listener交给了grpc_tcp_server

在`grpc_server_start(server_);`中把cqs中的grpc_pollset指针放到grpc_server的指针数组pollsets中，建立epoll，创建了socket进行bind、listen，

```cpp
  for (grpc_completion_queue* cq : cqs_) {
    if (grpc_cq_can_listen(cq)) {
      pollsets_.push_back(grpc_cq_pollset(cq));
    }
  }
```

后把epoll交由tcp_server管理，设置读回调，在accept新连接后，回调设置http2



combiner_data是个类似一轮循环中的待做事件，grpc_core::ExecCtx::Get()->Flush()后执行，没理解closure_list中是谁的闭包（listenfd有可读事件后，把上面这些事给default-excutor处理？），感觉线程、epoll模型也还就是muduo那一套。

感觉是没看到什么核心的东西就放弃了，或许可以去摸个线程的负载均衡策略。