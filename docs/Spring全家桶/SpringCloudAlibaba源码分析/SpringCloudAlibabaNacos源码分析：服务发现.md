# 一、Nacos服务发现流程图

![在这里插入图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b5b4ef0330dc4882b0fc2f73994face7.png "在这里插入图片描述")

建议大家自己梳理一下流程，也可以参考：[Nacos服务注册源码分析流程图](https://blog.csdn.net/Saintmm/article/details/121981184)

# 二、找源码入口

spring-cloud-commons包中定义了一套服务发现的规范，核心逻辑在`DiscoveryClient`接口中；
![在这里插入图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/75ebfd400023456faaafd95e7d9cdbf7.png "在这里插入图片描述")
集成Spring Cloud实现服务发现的组件都会实现`DiscoveryClient`接口；nacos-discovery包下的`NacosDiscoveryClient`类实现`DiscoveryClient`接口。
![在这里插入图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0c330a7ce5c744c4a2b25dc024b2430f.png "在这里插入图片描述")

# 三、客户端服务发现

> 1、当nacos客户端运?起来之后，它只是去做服务注册、配置获取等操作；并不会立即去请求服务信息；
> 2、当第一次请求时候，才会去获取服务，即`懒加载机制`；

#### 1）先从本地缓存serviceInfoMap中获取服务实例信息，获取不到则通过`NamingProxy`调用Nacos 服务端获取服务实例信息；最后开启定时任务每秒请求服务端 获取实例信息列表进而更新本地缓存serviceInfoMap；

```
// NacosDiscoveryClient#getInstances()
public List<ServiceInstance> getInstances(String serviceId) {
    try {
        // 通过NacosNamingService获取服务对应的实例信息；点进去
        List<Instance> instances = discoveryProperties.namingServiceInstance()
                .selectInstances(serviceId, true);
        return hostToServiceInstanceList(instances, serviceId);
    } catch (Exception e) {
        throw new RuntimeException(
                "Can not get hosts from nacos server. serviceId: " + serviceId, e);
    }
}

// NacosNamingService#selectInstances()
public List<Instance> selectInstances(String serviceName, boolean healthy) throws NacosException {
    return selectInstances(serviceName, new ArrayList<String>(), healthy);
}
public List<Instance> selectInstances(String serviceName, List<String> clusters, boolean healthy)
    throws NacosException {
    // 默认走订阅模式
    return selectInstances(serviceName, clusters, healthy, true);
}
public List<Instance> selectInstances(String serviceName, List<String> clusters, boolean healthy,
                                      boolean subscribe) throws NacosException {
    // 默认查询DEFAULT_GROUP下的服务实例信息
    return selectInstances(serviceName, Constants.DEFAULT_GROUP, clusters, healthy, subscribe);
}
public List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy, boolean subscribe) throws NacosException {

    ServiceInfo serviceInfo;
    // 默认走订阅模式，即subscribe为TRUE
    if (subscribe) {
        serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName), StringUtils.join(clusters, ","));
    } else {
        serviceInfo = hostReactor.getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName), StringUtils.join(clusters, ","));
    }
    return selectInstances(serviceInfo, healthy);
}
```

`HostReactor#getServiceInfo()`方法是真正获取服务实例信息的地方：

```
public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {

    NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
    String key = ServiceInfo.getKey(serviceName, clusters);
    if (failoverReactor.isFailoverSwitch()) {
        return failoverReactor.getService(key);
    }

    // 1、从本地缓存serviceInfoMap中获取实例信息
    ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);

    // 2、如果本地缓存中没有，则走HTTP调用从Nacos服务端获取
    if (null == serviceObj) {
        serviceObj = new ServiceInfo(serviceName, clusters);

        serviceInfoMap.put(serviceObj.getKey(), serviceObj);

        updatingMap.put(serviceName, new Object());
        updateServiceNow(serviceName, clusters);
        updatingMap.remove(serviceName);

    } else if (updatingMap.containsKey(serviceName)) {

        if (UPDATE_HOLD_INTERVAL > 0) {
            // hold a moment waiting for update finish
            synchronized (serviceObj) {
                try {
                    serviceObj.wait(UPDATE_HOLD_INTERVAL);
                } catch (InterruptedException e) {
                    NAMING_LOGGER.error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                }
            }
        }
    }

    // 3、开启一个定时任务，每隔一秒从Nacos服务端获取最新的服务实例信息，更新到本地缓存seriveInfoMap中
    scheduleUpdateIfAbsent(serviceName, clusters);

    // 4、 从本地缓存serviceInfoMap中获取服务实例信息
    return serviceInfoMap.get(serviceObj.getKey());
}
```

1、从本地缓存中获取服务实例信息：

```
private ServiceInfo getServiceInfo0(String serviceName, String clusters) {

    String key = ServiceInfo.getKey(serviceName, clusters);

    return serviceInfoMap.get(key);
}
```

2、则走HTTP调用从Nacos服务端获取服务实例信息：

```
public void updateServiceNow(String serviceName, String clusters) {
    ServiceInfo oldService = getServiceInfo0(serviceName, clusters);
    try {

        // 通过NamingProxy走HTTP接口调用，获取服务实例信息
        String result = serverProxy.queryList(serviceName, clusters, pushReceiver.getUDPPort(), false);
        if (StringUtils.isNotEmpty(result)) {
            // 更新本地缓存serviceInfoMap
            processServiceJSON(result);
        }
    } catch (Exception e) {
        NAMING_LOGGER.error("[NA] failed to update serviceName: " + serviceName, e);
    } finally {
        if (oldService != null) {
            synchronized (oldService) {
                oldService.notifyAll();
            }
        }
    }
}
```

3、开启一个定时任务，每隔一秒从Nacos服务端获取最新的服务实例信息，更新到本地缓存seriveInfoMap中：

```
public void scheduleUpdateIfAbsent(String serviceName, String clusters) {
    if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
        return;
    }

    synchronized (futureMap) {
        if (futureMap.get(ServiceInfo.getKey(serviceName, clusters)) != null) {
            return;
        }

        // 启动定时任务
        ScheduledFuture<?> future = addTask(new UpdateTask(serviceName, clusters));
        futureMap.put(ServiceInfo.getKey(serviceName, clusters), future);
    }
}

// 定时任务执行逻辑，UpdateTask#run()
public void run() {
    try {
        ServiceInfo serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));

        if (serviceObj == null) {
            updateServiceNow(serviceName, clusters);
            executor.schedule(this, DEFAULT_DELAY, TimeUnit.MILLISECONDS);
            return;
        }

        if (serviceObj.getLastRefTime() <= lastRefTime) {
            updateServiceNow(serviceName, clusters);
            serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
        } else {
            // if serviceName already updated by push, we should not override it
            // since the push data may be different from pull through force push
            refreshOnly(serviceName, clusters);
        }

        // 开启一个定时任务，1s之后执行
        executor.schedule(this, serviceObj.getCacheMillis(), TimeUnit.MILLISECONDS);

        lastRefTime = serviceObj.getLastRefTime();
    } catch (Throwable e) {
        NAMING_LOGGER.warn("[NA] failed to update serviceName: " + serviceName, e);
    }

}
```

查询服务实例列表：

```
public String queryList(String serviceName, String clusters, int udpPort, boolean healthyOnly)
    throws NacosException {

    final Map<String, String> params = new HashMap<String, String>(8);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put("clusters", clusters);
    params.put("udpPort", String.valueOf(udpPort));
    params.put("clientIP", NetUtils.localIP());
    params.put("healthyOnly", String.valueOf(healthyOnly));

    return reqAPI(UtilAndComs.NACOS_URL_BASE + "/instance/list", params, HttpMethod.GET);
}
```

#### 2）在HostReactor实例化的时候会实例化PushReceiver，进而开启一个线程死循环通过`DatagramSocket#receive()`监听Nacos服务端中服务实例信息发生变更后的UDP通知。

```
public class PushReceiver implements Runnable {
    private DatagramSocket udpSocket;

    public PushReceiver(HostReactor hostReactor) {
        try {
            this.hostReactor = hostReactor;
            udpSocket = new DatagramSocket();
            // 启动一个线程
            executorService = new ScheduledThreadPoolExecutor(1, new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r);
                    thread.setDaemon(true);
                    thread.setName("com.alibaba.nacos.naming.push.receiver");
                    return thread;
                }
            });

            executorService.execute(this);
        } catch (Exception e) {
            NAMING_LOGGER.error("[NA] init udp socket failed", e);
        }
    }

    public void run() {
        while (true) {
            try {
                // byte[] is initialized with 0 full filled by default
                byte[] buffer = new byte[UDP_MSS];
                DatagramPacket packet = new DatagramPacket(buffer, buffer.length);

                // 监听Nacos服务端服务实例信息变更后的通知
                udpSocket.receive(packet);

                String json = new String(IoUtils.tryDecompress(packet.getData()), "UTF-8").trim();
                NAMING_LOGGER.info("received push data: " + json + " from " + packet.getAddress().toString());

                PushPacket pushPacket = JSON.parseObject(json, PushPacket.class);
                String ack;
                if ("dom".equals(pushPacket.type) || "service".equals(pushPacket.type)) {
                    hostReactor.processServiceJSON(pushPacket.data);

                    // send ack to server
                    ack = "{\"type\": \"push-ack\""
                        + ", \"lastRefTime\":\"" + pushPacket.lastRefTime
                        + "\", \"data\":" + "\"\"}";
                } else if ("dump".equals(pushPacket.type)) {
                    // dump data to server
                    ack = "{\"type\": \"dump-ack\""
                        + ", \"lastRefTime\": \"" + pushPacket.lastRefTime
                        + "\", \"data\":" + "\""
                        + StringUtils.escapeJavaScript(JSON.toJSONString(hostReactor.getServiceInfoMap()))
                        + "\"}";
                } else {
                    // do nothing send ack only
                    ack = "{\"type\": \"unknown-ack\""
                        + ", \"lastRefTime\":\"" + pushPacket.lastRefTime
                        + "\", \"data\":" + "\"\"}";
                }

                udpSocket.send(new DatagramPacket(ack.getBytes(Charset.forName("UTF-8")),
                    ack.getBytes(Charset.forName("UTF-8")).length, packet.getSocketAddress()));
            } catch (Exception e) {
                NAMING_LOGGER.error("[NA] error while receiving push data", e);
            }
        }
    }

}
```

# 四、服务端服务发现

Nacos服务端的服务发现主要做两件事：

> 1、查询服务实例列表；先从缓存serviceMap中找到service对应的Cluster，再从Cluster的两个Set：`persistentInstances`、`ephemeralInstances`获取全量的实例信息；
> 2、将客户端传来的ip、udp端口号加添加到`clientMap`，进而做服务推送；clientMap属于`NamingSubscriberService`的实现类`NamingSubscriberServiceV1Impl`，其key是service name，value是订阅了该服务的客户端列表(ip+端口号)。

见naming项目下的 InstanceController类的list()方法：

#### 1）获取服务实例列表

```
@GetMapping("/list")
@Secured(parser = NamingResourceParser.class, action = ActionTypes.READ)
public Object list(HttpServletRequest request) throws Exception {

    String namespaceId = WebUtils.optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
    String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    NamingUtils.checkServiceNameFormat(serviceName);

    String agent = WebUtils.getUserAgent(request);
    String clusters = WebUtils.optional(request, "clusters", StringUtils.EMPTY);
    String clientIP = WebUtils.optional(request, "clientIP", StringUtils.EMPTY);
    int udpPort = Integer.parseInt(WebUtils.optional(request, "udpPort", "0"));
    boolean healthyOnly = Boolean.parseBoolean(WebUtils.optional(request, "healthyOnly", "false"));

    boolean isCheck = Boolean.parseBoolean(WebUtils.optional(request, "isCheck", "false"));

    String app = WebUtils.optional(request, "app", StringUtils.EMPTY);
    String env = WebUtils.optional(request, "env", StringUtils.EMPTY);
    String tenant = WebUtils.optional(request, "tid", StringUtils.EMPTY);

    Subscriber subscriber = new Subscriber(clientIP + ":" + udpPort, agent, app, clientIP, namespaceId, serviceName,
            udpPort, clusters);
    // 进去InstanceOperatorServiceImpl#listInstance()方法获取服务实例列表
    return getInstanceOperator().listInstance(namespaceId, serviceName, subscriber, clusters, healthyOnly);
}

//InstanceOperatorServiceImpl#listInstance()
public ServiceInfo listInstance(String namespaceId, String serviceName, Subscriber subscriber, String cluster,
            boolean healthOnly) throws Exception {
        ClientInfo clientInfo = new ClientInfo(subscriber.getAgent());
        String clientIP = subscriber.getIp();
        ServiceInfo result = new ServiceInfo(serviceName, cluster);
        Service service = serviceManager.getService(namespaceId, serviceName);
        long cacheMillis = switchDomain.getDefaultCacheMillis();

        // now try to enable the push
        try {
            // 尝试启用推送服务UdpPushService，即服务实例信息发生变更时通过UDP的方式通知Nacos Client
            if (subscriber.getPort() > 0 && pushService.canEnablePush(subscriber.getAgent())) {
                subscriberServiceV1.addClient(namespaceId, serviceName, cluster, subscriber.getAgent(),
                        new InetSocketAddress(clientIP, subscriber.getPort()), pushDataSource, StringUtils.EMPTY,
                        StringUtils.EMPTY);
                cacheMillis = switchDomain.getPushCacheMillis(serviceName);
            }
        } catch (Exception e) {
            Loggers.SRV_LOG.error("[NACOS-API] failed to added push client {}, {}:{}", clientInfo, clientIP,
                    subscriber.getPort(), e);
            cacheMillis = switchDomain.getDefaultCacheMillis();
        }

        if (service == null) {
            if (Loggers.SRV_LOG.isDebugEnabled()) {
                Loggers.SRV_LOG.debug("no instance to serve for service: {}", serviceName);
            }
            result.setCacheMillis(cacheMillis);
            return result;
        }

        // 检查服务是否禁用
        checkIfDisabled(service);

        // 这里是获取服务注册信息的关键代码，获取所有永久和临时服务实例
        List<com.alibaba.nacos.naming.core.Instance> srvedIps = service
                .srvIPs(Arrays.asList(StringUtils.split(cluster, StringUtils.COMMA)));

        // filter ips using selector，选择器过滤服务
        if (service.getSelector() != null && StringUtils.isNotBlank(clientIP)) {
            srvedIps = selectorManager.select(service.getSelector(), clientIP, srvedIps);
        }

        // 如果找不到服务则返回当前服务
        if (CollectionUtils.isEmpty(srvedIps)) {
        .......
        return result;
    }

// Service#srvIPs()
public List<Instance> srvIPs(List<String> clusters) {
    if (CollectionUtils.isEmpty(clusters)) {
        clusters = new ArrayList<>();
        clusters.addAll(clusterMap.keySet());
    }
    return allIPs(clusters);
}

// Service#allIPs()
public List<Instance> allIPs(List<String> clusters) {
    List<Instance> result = new ArrayList<>();
    for (String cluster : clusters) {
        // 服务注册的时候，会将实例信息写到clusterMap中，现在从其中取
        Cluster clusterObj = clusterMap.get(cluster);
        if (clusterObj == null) {
            continue;
        }

        result.addAll(clusterObj.allIPs());
    }
    return result;
}

// Cluster#allIPs()
public List<Instance> allIPs() {
    List<Instance> allInstances = new ArrayList<>();
    // 获取服务下所有的持久化实例
    allInstances.addAll(persistentInstances);
    // 获取服务下所有的临时实例
    allInstances.addAll(ephemeralInstances);
    return allInstances;
}
```

#### 2）采用UDP方式做服务实例推送

NamingSubscriberServiceV1Impl#addClient()：

```
public void addClient(String namespaceId, String serviceName, String clusters, String agent,
        InetSocketAddress socketAddr, DataSource dataSource, String tenant, String app) {

    // 初始化推送客户端实例PushClient
    PushClient client = new PushClient(namespaceId, serviceName, clusters, agent, socketAddr, dataSource, tenant,
            app);
    // 添加推送目标客户端
    addClient(client);
}

// 重载方法addClient()
public void addClient(PushClient client) {
    // client is stored by key 'serviceName' because notify event is driven by serviceName change
    // 客户端由键“ serviceName”存储，因为通知事件由serviceName更改驱动
    String serviceKey = UtilsAndCommons.assembleFullServiceName(client.getNamespaceId(), client.getServiceName());
    ConcurrentMap<String, PushClient> clients = clientMap.get(serviceKey);
    // 如果获取不到客户端想调用的ServiceName对应的推送客户端，则新建推送客户端，并缓存
    if (clients == null) {
        clientMap.putIfAbsent(serviceKey, new ConcurrentHashMap<>(1024));
        clients = clientMap.get(serviceKey);
    }

    PushClient oldClient = clients.get(client.toString());
    // 存在老的PushClient，则刷新
    if (oldClient != null) {
        oldClient.refresh();
    } else {
        // 否则缓存PushClient
        PushClient res = clients.putIfAbsent(client.toString(), client);
        if (res != null) {
            Loggers.PUSH.warn("client: {} already associated with key {}", res.getAddrStr(), res);
        }
        Loggers.PUSH.debug("client: {} added for serviceName: {}", client.getAddrStr(), client.getServiceName());
    }
}
```

# 五、总结

客户端：

> 1、优先从本地缓存中获取服务实例信息；
> 2、维护定时任务定时从Nacos服务端获取服务实例信息；

服务端：

> 1、返回指定命名空间下内存注册表中所有的永久实例和临时实例给客户端；
> 2、开启一个UDP服务实例信息变更推送服务；


# 参考文章
https://developer.aliyun.com/article/1058262
https://ost.51cto.com/posts/14835
https://developer.aliyun.com/article/1048465
https://zhuanlan.zhihu.com/p/70478036
https://juejin.cn/post/6999814668390760484#heading-8