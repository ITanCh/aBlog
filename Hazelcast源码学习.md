# Hazelcast

[Hazelcast](https://hazelcast.org/)是In-Memory Data Grid，一个分布式的集合系统，可以让一个集群里的节点共享如Map一般的集合数据。

## 启动

### 创建节点

一个jvm中可以启动多个Hazelcast节点实例。

创建Hazelcast实例的过程在HazelcastInstanceFactory中完成，流程如下：

1. 加载配置文件
2. 取名字。首先从配置文件获取，没有则给他随机生成一个
3. 创建InstanceFuture对象，该对象保存了真正的hz实例（其实是hz的代理）。该方法提供了set和get方法来访问hz实例。
    1. set，设置hz实例，同时唤醒等待InstanceFuture对象锁的其它线程。
    2. get，获取hz实例，如果有则直接返回。否则会wait线程，直到有人set。
4. 将InstanceFuture对象存入INSTANCE_MAP，一个ConcurrentMap，用来保存jvm里所有的InstanceFuture实例。key是刚才创建的实例名字。
5. 开始创建HazelcastInstanceImpl实例。创建hz实例时会创建如下组件。
    1. LifecycleServiceImpl。用来管理hz实例的生命周周期。使用该service可以关闭hz实例。同时，该service负责管理所有hz生命周期事件的观察者，并负责通知他们。
    2. HazelcastManagedContext。hz实例的容器上下文，像Spring中的一样。可以用来向需要上下文的对象注入hz实例的信息。
    3. userContext，一个Map，用来保存配置文件里的用户上下文。
    4. Node对象，描述hz实例的集群节点属性，重量级的一个对象。
        - 其中最重要的是在指定地址和端口上创建一个ServerSocketChannel。一个Node可能有多个EndpointQualifier（用来描述这个节点的角色，如成员、客户端），每个Qualifier可以创建各自的ServerSocketChannel。
        - 另一个重要对象NodeEngine。它会创建各种服务，类比Spring ApplicationContext。具体如下：
            - MetricsRegistry指标监控服务，用来收集各项指标。
            - ProxyService代理管理。
            - ServiceManager服务管理，所有的Service都会注册在该对象中进行管理，相当于一个service容器。
            - ExecutionService，管理各种线程池的。
            - OperationService，用来管理Operation（类似于runnable）集合，并用线程池来执行这些Operation。Operation是用来在各个分布式节点中进行传递的指令，自己实现了序列化和反序列化的方法。节点间的通信主要通过OperationService来实现，相当于RPC。
            - EventServiceImpl是用来管理事件发布、订阅的中介，包括内部的和远程的，相当于一个分布式消息系统。这些事件会在StripedExecutor线程池中执行，该线程池中的每个worker线程各有一个等待队列，每个worker只从自己队列中获取待执行的任务，该方式可以让特定的几个任务在同一线程中执行，所以叫“条纹”线程池。Event分为同步和异步，同步远程event会将其包装为Operation进行发送。
            - OperationParker。用队列维护等待中的Operation，负责park和unpark Operation，且对阻塞在不同key的Operation用不同的queue管理，且保证同一key下的Operation的串行执行。
            - ClusterWideConfigurationService。负责管理整个集群的配置，配置变更时，会通知其他节点进行更新。
            - TransactionManagerServiceImpl。负责执行事务任务。
            - WanReplicationService，广域网复制服务，向广域网中节点复制值。需要应对高延迟、慢传输。
            - PacketDispatcher，分发Packet到合适的service，如分发operation到OperationService，event到EventService。
            - SplitBrainProtectionServiceImpl，裂脑(两台高可用服务器之间在指定时间内，无法互相检测到对方心跳而各自启动故障转移功能，取得了资源及服务的所有权)保护。
            - SplitBrainMergePolicyProvider，用来管理脑裂恢复后，数据合并的策略。如高命中率优先，最近更新优先等。
        - NetworkingService，管理网络连接，实现了一个NIO框架。它创建了inbound和outbound两组工作线程，两组线程都会阻塞在各自线程上的Selector上，直到有数据出、入时工作线程会被唤醒，并从Selector中获得相应的SelectionKey，然后获取SelectionKey对应的NioPipeline进行处理。NioPipeline具体为NioxxxboundPipeline，它负责管理xxxboundHandler，以处理输入、输出的网络数据。整体设计类似netty。整个网络连接被异步化的过程：  
            - 创建网络连接SocketChannel
            - 在NetworkingService中注册SocketChannel
            - 注册时创建NioChannel，来维护SocketChannel和Pipeline之间的关系
            - 创建NioInboundPipeline、NioOutboundPipeline，它们管理InboundHandler、OutboundHandler责任链
            - 在创建Pipeline时会将该Pipeline哈希到某个工作线程中，并且在该线程中将Pipeline对应的SocketChannel注册到线程负责的Selector上。所以每个工作线程会管理多个网络连接。
            - 写过程：首先调用`NioOutboundPipeline#write`接口，把要写的帧放在该pipline队列中。然后，唤醒对应的selector。Selector上的线程被唤醒，取得等待执行的pipline。在工作线程上Pipline依次执行handler，最后将数据写入socket。
            - 读过程：Socket连接有数据过来时，Selector上的工作线程被唤醒，取得对应的inBoundPipline。在工作线程中InBoundPipline从socket连接中取得字节数据，依次执行handler。
        - ClientEngine，客户端请求处理。
        - DiscoveryService，节点发现服务。它会通过SPI加载定义的各种DiscoveryStrategy来发现节点。默认的是MulticastDiscoveryStrategy：
            - 该策略会首先在一个指定端口上创建广播socket服务
            - 发送：然后启动线程周期向外发送本机IP和端口
            - 接收：从广播socket服务读取接收到的节点信息
        - ClusterService，集群管理服务，负责成员节点查询、加入和离开。

    5. 


用一个静态ConcurrentMap来保存jvm中的所有节点实例。每个实例有独一无二的名字，该名字为map中的key。


## 小技巧

1. 用AtomicReference来引用一个对象，可以用compareAndSet方法来判断该对象被新对象替换前，是否已经被其它线程替换。

