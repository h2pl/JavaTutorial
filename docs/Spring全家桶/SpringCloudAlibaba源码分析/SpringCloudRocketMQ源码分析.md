### 一、NameServer启动

源码入口：NamesrvStartup#main

##### 1.NamesrvController controller = createNamesrvController(args);

*   检测命令行参数
*   创建核心配置对象，NamesrvConfig、NettyServerConfig
*   解析 -c 、-p参数
*   检查RocketMQ_HOME环境变量
*   final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);创建controller
*   controller.getConfiguration().registerConfig(properties); 注册所有配置信息

##### 2.start(controller);

*   controller.initialize()； 执行初始化
    ○ this.kvConfigManager.load(); 加载KV配置
    ○ this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);创建NettyServer网络处理对象
    ○ this.remotingExecutor =Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_")); 创建Netty服务器工作线程池
    ○ this.registerProcessor(); 注册NameServer的Processor 注册到RemotingServer中
    ○ NamesrvController.this.routeInfoManager.scanNotActiveBroker() 启动定时任务，移除不活跃的Broker
    ○ NamesrvController.this.kvConfigManager.printAllPeriodically() 定时打印KV配置信息
*   Runtime.getRuntime().addShutdownHook 注册关闭钩子，在关闭服务时释放资源
*   controller.start()； 启动controller

NameServer的作用主要有两个：
1.维护broker的服务地址信息，并进行更新
2.给Producer、consumer提供Broker的服务列表





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/3245844-46ca880d83fb583b.png)



image.png



### 二、Broker启动

源码入口：Brokerstartup#main

##### 1.createBrokerController(args)

*   构建四个核心配置对象：BrokerConfig、NettyServerConfig、NettyClientConfig、MessageStoreConfig
*   BrokerConfig只解析 -c参数
*   RocketMq_HOME环境变量检查
*   RemotingUtil.string2SocketAddress(addr) 将namesrvAddr地址进行拆分
*   messageStoreConfig.getBrokerRole() 通过BrokerId判断主从：masterId=0，Deldger集群的所有Broker节点ID都是-1
*   解析 -p、-m参数，并将解析的参数添加到四个核心配置对象中
*   BrokerController controller = new BrokerController 创建brokerController，将四个核心配置类传入
*   controller.getConfiguration().registerConfig(properties); 重新注册（更新）配置
*   controller.initialize(); 初始化controller
    ○ 加载磁盘上的配置文件：topicConfigManager、consumerOffsetManager、subscriptionGroupManager、consumerFilterManager
    ○ this.messageStore =new DefaultMessageStore() 构建消息存储组件
    ○ this.messageStore.load() 加载磁盘文件
    ○ this.remotingServer = new NettyRemotingServer 构建Netty网络组件
    ○ this.fastRemotingServer = new NettyRemotingServer 这个fastRemotingServer与RemotingServer功能基本差不多，处理VIP端口请求
    ○ 后面就是初始化一些线程池
    ○ this.registerProcessor(); broker注册一些Processor处理方法
*   Runtime.getRuntime().addShutdownHook 注册关闭钩子

##### 2.start(BrokerController controller)

*   this.messageStore.start(); 这里启动服务主要是为了将CommitLog的写入事件分发给ComsumeQueue和IndexFile
*   启动两个Netty服务：remotingServer、fastRemotingServer
*   this.fileWatchService.start(); 文件监听服务
*   this.brokerOuterAPI.start(); brokerOuterAPI可以理解为一个Netty客户端，往外发请求的组件，例如发送心跳
*   this.pullRequestHoldService.start(); 长轮询请求暂停服务
*   this.filterServerManager.start(); 使用filter进行过滤
*   BrokerController.this.registerBrokerAll() Broker核心的心跳注册任务,主要作用就是将broker注册到Namesrv中

broker的核心作用：
**1.作为client时，向nameServer发送心跳信息、发起事务的状态检查**
**2.作为服务端时，用于存储消息、响应consumer端的请求**

### 三、Netty服务注册框架

### 四、Broker心跳注册过程

源码入口：BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister())



```
public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway, boolean forceRegister) {
    TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();

    if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission())
        || !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {
        ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>();
        for (TopicConfig topicConfig : topicConfigWrapper.getTopicConfigTable().values()) {
            TopicConfig tmp =
                new TopicConfig(topicConfig.getTopicName(), topicConfig.getReadQueueNums(), topicConfig.getWriteQueueNums(),
                                this.brokerConfig.getBrokerPermission());
            topicConfigTable.put(topicConfig.getTopicName(), tmp);
        }
        topicConfigWrapper.setTopicConfigTable(topicConfigTable);
    }
    //这里才是比较关键的地方。先判断是否需要注册，然后调用doRegisterBrokerAll方法真正去注册。
    if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
                                      this.getBrokerAddr(),
                                      this.brokerConfig.getBrokerName(),
                                      this.brokerConfig.getBrokerId(),
                                      this.brokerConfig.getRegisterBrokerTimeoutMills())) {
        doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
    }
}

```





```
// Broker注册最核心的部分
private void doRegisterBrokerAll(boolean checkOrderConfig, boolean oneway,
                                 TopicConfigSerializeWrapper topicConfigWrapper) {
    // 注册broker方法
    List<RegisterBrokerResult> registerBrokerResultList = this.brokerOuterAPI.registerBrokerAll(
        this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId(),
        this.getHAServerAddr(),
        topicConfigWrapper,
        this.filterServerManager.buildNewFilterServerList(),
        oneway,
        this.brokerConfig.getRegisterBrokerTimeoutMills(),
        this.brokerConfig.isCompressedRegister());

    if (registerBrokerResultList.size() > 0) {
        RegisterBrokerResult registerBrokerResult = registerBrokerResultList.get(0);
        if (registerBrokerResult != null) {
            //注册完保存主从节点的地址
            if (this.updateMasterHAServerAddrPeriodically && registerBrokerResult.getHaServerAddr() != null) {
                this.messageStore.updateHaMasterAddress(registerBrokerResult.getHaServerAddr());
            }

            this.slaveSynchronize.setMasterAddr(registerBrokerResult.getMasterAddr());

            if (checkOrderConfig) {
                this.getTopicConfigManager().updateOrderTopicConfig(registerBrokerResult.getKvTable());
            }
        }
    }
}

```





```
public List<RegisterBrokerResult> registerBrokerAll(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final boolean oneway,
    final int timeoutMills,
    final boolean compressed) {
    //使用CopyOnWriteArrayList提升并发安全性
    final List<RegisterBrokerResult> registerBrokerResultList = new CopyOnWriteArrayList<>();
    // 获取所有nameServer的地址信息
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {

        final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
        requestHeader.setBrokerAddr(brokerAddr);
        requestHeader.setBrokerId(brokerId);
        requestHeader.setBrokerName(brokerName);
        requestHeader.setClusterName(clusterName);
        requestHeader.setHaServerAddr(haServerAddr);
        requestHeader.setCompressed(compressed);

        RegisterBrokerBody requestBody = new RegisterBrokerBody();
        requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
        requestBody.setFilterServerList(filterServerList);
        final byte[] body = requestBody.encode(compressed);
        final int bodyCrc32 = UtilAll.crc32(body);
        requestHeader.setBodyCrc32(bodyCrc32);
        //通过CountDownLatch，保证在所有NameServer上完成注册后再一起结束。
        final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
        for (final String namesrvAddr : nameServerAddressList) {
            brokerOuterExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        RegisterBrokerResult result = registerBroker(namesrvAddr, oneway, timeoutMills, requestHeader, body);
                        if (result != null) {
                            registerBrokerResultList.add(result);
                        }

                        log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                    } catch (Exception e) {
                        log.warn("registerBroker Exception, {}", namesrvAddr, e);
                    } finally {
                        countDownLatch.countDown();
                    }
                }
            });
        }

        try {
            countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
        }
    }

    return registerBrokerResultList;
}

```



NameServer处理请求：



```
//NameServer处理请求的核心代码
@Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                          RemotingCommand request) throws RemotingCommandException {

    if (ctx != null) {
        log.debug("receive request, {} {} {}",
                  request.getCode(),
                  RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                  request);
    }

    switch (request.getCode()) {
        case RequestCode.PUT_KV_CONFIG:
            return this.putKVConfig(ctx, request);
        case RequestCode.GET_KV_CONFIG:
            return this.getKVConfig(ctx, request);
        case RequestCode.DELETE_KV_CONFIG:
            return this.deleteKVConfig(ctx, request);
        case RequestCode.QUERY_DATA_VERSION:
            return queryBrokerTopicConfig(ctx, request);
        case RequestCode.REGISTER_BROKER: //Broker注册请求处理。版本默认是当前框架版本
            Version brokerVersion = MQVersion.value2Version(request.getVersion());
            if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                return this.registerBrokerWithFilterServer(ctx, request); //当前版本
            } else {
                return this.registerBroker(ctx, request);
            }
        case RequestCode.UNREGISTER_BROKER:
            return this.unregisterBroker(ctx, request);
        case RequestCode.GET_ROUTEINFO_BY_TOPIC:
            return this.getRouteInfoByTopic(ctx, request);
        case RequestCode.GET_BROKER_CLUSTER_INFO:
            return this.getBrokerClusterInfo(ctx, request);
        case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
            return this.wipeWritePermOfBroker(ctx, request);
        case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
            return getAllTopicListFromNameserver(ctx, request);
        case RequestCode.DELETE_TOPIC_IN_NAMESRV:
            return deleteTopicInNamesrv(ctx, request);
        case RequestCode.GET_KVLIST_BY_NAMESPACE:
            return this.getKVListByNamespace(ctx, request);
        case RequestCode.GET_TOPICS_BY_CLUSTER:
            return this.getTopicsByCluster(ctx, request);
        case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
            return this.getSystemTopicListFromNs(ctx, request);
        case RequestCode.GET_UNIT_TOPIC_LIST:
            return this.getUnitTopicList(ctx, request);
        case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
            return this.getHasUnitSubTopicList(ctx, request);
        case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
            return this.getHasUnitSubUnUnitTopicList(ctx, request);
        case RequestCode.UPDATE_NAMESRV_CONFIG:
            return this.updateConfig(ctx, request);
        case RequestCode.GET_NAMESRV_CONFIG:
            return this.getConfig(ctx, request);
        default:
            break;
    }
    return null;
}

```



实际就是将broker信息注册到routeInfo中：



```
public RemotingCommand registerBrokerWithFilterServer(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(RegisterBrokerResponseHeader.class);
    final RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.readCustomHeader();
    final RegisterBrokerRequestHeader requestHeader =
        (RegisterBrokerRequestHeader) request.decodeCommandCustomHeader(RegisterBrokerRequestHeader.class);

    if (!checksum(ctx, request, requestHeader)) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("crc32 not match");
        return response;
    }

    RegisterBrokerBody registerBrokerBody = new RegisterBrokerBody();

    if (request.getBody() != null) {
        try {
            registerBrokerBody = RegisterBrokerBody.decode(request.getBody(), requestHeader.isCompressed());
        } catch (Exception e) {
            throw new RemotingCommandException("Failed to decode RegisterBrokerBody", e);
        }
    } else {
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setCounter(new AtomicLong(0));
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setTimestamp(0);
    }
    //routeInfoManager就是管理路由信息的核心组件。
    RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
        requestHeader.getClusterName(),
        requestHeader.getBrokerAddr(),
        requestHeader.getBrokerName(),
        requestHeader.getBrokerId(),
        requestHeader.getHaServerAddr(),
        registerBrokerBody.getTopicConfigSerializeWrapper(),
        registerBrokerBody.getFilterServerList(),
        ctx.channel());

    responseHeader.setHaServerAddr(result.getHaServerAddr());
    responseHeader.setMasterAddr(result.getMasterAddr());

    byte[] jsonValue = this.namesrvController.getKvConfigManager().getKVListByNamespace(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG);
    response.setBody(jsonValue);

    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
}

```



### 五、Producer发送消息

源码入口：DefaultMQProducer#start
1.this.defaultMQProducerImpl.start(); 生产端启动



```
public void start(final boolean startFactory) throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            // 默认就是CREATE_JUST
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig();
            //修改当前的instanceName为当前进程ID
            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }
            //客户端核心的MQ客户端工厂 对于事务消息发送者，在这里面会完成事务消息的发送者的服务注册
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
            //注册MQ客户端工厂示例
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                                            + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                                            null);
            }

            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
            //启动示例 --所有客户端组件都交由mQClientFactory启动
            if (startFactory) {
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                     this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                                        + this.serviceState
                                        + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                                        null);
        default:
            break;
    }
    // 向所有的broker发送心跳
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

    this.startScheduledTask();

}

```



### 六、Consumer消费消息

消费端入口：DefaultMQPushConsumer#start
this.defaultMQPushConsumerImpl.start();



```
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                     this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig();

            this.copySubscription();

            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }
            //客户端示例工厂，生产者也是交由这个工厂启动的。
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
            //负载均衡策略
            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

            this.pullAPIWrapper = new PullAPIWrapper(
                mQClientFactory,
                this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                //从这里可以看出，广播模式与集群模式的最本质区别就是offset存储的地方不一样。
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                        //广播模式是在消费者本地存储offset
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                        //集群模式是在Broker远端存储offset
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            this.offsetStore.load();
            //顺序消费监听创建ConsumeMessageOrderlyService
            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService =
                    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
                //并发消费监听创建ConsumeMessageConcurrentlyService
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
               this.consumeOrderly = false;
               this.consumeMessageService =
               new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }

           this.consumeMessageService.start();
           //注册消费者。与生产者类似，客户端只要按要求注册即可，后续会随mQClientFactory一起启动。
           boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
           if (!registerOK) {
               this.serviceState = ServiceState.CREATE_JUST;
               this.consumeMessageService.shutdown(defaultMQPushConsumer.getAwaitTerminationMillisWhenShutdown());
               throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
               + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
               null);
           }

           mQClientFactory.start();
           log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
           this.serviceState = ServiceState.RUNNING;
           break;
           case RUNNING:
           case START_FAILED:
           case SHUTDOWN_ALREADY:
            throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),null);
           default:
                break;
           }

               this.updateTopicSubscribeInfoWhenSubscriptionChanged();
               this.mQClientFactory.checkClientInBroker();
               this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
               this.mQClientFactory.rebalanceImmediately();
           }

```



**1、consumer端的消费模式：**
● 集群模式：集群模式下每个consumer都会分配不同的消息
● 广播模式：广播模式下每个消息都推送给所有consumer
**2、关于offset存储：**
● 广播模式：this.offsetStore = new LocalFileOffsetStore(); 存储在每个consumer中
● 集群模式：this.offsetStore = new RemoteBrokerOffsetStore(); 存储在broker端



作者：枫叶红花
链接：https://www.jianshu.com/p/8dd4cfeae39d
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning