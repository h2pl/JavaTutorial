作者 | 一起撸Java
来源 |今日头条

**学习目标**

*   Sentinel的工作原理
    **第1章 限流原理**
    在Sentinel中，所有的资源都对应一个资源名称以及一个Entry。每一个entry可以表示一个请求。而Sentinel中，会针对当前请求基于规则的判断来实现流控的控制，原理如下图所示。

![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1296c955070646bbc74310e726679bbabfe585.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")上图仅作为设计思想的展示，图中 Slot 的顺序已和最新版 Sentinel Slot Chain 顺序不一致
当一个外部请求过来之后，会创建一个Entry，而创建Entry的同时，也会创建一系列的slot 组成一个责任链，每个slot有不同的工作职责。

*   NodeSelectorSlot 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
*   ClusterBuilderSlot 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS,thread count 等等，这些信息将用作为多维度限流，降级的依据；
*   StatisticSlot 则用于记录、统计不同纬度的 runtime 指标监控信息；
*   FlowSlot 则用于根据预设的限流规则以及前面 slot 统计的状态，来进行流量控制；
*   AuthoritySlot 则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
*   DegradeSlot 则通过统计信息以及预设的规则，来做熔断降级；
*   SystemSlot 则通过系统的状态，例如 load1 等，来控制总的入口流量；
*   LogSlot 在出现限流、熔断、系统保护时负责记录日志
*   ...
    Sentinel 将 ProcessorSlot 作为 SPI 接口进行扩展（1.7.2 版本以前 SlotChainBuilder 作为SPI），使得 Slot Chain 具备了扩展的能力。您可以自行加入自定义的 slot 并编排 slot 间的顺序，从而可以给 Sentinel 添加自定义的功能。

![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d65c2688084bf5be06d3687ce8663cb1b7167b.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")**Spring Cloud 集成Sentinel的原理**

Spring Cloud 中集成Sentinel限流，是基于拦截器来实现，具体的实现路径如下。

SentinelWebAutoConfiguration——>addInterceptors——>SentinelWebInterceptor->AbstractSentinelInterceptor



```
public boolean preHandle(HttpServletRequest request, HttpServletResponseresponse, Object handler) throws Exception {  try {    String resourceName = this.getResourceName(request);    if (StringUtil.isEmpty(resourceName)) {      return true;   } else if (this.increaseReferece(request,this.baseWebMvcConfig.getRequestRefName(), 1) != 1) {      return true;   } else {      String origin = this.parseOrigin(request);      String contextName = this.getContextName(request);      ContextUtil.enter(contextName, origin);      Entry entry = SphU.entry(resourceName, 1, EntryType.IN);     request.setAttribute(this.baseWebMvcConfig.getRequestAttributeName(), entry);      return true;   } } catch (BlockException var12) {    BlockException e = var12;    try {      this.handleBlockException(request, response, e);   } finally {      ContextUtil.exit();   }    return false; }}
```









> 资源调用的流量类型，是入口流量（ EntryType.IN ）还是出口流量（ EntryType.OUT ），注意系统规则只对 IN 生效

**第2章 SphU.entry**
不管是集成dubbo也好，还是集成到spring cloud中也好，最终都是调用SphU.entry这个方法来进行限流判断的，接下来我们从SphU.entry这个方法中去了解它的实现原理。

代码中我们可能唯一疑惑的，也是最关键的一步是 SphU.entry(resource) ， 我们传进去了一个资源，这个资源可用是方法名，可以是接口，那么他具体做了什么呢？让我们来一步步揭开他的神秘面纱：



```
public static Entry entry(String name) throws BlockException {    return Env.sph.entry(name, EntryType.OUT, 1, OBJECTS0);}public class Env {    public static final Sph sph = new CtSph();    ......//省略部分代码}
```









从 SphU.entry() 方法往下执行会进入到 Sph.entry() ，Sph的默认实现类是 CtSph,而最终会进入CtSph 的entry 方法：



```
@Overridepublic Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {　　 //封装了一个资源对象    StringResourceWrapper resource = new StringResourceWrapper(name, type);    return entry(resource, count, args);}
```









这里的主要步骤是通过我们给定的资源去封装了一个 StringResourceWrapper ，然后传入自己的重载方法，继而调用 entryWithPriority(resourceWrapper, count, false, args)：

*   ResourceWrapper 表示sentinel的资源，做了封装
*   count表示本次请求的占用的并发数量，默认是1
*   prioritized，优先级



```
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count,boolean prioritized, Object... args)    throws BlockException {    //获取上下文环境，存储在ThreadLocal中，context中会存储整个调用链    Context context = ContextUtil.getContext();    //如果是 NullContext，那么说明 context name 超过了 2000 个，参见 ContextUtil#trueEnter    //这个时候，Sentinel 不再接受处理新的 context 配置，也就是不做这些新的接口的统计、限流熔断等    if (context instanceof NullContext) {        // The {@link NullContext} indicates that the amount of context has exceeded the threshold,        // so here init the entry only. No rule checking will be done.        return new CtEntry(resourceWrapper, null, context);    }    if (context == null) {//使用默认context        // 生成Context的部分        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);    }    // Global switch is close, no rule checking will do.    if (!Constants.ON) {//全局限流开关是否已经开启，如果关闭了，就不进行限流规则检查        return new CtEntry(resourceWrapper, null, context);    }    //设计模式中的责任链模式。    //构建一个slot链表    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);    //根据 lookProcessChain 方法，我们知道，当 resource 超过 Constants.MAX_SLOT_CHAIN_SIZE，    // 也就是 6000 的时候，Sentinel 开始不处理新的请求，这么做主要是为了 Sentinel 的性能考虑    if (chain == null) {        return new CtEntry(resourceWrapper, null, context);    }    //下面这里才真正开始，生成个entry    Entry e = new CtEntry(resourceWrapper, chain, context);    try {        //开始检测限流规则        chain.entry(context, resourceWrapper, null, count, prioritized, args);    } catch (BlockException e1) {        e.exit(count, args); //被限流，抛出异常。        throw e1;    } catch (Throwable e1) {        // This should not happen, unless there are errors existing in Sentinel internal.        RecordLog.info("Sentinel unexpected exception", e1);    }    return e;//返回正常的结果}
```









从上面的代码我们可以知道，该方法中主要是获取到了本资源所对应的资源处理链，从起命名 lookProcessChain 中发现，就是去获取到一条处理链，去执行资源的整合处理，当然，这里处于限流的环境下，那么这个处理链肯定是对于当前环境下请求的流量整合限流相关的处理。可以分为以下几个部分：

*   对参全局配置项做检测，如果不符合要求就直接返回了一个CtEntry对象，不会再进行后面的限流检测，否则进入下面的检测流程。根据包装过的资源对象获取对应的SlotChain
*   执行SlotChain的entry方法，如果SlotChain的entry方法抛出了BlockException，则将该异常继续向上抛出，如果SlotChain的entry方法正常执行了，则最后会将该entry对象返回
*   如果上层方法捕获了BlockException，则说明请求被限流了，否则请求能正常执行
    **2.1 创建Context**
    InternalContextUtil.internalEnter--->trueEnter



```
protected static Context trueEnter(String name, String origin) {    //从ThreadLocal中获取，第一次肯定是null    Context context = contextHolder.get();    if (context == null) {        //这里是根据Context的名字获取Node        Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;        DefaultNode node = localCacheNameMap.get(name);        if (node == null) {            if (localCacheNameMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {                setNullContext();                return NULL_CONTEXT;            } else {                LOCK.lock();                try {                    node = contextNameNodeMap.get(name);                    if (node == null) {                        if (contextNameNodeMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {                            setNullContext();                            return NULL_CONTEXT;                        } else {                            //创建个EntranceNode                            node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);                            //加入全局的节点                            // Add entrance node.                            Constants.ROOT.addChild(node);//加入map中                            Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);                            newMap.putAll(contextNameNodeMap);                            newMap.put(name, node);                            contextNameNodeMap = newMap;                        }                    }                } finally {                    LOCK.unlock();                }            }        }        context = new Context(node, name);        context.setOrigin(origin);        //放入ThreadLocal中        contextHolder.set(context);    }    return context;}
```









这里的逻辑还是比较简单的

*   首先在ThreadLocal获取，获取不到就创建，不然就返回
*   然后再Map中根据ContextName找一个Node
*   没有找到Node就加锁的方式，创建一个EntranceNode，然后放入Map中
*   创建Context，设置node，name，origin，再放入ThreadLocal中
    到此Context就创建完成

目前Context对象的状态如下图![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/a114ee154527b7fcf24169d5291c7bac87ac93.png "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")**2.2 构建slot链**
构建一个slot链，链路的组成为

> DefaultProcessorSlotChain -> NodeSelectorSlot -> ClusterBuilderSlot -> LogSlot ->StatisticSlot -> AuthoritySlot -> SystemSlot -> ParamFlowSlot -> FlowSlot -> DegradeSlot



```
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {    //可以看出，chain链根据资源来作为key，不同的资源肯定是不同chain链    ProcessorSlotChain chain = chainMap.get(resourceWrapper);    if (chain == null) {////这里与spring(缓存bean) dubbo(双重检查锁)中如出一辙，采用缓存机制        synchronized (LOCK) {            chain = chainMap.get(resourceWrapper);            if (chain == null) {                //chainMap大小大于一个值，也就是entry数量大小限制了，一个chain对应一个entry                if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {                    return null;                }                //构建一个slot chain                chain = SlotChainProvider.newSlotChain();                //这里的逻辑是，新建一个Map大小是oldMap+1                Map<ResourceWrapper, ProcessorSlotChain> newMap = new                    HashMap<ResourceWrapper, ProcessorSlotChain>(                    chainMap.size() + 1);                //然后先整体放入oldMap，再放新建的chain                newMap.putAll(chainMap);                newMap.put(resourceWrapper, chain); //添加到newMap， 这里应该是考虑避免频繁扩容                chainMap = newMap;            }        }    }    return chain;}
```









这里的代码很清晰可以发现，首先从缓存中获取该处理链，而第一次进来肯定是没有的，所以这里会走 SlotChainProvider 去构造处理链，构造完成后将起放入缓存以备下次使用：



```
public static ProcessorSlotChain newSlotChain() {    if (slotChainBuilder != null) {        return slotChainBuilder.build();    }    // 这里通过spi机制去创建处理链，如果想自己创建slot的话只需要按照SPI机制实现SlotChainBuilder接口就好    //Sentinel默认的链在sentinel-core包下的META-INF.services下    slotChainBuilder = SpiLoader.loadFirstInstanceOrDefault(SlotChainBuilder.class, DefaultSlotChainBuilder.class);    if (slotChainBuilder == null) {        // Should not go through here.        RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");        slotChainBuilder = new DefaultSlotChainBuilder();    } else {        RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: "                       + slotChainBuilder.getClass().getCanonicalName());    }    return slotChainBuilder.build();}
```









这个方法进行了多次的校验，确保builder 不为空，然后通过其去构造这个处理链：



```
public class DefaultSlotChainBuilder implements SlotChainBuilder {    @Override    public ProcessorSlotChain build() {        ProcessorSlotChain chain = new DefaultProcessorSlotChain();        chain.addLast(new NodeSelectorSlot());        chain.addLast(new ClusterBuilderSlot());        chain.addLast(new LogSlot());        chain.addLast(new StatisticSlot());        chain.addLast(new SystemSlot());        chain.addLast(new AuthoritySlot());        chain.addLast(new FlowSlot());        chain.addLast(new DegradeSlot());        return chain;    }}
```









到了这里我们终于发现了这个处理链的组成情况，官网也有对其进行说明，毕竟是Sentinel的限流核心算法的实现腹地，我们看一下官网的介绍：

在 Sentinel 里面，所有的资源都对应一个资源名称（resourceName），每次资源调用都会创建一个 Entry 对象。Entry 可以通过对主流框架的适配自动创建，也可以通过注解的方式或调用 SphU API 显式创建。Entry 创建的时候，同时也会创建一系列功能插槽（slot chain），这些插槽有不同的职责。具体职责在上面已经提到了。

**整体流程**

整体的执行流程如下。

* NodeSelectorSlot：主要用于构建调用链。

* ClusterBuilderSlot：用于集群限流、熔断。

* LogSlot：用于记录日志。

* StatisticSlot：用于实时收集实时消息。

* AuthoritySlot：用于权限校验的。

* SystemSlot：用于验证系统级别的规则。

* FlowSlot：实现限流机制。

* DegradeSlot：实现熔断机制。
  **2.3 创建Entry**

  > Entry e = new CtEntry(resourceWrapper, chain, context);



```
CtEntry(ResourceWrapper resourceWrapper, ProcessorSlot<Object> chain, Context context) {    super(resourceWrapper);    this.chain = chain;    this.context = context;    setUpEntryFor(context);}private void setUpEntryFor(Context context) {    // The entry should not be associated to NullContext.    if (context instanceof NullContext) {        return;    }    this.parent = context.getCurEntry();    if (parent != null) {        ((CtEntry) parent).child = this;    }    context.setCurEntry(this);}

```









当第一次Entry生成的时候，context.getCurEntry必定是NULL，那么直接执行Context.setCurEntry方法

然后这个Context的状态如下图  ![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/57273a794d51cfc587c923de97943491e9da0b.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")再执行一次新的Sphu.entry后会再次新建一个Entry，这个时候curEntry不是null，那么执行((CtEntry)parent).child = this;

结果如下图![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c3c36f101ef031102dd270287bf7635e715229.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")可以看出，原来的CtEntry被移出Context，新建的CtEntry和旧CtEntry通过内部的parent和child引用相连

**2.4 NodeSelectorSlot**
这个类主要用于构建调用链，这个需要讲解一下，在后续过程中会比较关键，代码如下。



```
@Overridepublic void entry(Context context, ResourceWrapper resourceWrapper, Object obj,                  int count, boolean prioritized, Object... args)    throws Throwable {    //这里有个缓存，根据context的名字缓存node    DefaultNode node = map.get(context.getName());    //双重检测，线程安全    if (node == null) {        synchronized (this) {            node = map.get(context.getName());            if (node == null) {                //这里生成的是DefaultNode节点                node = new DefaultNode(resourceWrapper, null);                //下面这些逻辑是放入map的逻辑，因为后期map比较大，所以这样放入，性能会高一些                HashMap<String, DefaultNode> cacheMap = new HashMap<String,DefaultNode>(map.size());                cacheMap.putAll(map);                cacheMap.put(context.getName(), node);                map = cacheMap;                // 关键在这，这是修改调用链树的地方                ((DefaultNode) context.getLastNode()).addChild(node);            }        }    }    //替换context中的curEntry中的curNode    context.setCurNode(node);    fireEntry(context, resourceWrapper, node, count, prioritized, args);}
```









查询缓存中是否有这个node这里的逻辑也很简单

*   根据ContextName查询缓存是否有这个Node
*   没有就生成这个DefaultNode，放入缓存，然后构造调用树链
*   Context的curEntry中的curnode设置为这个node
    这里有几个对象要单独说明：

1、context：表示上下文，一个线程对应一个context，其中包含一些属性如下

*   name：名字
*   entranceNode：调用链入口
*   curEntry：当前entry
*   origin：调用者来源
*   async：异步
    2、Node： 表示一个节点，这个节点会保存某个资源的各个实时统计数据，通过访问某个节点，就可以获得对应资源的实时状态，根据这个信息来进行限流和降级，它有几种节点类型
*   StatisticNode：实现了Node接口，封装了基础的流量统计和获取方法
*   DefaultNode：默认节点，NodeSelectorSlot中创建的就是这个节点；代表同个资源在不同上下文中各自的流量情况
*   ClusterNode：集群节点，代表同个资源在不同上下文中总体的流量情况
*   EntranceNode：该节点表示一棵调用链树的入口节点，通过他可以获取调用链树中所有的子节点；每个上下文都会有一个入口节点，用来统计当前上下文的总体流量情况
*   OriginNode：是一个StatisticNode类型的节点，代表了同个资源请求来源的流量情况
    **2.5 StatisticSlot**
    在整个slot链路中，比较重要的，就是流量数据统计以及流量规则检测这两个slot，我们先来分析一下StatisticSlot这个对象。

StatisticSlot是 Sentinel 的核心功能插槽之一，用于统计实时的调用数据。

*   clusterNode：资源唯一标识的 ClusterNode 的 runtime 统计
*   origin：根据来自不同调用者的统计信息
*   defaultnode: 根据上下文条目名称和资源 ID 的 runtime 统计
*   入口的统计



```
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,                  boolean prioritized, Object... args) throws Throwable {    try {        // 先交由后续的限流&降级等processorSlot处理，然后根据处理结果进行统计        // Sentinel责任链的精华（不使用 for 循环遍历调用 ProcessorSlot 的原因）        fireEntry(context, resourceWrapper, node, count, prioritized, args);        //执行到这里表示通过检查，不被限流        // Request passed, add thread count and pass count.        node.increaseThreadNum(); //当前节点的请求线程数加1        node.addPassRequest(count);        //针对不同类型的node记录线程数量和请求通过数量的统计。        if (context.getCurEntry().getOriginNode() != null) {            // Add count for origin node.            context.getCurEntry().getOriginNode().increaseThreadNum();            context.getCurEntry().getOriginNode().addPassRequest(count);        }        if (resourceWrapper.getEntryType() == EntryType.IN) {            // Add count for global inbound entry node for global statistics.            Constants.ENTRY_NODE.increaseThreadNum();            Constants.ENTRY_NODE.addPassRequest(count);        }        //可调用 StatisticSlotCallbackRegistry#addEntryCallback 静态方法注册ProcessorSlotEntryCallback        for (ProcessorSlotEntryCallback<DefaultNode> handler :             StatisticSlotCallbackRegistry.getEntryCallbacks()) {            handler.onPass(context, resourceWrapper, node, count, args);        }        //优先级等待异常，这个在FlowRule中会有涉及到。    } catch (PriorityWaitException ex) {//增加线程统计        node.increaseThreadNum();        if (context.getCurEntry().getOriginNode() != null) {            // Add count for origin node.            context.getCurEntry().getOriginNode().increaseThreadNum();        }        if (resourceWrapper.getEntryType() == EntryType.IN) {            // Add count for global inbound entry node for global statistics.            Constants.ENTRY_NODE.increaseThreadNum();        }        // Handle pass event with registered entry callback handlers.        for (ProcessorSlotEntryCallback<DefaultNode> handler :             StatisticSlotCallbackRegistry.getEntryCallbacks()) {            handler.onPass(context, resourceWrapper, node, count, args);        }    } catch (BlockException e) {        // Blocked, set block exception to current entry.        context.getCurEntry().setBlockError(e); //设置限流异常到当前entry中        // Add block count.        node.increaseBlockQps(count); //增加被限流的数量        //根据不同Node类型增加阻塞限流的次数        if (context.getCurEntry().getOriginNode() != null) {            context.getCurEntry().getOriginNode().increaseBlockQps(count);        }        if (resourceWrapper.getEntryType() == EntryType.IN) {            // Add count for global inbound entry node for global statistics.            Constants.ENTRY_NODE.increaseBlockQps(count);        }        // Handle block event with registered entry callback handlers.        for (ProcessorSlotEntryCallback<DefaultNode> handler :             StatisticSlotCallbackRegistry.getEntryCallbacks()) {            handler.onBlocked(e, context, resourceWrapper, node, count, args);        }        throw e;    } catch (Throwable e) {        // Unexpected internal error, set error to current entry.        context.getCurEntry().setError(e);        throw e;    }}
```









代码分成了两部分，第一部分是entry方法，该方法首先会触发后续slot的entry方法，即SystemSlot、FlowSlot、DegradeSlot等的规则，如果规则不通过，就会抛出BlockException，则会在node中统计被block的数量。反之会在node中统计通过的请求数和线程数等信息。第二部分是在exit方法中，当退出该Entry入口时，会统计rt的时间，并减少线程数。

我们可以看到 node.addPassRequest() 这段代码是在fireEntry执行之后执行的，这意味着，当前请求通过了sentinel的流控等规则，此时需要将当次请求记录下来，也就是执行 node.addPassRequest()这行代码，我们跟进去看看：

**2.5.1 addPassRequest**
@Overridepublic void addPassRequest(int count) {    // 调用父类（StatisticNode）来进行统计    super.addPassRequest(count);    // 根据clusterNode 汇总统计（背后也是调用父类StatisticNode）    this.clusterNode.addPassRequest(count);}
首先我们知道这里的node是一个 DefaultNode 实例，在第一个NodeSelectorSlot 的entry方法中对资源进行了封装，封装成了一个DefaultNode。

*   DefaultNode：保存着某个resource在某个context中的实时指标，每个DefaultNode都指向一个ClusterNode
*   ClusterNode：保存着某个resource在所有的context中实时指标的总和，同样的resource会共享同一个ClusterNode，不管他在哪个context中
    分别调用两个时间窗口来递增请求数量。

内部实际调用的是ArrayMetric来进行请求数量的统计



```
//按照秒来统计，分成两个窗口，每个窗口500ms，用来统计QPSprivate transient volatile Metric rollingCounterInSecond = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT,        IntervalProperty.INTERVAL);//按照分钟统计，生成60个窗口，每个窗口1000msprivate transient Metric rollingCounterInMinute = new ArrayMetric(60, 60 * 1000, false);public void addPassRequest(int count) {    rollingCounterInSecond.addPass(count);    rollingCounterInMinute.addPass(count);}
```









这里采用的是滑动窗口的方式来记录请求的次数。![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/e4624bb214fd8020d1447098c8cde98a554e59.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")整个类的关系图实际上是比较清晰的，ArrayMetric实际上是一个包装类，内部通过LeapArray来实现具体的统计逻辑，而LeapArray中维护了多个WindowWrap（滑动窗口），而WindowWrap中采用了MetricBucket来进行指标数据的统计。

*   Metric: 指标收集的接口，定义滑动窗口中成功数量、异常数量、阻塞数量、TPS、响应时间等数据
*   ArrayMetric 滑动窗口核心实现类
*   LeapArray
*   WindowWrap 每一个滑动窗口的包装类，内部的数据结构采用MetricBucket
*   MetricBucket， 表示指标桶，包含阻塞数量、异常数量、成功数、响应时间等
*   MetricEvent 指标类型，通过数、阻塞数、异常数、成功数等
    **2.5.2 ArrayMetric.addPass**
    继续沿着代码往下看，进入到ArrayMetric.addPass方法。
*   从LeapArray中根据当前时间点得到对应的窗口
*   调用MetricBucket中的addPass方法，增加当前窗口中的统计次数

从代码中我们可以看到，增加指标调用 addPass 是通过一个叫 ArrayMetric 的类，现在我们在进入 ArrayMetric 中看一下。具体的代码如下所示：



```
private final LeapArray<MetricBucket> data;// SAMPLE_COUNT=2  INTERVAL=1000public ArrayMetric(int sampleCount, int intervalInMs) {  //这两个参数表示，滑动窗口的大小是2个，每一个滑动窗口的时间单位是500ms    this.data = new OccupiableBucketLeapArray(sampleCount, intervalInMs);}public void addPass(int count) {    WindowWrap<MetricBucket> wrap = data.currentWindow();    wrap.value().addPass(count);}
```









这里终于出现了与滑动窗口有那么点关联的 window了，window不就是窗户嘛，这里通过 data 来获取当前窗口。而这里的窗口大小为 sampleCount=2.我们可以看到，这里是通过 MetricBucket 来保存各项指标，其中维护了一个统计是数组LongAdder[] counters 来保存，而 WindowWrap，我们可以看到每一个 WindowWrap对象由三个部分组成：



```
public class WindowWrap<T> {　　// 时间窗口的长度    private final long windowLengthInMs;　　// 时间窗口的开始时间，单位是毫秒    private long windowStart;　　 //时间窗口的内容，在 WindowWrap 中是用泛型表示这个值的，但实际上就是 MetricBucket 类    private T value;    //......省略部分代码}
```









再看 LeapArray 这个类：



```
public abstract class LeapArray<T> {    // 时间窗口的长度    protected int windowLength;    // 采样窗口的个数    protected int sampleCount;    // 以毫秒为单位的时间间隔    protected int intervalInMs;    // 采样的时间窗口数组    protected AtomicReferenceArray<WindowWrap<T>> array;    /**     * LeapArray对象     * @param windowLength 时间窗口的长度，单位：毫秒     * @param intervalInSec 统计的间隔，单位：秒     */    public LeapArray(int windowLength, int intervalInSec) {        this.windowLength = windowLength;        // 时间窗口的采样个数，默认为2个采样窗口        this.sampleCount = intervalInSec * 1000 / windowLength;        this.intervalInMs = intervalInSec * 1000;//在以秒为单位的时间窗口中，会初始化两个长度的数组：`AtomicReferenceArray<WindowWrap<T>>array`，这个数组表示滑动窗口的大小。其中，每个窗口会占用500ms的时间。        this.array = new AtomicReferenceArray<WindowWrap<T>>(sampleCount);    }}
```









可以很清晰的看出来在 LeapArray 中创建了一个 AtomicReferenceArray 数组，用来对时间窗口中的统计值进行采样。通过采样的统计值再计算出平均值，就是我们需要的最终的实时指标的值了。可以看到我在上面的代码中通过注释，标明了默认采样的时间窗口的个数是2个，这个值是怎么得到的呢？我们回忆一下 LeapArray 对象创建，是通过在 StatisticNode 中，new了一个 ArrayMetric，然后将参数一路往上传递后创建的：



```
private transient volatile Metric rollingCounterInSecond = new ArrayMetric(SampleCountProperty.SAMPLE_COUNT,IntervalProperty.INTERVAL);
```









**2.5.3 currentWindow**
我们跟进获取当前窗口的方法 data.currentWindow() 中：



```
@Overridepublic WindowWrap<Window> currentWindow(long time) {    .....//省略部分代码    //计算当前时间在滑动窗口中的索引，计算方式比较简单，当前时间除以单个时间窗口的时间长度，再从整个时间窗口长度进行取模　　 int idx = calculateTimeIdx(timeMillis);    //计算当前时间在时间窗口中的开始时间    long windowStart = calculateWindowStart(timeMillis);    // time每增加一个windowLength的长度，timeId就会增加1，时间窗口就会往前滑动一个        while (true) {        // 从采样数组中根据索引获取缓存的时间窗口        WindowWrap<Window> old = array.get(idx);        // array数组长度不宜过大，否则old很多情况下都命中不了，就会创建很多个WindowWrap对象        //如果为空，说明此处还未初始化        if (old == null) {            // 如果没有获取到，则创建一个新的            WindowWrap<Window> window = new WindowWrap<Window>(windowLength, currentWindowStart, new Window());            // 通过CAS将新窗口设置到数组中去            if (array.compareAndSet(idx, null, window)) {                // 如果能设置成功，则将该窗口返回                return window;            } else {                // 否则当前线程让出时间片，等待                Thread.yield();            }        // 如果当前窗口的开始时间与old的开始时间相等，则直接返回old窗口        } else if (currentWindowStart == old.windowStart()) {            return old;        // 如果当前时间窗口的开始时间已经超过了old窗口的开始时间，则放弃old窗口        // 并将time设置为新的时间窗口的开始时间，此时窗口向前滑动        } else if (currentWindowStart > old.windowStart()) {            if (addLock.tryLock()) {                try {                    // if (old is deprecated) then [LOCK] resetTo currentTime.                    return resetWindowTo(old, currentWindowStart);                } finally {                    addLock.unlock();                }            } else {                Thread.yield();            }        // 这个条件不可能存在        } else if (currentWindowStart < old.windowStart()) {            // Cannot go through here.            return new WindowWrap<Window>(windowLength, currentWindowStart, new Window());        }    }}
```









代码很长，我们逐步将其分解，我们实际可以把他分成以下几步：

1.  根据当前时间，算出该时间的timeId，并根据timeId算出当前窗口在采样窗口数组中的索引idx。
2.  根据当前时间算出当前窗口的应该对应的开始时间time，以毫秒为单位。
3.  根据索引idx，在采样窗口数组中取得一个时间窗口。
4.  循环判断直到获取到一个当前时间窗口 old 。

*   如果old为空，则创建一个时间窗口，并将它插入到array的第idx个位置，array上面已经分析过了，是一个 AtomicReferenceArray。
*   如果当前窗口的开始时间time与old的开始时间相等，那么说明old就是当前时间窗口，直接返回old。
*   如果当前窗口的开始时间time大于old的开始时间，则说明old窗口已经过时了，将old的开始时间更新为最新值：time，进入下一次得循环再判断当前窗口的开始时间time与old的开始时间相等的时候返回。
*   如果当前窗口的开始时间time小于old的开始时间，实际上这种情况是不可能存在的，因为time是当前时间，old是过去的一个时间。
    另外timeId是会随着时间的增长而增加，当前时间每增长一个windowLength的长度，timeId就加1。但是idx不会增长，只会在0和1之间变换，因为array数组的长度是2，只有两个采样时间窗口。至于为什么默认只有两个采样窗口，个人觉得因为sentinel是比较轻量的框架。时间窗口中保存着很多统计数据，如果时间窗口过多的话，一方面会占用过多内存，另一方面时间窗口过多就意味着时间窗口的长度会变小，如果时间窗口长度变小，就会导致时间窗口过于频繁的滑动。先来看一下其中的第一步及第二步：



```
private int calculateTimeIdx(/*@Valid*/ long timeMillis) {    // time每增加一个windowLength的长度，timeId就会增加1，时间窗口就会往前滑动一个    long timeId = timeMillis / windowLengthInMs;     // idx被分成[0,arrayLength-1]中的某一个数，作为array数组中的索引    return (int)(timeId % array.length());}protected long calculateWindowStart(/*@Valid*/ long timeMillis) {    return timeMillis - timeMillis % windowLengthInMs;}
```









根据当前时间除于 windowLength 得到一个 timeId(相差500ms计算出来的值将是一致的),再用timeId跟取样窗口的长度进行一个取模，那么她一定会落在 0，1两个位置的其中一个。然后根据当前时间算出当前窗口的应该对应的开始时间time。由于刚刚开始的时候 array 是空的，那么她获取到的old应当是null，那么他会创建一个新的实例，我们用图看一下初始化的 LeapArray：

对应上面 currentWindow 方法的 4.1 步骤(假设idx=0)：![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/95b0b64926eae4d2753986af9c978da9980da0.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")当获取到的是null,那么初始的时候arrays数组中只有一个窗口(可能是第一个(idx=0)，也可能是第二个(idx=1))，每个时间窗口的长度是500ms，这就意味着只要当前时间与时间窗口的差值在500ms之内，时间窗口就不会向前滑动。例如，假如当前时间走到300或者500时，当前时间窗口仍然是相同的那个：

对应上面 currentWindow 方法的 4.2 步骤：![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/85c047336a2c2186bcc7737a4c28da852cb603.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")时间继续往前走，当超过500ms时，时间窗口就会向前滑动到下一个，这时就会更新当前窗口的开始时间,时间继续往前走，只要不超过1000ms，则当前窗口不会发生变化，其中代码实现是 resetWindowTo 方法：



```
protected WindowWrap<MetricBucket> resetWindowTo(WindowWrap<MetricBucket> w, long time) {    // Update the start time and reset value.    // 重置windowStart    w.resetTo(time);    MetricBucket borrowBucket = borrowArray.getWindowValue(time);    if (borrowBucket != null) {        w.value().reset();        w.value().addPass((int)borrowBucket.pass());    } else {        w.value().reset();    }    return w;}
```









对应上面 currentWindow 方法的 4.3 步骤![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/03d6e3886f95f5768e05969d7eb8307c26f381.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")当时间继续往前走，当前时间超过1000ms时，就会再次进入下一个时间窗口，此时arrays数组中的窗口将会有一个失效，会有另一个新的窗口进行替换：![SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241926a53c2a5b06938844199046d31e593d62.jpg "SpringCloud Alibaba系列——15Sentinel原理分析-开源基础软件社区")以此类推随着时间的流逝，时间窗口也在发生变化，在当前时间点中进入的请求，会被统计到当前时间对应的时间窗口中，回到addpass 方法中：



```
public void addPass(int count) {    WindowWrap<MetricBucket> wrap = data.currentWindow();    wrap.value().addPass(count);}
```









获取到窗口以后会进入到 wrap.value().addPass(count); QPS的增加。而这里的 wrap.value() 得到的是之前提到的 MetricBucket ，在 Sentinel 中QPS相关数据的统计结果是维护在这个类的 LongAdder[] 中，最终由这个指标来与我们实现设置好的规则进行匹配，查看是否限流，也就是 StatisticSlot的entry 方法中的 fireEntry(context, resourceWrapper, node, count, prioritized, args); 都要先进入到 FlowSlot的entry方法进行限流过滤：



```
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,                  boolean prioritized, Object... args) throws Throwable {    checkFlow(resourceWrapper, context, node, count, prioritized);    fireEntry(context, resourceWrapper, node, count, prioritized, args);}
```









可以看到这里有个很重要的方法 checkFlow ，进去看看：



```
public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource,                      Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {    if (ruleProvider == null || resource == null) {        return;    }    Collection<FlowRule> rules = ruleProvider.apply(resource.getName());    if (rules != null) {        for (FlowRule rule : rules) {            if (!canPassCheck(rule, context, node, count, prioritized)) {                throw new FlowException(rule.getLimitApp(), rule);            }        }    }}
```









到这里一切都应该清晰了，这里拿到了我们设置的 FlowRule 循环匹配资源进行限流过滤。这就是Sentinel 能做到限流的原因。

**2.6 FlowRuleSlot**
这个 slot 主要根据预设的资源的统计信息，按照固定的次序，依次生效。如果一个资源对应两条或者多条流控规则，则会根据如下次序依次检验，直到全部通过或者有一个规则生效为止:

*   指定应用生效的规则，即针对调用方限流的；
*   调用方为 other 的规则；
*   调用方为 default 的规则。



```
@Overridepublic void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,                  boolean prioritized, Object... args) throws Throwable {    checkFlow(resourceWrapper, context, node, count, prioritized);    fireEntry(context, resourceWrapper, node, count, prioritized, args);}
```









**2.6.1 checkFlow**
进入到FlowRuleChecker.checkFlow方法中。

*   根据资源名称，找到限流规则列表
*   如果限流规则不为空，则遍历规则，调用canPassCheck方法进行校验。



```
public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider,ResourceWrapper resource,                      Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {    if (ruleProvider == null || resource == null) {        return;    }    Collection<FlowRule> rules = ruleProvider.apply(resource.getName());    if (rules != null) {        for (FlowRule rule : rules) {            if (!canPassCheck(rule, context, node, count, prioritized)) {                throw new FlowException(rule.getLimitApp(), rule);            }        }    }}
```









**2.6.2 canPassCheck**
判断是否是集群限流模式，如果是，则走passClusterCheck，否则，调用passLocalCheck方法。



```
public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context,DefaultNode node, int acquireCount,                            boolean prioritized) {    String limitApp = rule.getLimitApp();    if (limitApp == null) {        return true;    }    if (rule.isClusterMode()) {        return passClusterCheck(rule, context, node, acquireCount, prioritized);    }    return passLocalCheck(rule, context, node, acquireCount, prioritized);}
```









**2.6.3 passLocalCheck**

*   selectNodeByRequesterAndStrategy，根据请求和策略来获得Node
*   rule.getRater(), 根据不同的限流控制行为来，调用canPass进行校验。



```
private static boolean passLocalCheck(FlowRule rule, Context context,DefaultNode node, int acquireCount,                                      boolean prioritized) {    Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);    if (selectedNode == null) {        return true;    }    return rule.getRater().canPass(selectedNode, acquireCount, prioritized);}
```









**2.6.4 DefaultController.canPass**
通过默认的限流行为（直接拒绝），进行限流判断。



```
@Overridepublic boolean canPass(Node node, int acquireCount, boolean prioritized) {    //先根据node获取资源当前的使用数量，这里会根据qps或者并发数策略来获得相关的值    int curCount = avgUsedTokens(node);    //当前已使用的请求数加上本次请求的数量是否大于阈值    if (curCount + acquireCount > count) {//如果为true，说明应该被限流        // 如果此请求是一个高优先级请求，并且限流类型为qps，则不会立即失败，而是去占用未来的时间窗口，等到下一个时间窗口通过请求。        if (prioritized && grade == RuleConstant.FLOW_GRADE_QPS) { //            long currentTime;            long waitInMs;            currentTime = TimeUtil.currentTimeMillis();            waitInMs = node.tryOccupyNext(currentTime, acquireCount, count);            if (waitInMs < OccupyTimeoutProperty.getOccupyTimeout()) {                node.addWaitingRequest(currentTime + waitInMs, acquireCount);                node.addOccupiedPass(acquireCount);                sleep(waitInMs);                // PriorityWaitException indicates that the request will pass after waiting for {@link @waitInMs}.                throw new PriorityWaitException(waitInMs);            }        }        return false;    }    return true;}
```









一旦被拒绝，则抛出 FlowException 异常。

**2.7 PriorityWait**
在DefaultController.canPass中，调用如下代码去借后续的窗口



```
node.addWaitingRequest(currentTime + waitInMs, acquireCount);node.addOccupiedPass(acquireCount);
```









> addWaitingRequest -> ArrayMetric.addWaiting->OccupiableBucketLeapArray.addWaiting

borrowArray，它是一个FutureBucketLeapArray对象，这里定义的是未来的时间窗口，然后获得未来时间的窗口去增加计数



```
@Overridepublic void addWaiting(long time, int acquireCount) {    WindowWrap<MetricBucket> window = borrowArray.currentWindow(time);    window.value().add(MetricEvent.PASS, acquireCount);}
```









最终，在StatisticSlot.entry中，捕获异常

如果存在优先级比较高的任务，并且当前的请求已经达到阈值，抛出这个异常，实际上是去占用未来的一个时间窗口去进行计数，抛出这个异常之后，会进入到StatisticSlot中进行捕获。然后直接通过



```
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,                  boolean prioritized, Object... args) throws Throwable {    try{        //...    } catch (PriorityWaitException ex) {        node.increaseThreadNum();        if (context.getCurEntry().getOriginNode() != null) {            // Add count for origin node.            context.getCurEntry().getOriginNode().increaseThreadNum();        }        if (resourceWrapper.getEntryType() == EntryType.IN) {            // Add count for global inbound entry node for global statistics.            Constants.ENTRY_NODE.increaseThreadNum();        }        // Handle pass event with registered entry callback handlers.        for (ProcessorSlotEntryCallback<DefaultNode> handler :             StatisticSlotCallbackRegistry.getEntryCallbacks()) {            handler.onPass(context, resourceWrapper, node, count, args);        }    }}
```



