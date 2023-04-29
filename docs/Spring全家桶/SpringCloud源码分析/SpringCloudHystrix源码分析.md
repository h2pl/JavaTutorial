学习目标

1. 手写Mini版的Hystrix
2. RxJava知识梳理
3. Hystrix的核心流程分析
4. 源码验证
   第1章 手写Mini版
   上文中已经给大家介绍过了Hystrix的核心功能和使用了，它无非就是提供了熔断、降级、隔离等功能，其中熔断和隔离是目的，降级是结果。在使用过程中其实最核心的有三个注解：@EnableHystrix、@HystrixCommand和@HystrixCollapser。可以通过注解 @HystrixCommand、或者继承 HystrixCommand 来实现降级，以及一些请求合并等操作。

在正式讲解原理之前，我们首先要明确一个点，当采用 @HystrixCommand 注解来实现服务降级，在Hystrix 的内部是采用AOP的方式进行拦截处理请求的，这块内容，后面也会详细分析。我们这里就先来实现一下简易版的 Hystrix 来体会一下，主要分为以下步骤

- 定义自己的@HystrixCommand 注解。
- 实现拦截请求的处理逻辑。
- 测试调用。
  1.自定义注解



    @Target({ElementType.METHOD})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface MyHystrixCommand {
        //默认超时时间
        int timeout() default 1000;
        //回退方法
        String fallback() default "";
    }









2.自定义切面类



    @Aspect  //开启Aspect支持并且标记为一个切面类
    @Component
    public class MyHystrixCommandAspect {
        ExecutorService executorService= Executors.newFixedThreadPool(10);
    
        //定义切点
        @Pointcut(value = "@annotation(MyHystrixCommand)")
        public void pointCut(){
    
        }
        //在切点方法外环绕执行  @Around相当于@Before和@AfterReturning功能的总和
        @Around(value = "pointCut()&&@annotation(hystrixCommand)")
        public Object doPointCut(ProceedingJoinPoint joinPoint, MyHystrixCommand hystrixCommand) throws Exception {
            int timeout=hystrixCommand.timeout();
            Future future=executorService.submit(()->{
                try {
                    //执行proceed方法的作用是让目标方法执行
                    return joinPoint.proceed();
                } catch (Throwable throwable) {
                    throwable.printStackTrace();
                }
                return null;
            });
            Object rs;
            try {
                //通过get的异步等待来实现超时
                rs=future.get(timeout, TimeUnit.MILLISECONDS);
            } catch (InterruptedException | ExecutionException | TimeoutException e) {
                future.cancel(true);
                if(StringUtils.isBlank(hystrixCommand.fallback())){
                    throw new Exception("fallback is null");
                }
                //调用fallback
                rs=invokeFallback(joinPoint,hystrixCommand.fallback());
            }
            return rs;
        }
        private Object invokeFallback(ProceedingJoinPoint joinPoint,String fallback) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException {
            //获取被代理的方法参数和Method
            MethodSignature signature=(MethodSignature)joinPoint.getSignature(); //获取代理类和被代理类的信息
            Method method=signature.getMethod();
            Class<?>[] parameterTypes=method.getParameterTypes();
            //得到回调方法
            try {
                Method fallbackMethod=joinPoint.getTarget().getClass().getMethod(fallback,parameterTypes);
                method.setAccessible(true);
                //通过反射回调
                return fallbackMethod.invoke(joinPoint.getTarget(),joinPoint.getArgs());
            } catch (Exception e) {
                throw e;
            }
        }
    }









3.自定义测试



    @RestController
    public class MyHystrixController {
        @Autowired
        OrderServiceClient orderServiceClient;
        @MyHystrixCommand(fallback = "fallback",timeout = 2000)
        @GetMapping("/myhystrix/get/{num}")
        public String get(@PathVariable("num") int num){
            return orderServiceClient.orderLists(num);
        }
        public String fallback(int num){
            return "自定义注解方法被降级";
        }
    }









当请求http://localhost:8080/myhystrix/get/1时会触发降级，因为在服务端，当num=1时会休眠3s。

OK，这样我们就实现了一个简易版的HystrixCommand，但是我们只是实现了Hystrix的第一步，定义了一个注解和切面，但是它的底层逻辑远远没有这么简单，在讲源码之前，我们先来捋一捋RxJava是什么，因为Hystrix底层逻辑是基于响应式编程实现的。

第2章 RxJava体验
2.1 RxJava概述
RxJava 是一种响应式编程，来创建基于事件的异步操作库。基于事件流的链式调用、逻辑清晰简洁。

RxJava观察者模式的对比

- 传统观察者是一个被观察者多过观察者，当被观察者发生改变时候及时通知所有观察者
- RxJava是一个观察者多个被观察者，被观察者像链条一样串起来，数据在被观察者之间朝着一个方向传递，直到传递给观察者 。
  其实说白了，就是在RxJava中存在2种概念，一种是被观察者，一种是观察者，当多个被观察者订阅了同一个观察者的时候，那么随着被观察者完成某个事件的时候就会去回调观察者。

2.2 观察者

Observer



    Observer observer = new Observer() {
        @Override
        public void onCompleted() {
            System.out.println("当被观察者生产Complete事件，调用该方法");
        }
        @Override
        public void onError(Throwable throwable) {
            System.out.println("对Error事件作出响应");
        }
        @Override
        public void onNext(Object o) {
            System.out.println("对Next事件作出响应:" + o);
        }
    };











    //Subscriber类 = RxJava 内置的一个实现了 Observer 的抽象类，对 Observer 接口进行了扩展
    Subscriber subscriber = new Subscriber() {
        @Override
        public void onCompleted() {
            System.out.println("当被观察者生产Complete事件，调用该方法");
        }
        @Override
        public void onError(Throwable throwable) {
            System.out.println("对Error事件作出响应");
        }
        @Override
        public void onNext(Object o) {
            System.out.println("对Next事件作出响应:" + o);
        }
    };









Subscriber 抽象类与Observer 接口的区别

二者基本使用方式一致（在RxJava的subscribe过程中，Observer会先被转换成Subscriber再使用）
Subscriber抽象类对 Observer 接口进行了扩展，新增了两个方法：

- onStart()：在还未响应事件前调用，用于做一些初始化工作，他是在subscribe 所在的线程调用，不能切换线程，所以不能进行界面UI更新比如弹框这些。
- unsubscribe()：用于取消订阅。在该方法被调用后，观察者将不再接收响应事件，比如在onStop方法中可以调用此方法结束订阅。调用该方法前，先使用 isUnsubscribed() 判断状态，确定被观察者Observable是否还持有观察者Subscriber的引用。
  2.3 被观察者
  RxJava 提供了多种方法用于 创建被观察者对象Observable，这里介绍两种



    // 方法1：just(T...)：直接将传入的参数依次发送出来
    Observable observable = Observable.just("A", "B", "C");
    // 将会依次调用：
    // onNext("A");
    // onNext("B");
    // onNext("C");
    // onCompleted();
    // 方法2：fromArray(T[]) / from(Iterable<? extends T>) : 将传入的数组 / Iterable 拆分成具体对象后，依次发送出来
    String[] words = {"A", "B", "C"};
    Observable observable = Observable.fromArray(words);
    // 将会依次调用：
    // onNext("A");
    // onNext("B");
    // onNext("C");
    // onCompleted();









2.4 订阅



    observable.subscribe(observer); //建立订阅关系









2.5 案例



    public class RxJavaDemo {
        // ReactiveX Java  响应式编程框架(android）
        // Java stream() java8
        //观察者模式
        public static void main(String[] args) throws ExecutionException, InterruptedException {
            final String[] datas = new String[]{"事件1"};
            // 命令执行完的回调操作 终止命令清理
            //会在Observable结束前触发回调该call方法，无论是正常还是异常终止
            final Action0 onComplated = new Action0() {
                @Override
                public void call() {
                    System.out.println("被观察者要结束了");
                }
            };
            //被观察者
            Observable<String> observable = Observable.defer(new Func0<Observable<String>>() {
                @Override
                public Observable<String> call() {
                    Observable observable1 = Observable.from(datas);
                    return observable1.doOnCompleted(onComplated);
                }
            });
    //        Observable<String> observable = Observable.just("事件1","事件2","结束");
            //观察者
            Observer observer = new Observer() {
                @Override
                public void onCompleted() {
                    System.out.println("对Comlate事件做出响应");
                }
                @Override
                public void onError(Throwable throwable) {
                    System.out.println("对Error事件作出响应");
                }
                @Override
                public void onNext(Object o) {
                    System.out.println("对Next事件作出响应:" + o);
                }
            };
            observable.subscribe(observer); //建立订阅关系
    
    //        String s = observable.toBlocking().toFuture().get();//异步等待结果
    //        System.out.println(s);
        }
    }









OK，大体的指导如何使用RxJava编程了，记下来我们开始撸源码。

第3章 源码解析
先上官网提供的源码流程图，从图上可以看出来，其实就是先去扫描带有HystrixCommand注解的方法，然后进行切面拦截，执行切面的逻辑。这个切面定义了两个方法：execute和queue，二选一进行调用，然后进入真正的拦截逻辑。所以入口是HystrixCommand注解，而开启Hystrix是@EnableHystrix注解。



    @SpringBootApplication
    @EnableFeignClients("com.example.clients")
    //@EnableDiscoveryClient //注销表示User服务不注册
    @EnableHystrix //注解方式开启Hystrix
    public class HystrixEclipseUserApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(HystrixEclipseUserApplication.class, args);
        }
    
    }









进入到@EnableHystrix注解中



    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @EnableCircuitBreaker
    public @interface EnableHystrix {
    }
    //最终@EnableHystrix继承了@EnableCircuitBreaker
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Inherited
    @Import(EnableCircuitBreakerImportSelector.class)
    public @interface EnableCircuitBreaker {
    }









看到这步代码，我相信很多学过springboot的同学都很熟悉了，这里用到了Import注解，那肯定是引进来一些配置类了，然后我们再进
EnableCircuitBreakerImportSelector类中;



    @Order(Ordered.LOWEST_PRECEDENCE - 100)
    public class EnableCircuitBreakerImportSelector
    		extends SpringFactoryImportSelector<EnableCircuitBreaker> {
    	@Override
    	protected boolean isEnabled() {
    		return getEnvironment().getProperty("spring.cloud.circuit.breaker.enabled",
    				Boolean.class, Boolean.TRUE);
    	}
    }









EnableCircuitBreakerImportSelector继承了SpringFactoryImportSelector，进入SpringFactoryImportSelector类后发现是我们熟悉的代码，它实现了DeferredImportSelector接口，实现了selectImports方法，selectImports方法会从配置文件spring.factories里加载对应的类 org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker，我们来看看spring.facotries文件。



    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.cloud.netflix.hystrix.HystrixAutoConfiguration,\
    org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerAutoConfiguration,\
    org.springframework.cloud.netflix.hystrix.ReactiveHystrixCircuitBreakerAutoConfiguration,\
    org.springframework.cloud.netflix.hystrix.security.HystrixSecurityAutoConfiguration
    org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
    org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerConfiguration









对应EnableAutoConfiguration的这些实现类在spring启动的时候通过自动装配机制会去实例化并且注入到IoC容器中，这里我们核心关注
HystrixCircuitBreakerConfiguration类。



    @Configuration(proxyBeanMethods = false)
    public class HystrixCircuitBreakerConfiguration {
        //这里是核心的切面bean
    	@Bean
    	public HystrixCommandAspect hystrixCommandAspect() {
    		return new HystrixCommandAspect();
    	}
    	...
    }









进入到这个切面类中会发现，这个切面主要针对了两个注解作为切入点@HystrixCommand和@HystrixCollapser，当执行这两个注解修饰的方法时，会被拦截执行
methodsAnnotatedWithHystrixCommand

3.1 HystrixCommandAspect



    @Aspect
    public class HystrixCommandAspect {
        private static final Map<HystrixPointcutType, MetaHolderFactory> META_HOLDER_FACTORY_MAP;
        static {
            //通过静态方法将两个注解的两个工厂实例化
            META_HOLDER_FACTORY_MAP = ImmutableMap.<HystrixPointcutType, MetaHolderFactory>builder()
                .put(HystrixPointcutType.COMMAND, new CommandMetaHolderFactory())
                .put(HystrixPointcutType.COLLAPSER, new CollapserMetaHolderFactory())
                .build();
        }
        //定义切入点注解HystrixCommand
        @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")
        public void hystrixCommandAnnotationPointcut() {
        }
        //定义切入点注解HystrixCollapser（请求合并）
        @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
        public void hystrixCollapserAnnotationPointcut() {
        }
        //环绕通知
        @Around("hystrixCommandAnnotationPointcut() || hystrixCollapserAnnotationPointcut()")
        public Object methodsAnnotatedWithHystrixCommand(final ProceedingJoinPoint joinPoint) throws Throwable {
            //获取目标方法
            Method method = getMethodFromTarget(joinPoint);
            Validate.notNull(method, "failed to get method from joinPoint: %s", joinPoint);
            //只处理这两种注解标注的方法
            if (method.isAnnotationPresent(HystrixCommand.class) && method.isAnnotationPresent(HystrixCollapser.class)) {
                throw new IllegalStateException("method cannot be annotated with HystrixCommand and HystrixCollapser " +
                                                "annotations at the same time");
            }
            //根据不同的注解，选择对应的metaHolderFactory, 创建MetaHolder, MetaHolder 里面包含了所有信息
            MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
            //获取目标方法的的元数据，方法签名，参数等
            MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
            /**
             * 创建处理器CommandCollapser 或 GenericCommand （同步） 或GenericObservableCommand（异步）
             * GenericCommand里有很多super，最终通过HystrixCommandBuilderFactory.getInstance().create(metaHolder) 构建了一个HystrixCommandBuilder作为GenericCommad的参数
             * new  GenericCommand 通过super到AbstractHystrixCommand，
             * AbstractHystrixCommand 通过super到HystrixCommand，
             * HystrixCommand最终到了AbstractCommand  一路传递
             * 一会在AbstractCommand中分析下
             */
            HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
            //根据返回值推断执行类型
            ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ?
                metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();
            //根据不同的命令类型，执行命令，返回结果
            Object result;
            try {
                //是否是响应式的（由于我们这些都是同步的会走这个逻辑）
                if (!metaHolder.isObservable()) {
                    //execute执行
                    result = CommandExecutor.execute(invokable, executionType, metaHolder);
                } else {
                    result = executeObservable(invokable, executionType, metaHolder);
                }
            } catch (HystrixBadRequestException e) {
                throw e.getCause();
            } catch (HystrixRuntimeException e) {
                throw hystrixRuntimeExceptionToThrowable(metaHolder, e);
            }
            return result;
        }
        //HystrixCommand的时候MetaHolder的创建
        private static class CommandMetaHolderFactory extends MetaHolderFactory {
            @Override
            public MetaHolder create(Object proxy, Method method, Object obj, Object[] args, final ProceedingJoinPoint joinPoint) {
                //获取注解HystrixCommand
                HystrixCommand hystrixCommand = method.getAnnotation(HystrixCommand.class);
                //根据返回结果推断任务类型，可以知道以哪种方式执行
                ExecutionType executionType = ExecutionType.getExecutionType(method.getReturnType());
                MetaHolder.Builder builder = metaHolderBuilder(proxy, method, obj, args, joinPoint);
                if (isCompileWeaving()) {
                    builder.ajcMethod(getAjcMethodFromTarget(joinPoint));
                }
                //这里没有多少参数，最重要的一个hystrixCommand，你在注解里加了什么
                return builder.defaultCommandKey(method.getName())
                    .hystrixCommand(hystrixCommand)
                    .observableExecutionMode(hystrixCommand.observableExecutionMode())  //执行模式
                    .executionType(executionType) //执行方式
                    .observable(ExecutionType.OBSERVABLE == executionType)
                    .build();
            }
        }
    }
    //在枚举ExecutionType类里
    public static ExecutionType getExecutionType(Class<?> type) {
        if (Future.class.isAssignableFrom(type)) {
            return ExecutionType.ASYNCHRONOUS;
        } else if (Observable.class.isAssignableFrom(type)) {
            return ExecutionType.OBSERVABLE;
        } else {
            return ExecutionType.SYNCHRONOUS;
        }
    }









我们重点分析下同步处理，通过代码我们可以看到HystrixInvokable 是 GenericCommand，我们同步里的看下 CommandExecutor.execute(invokable, executionType, metaHolder)



    public class CommandExecutor {
        public static Object execute(HystrixInvokable invokable, ExecutionType executionType, MetaHolder metaHolder) throws RuntimeException {
            Validate.notNull(invokable);
            Validate.notNull(metaHolder);
    
            switch (executionType) {
                case SYNCHRONOUS: {
                    //重点看同步处理这个，先把GenericCommand 转成HystrixExecutable 再执行execute
                    return castToExecutable(invokable, executionType).execute();
                }
                case ASYNCHRONOUS: {
                    // 强转成HystrixExecutable  异步执行
                    HystrixExecutable executable = castToExecutable(invokable, executionType);
                    // 如果有 fallback方法，且是异步执行，则执行并返回包装结果
                    if (metaHolder.hasFallbackMethodCommand()
                            && ExecutionType.ASYNCHRONOUS == metaHolder.getFallbackExecutionType()) {
                        return new FutureDecorator(executable.queue());
                    }
                    return executable.queue();
                }
                case OBSERVABLE: {
                    // 强转成 HystrixObservable
                    HystrixObservable observable = castToObservable(invokable);
                    // 判断执行模式是不是急切/懒惰，来选择模式执行
                    return ObservableExecutionMode.EAGER == metaHolder.getObservableExecutionMode() ? observable.observe() : observable.toObservable();
                }
                default:
                    throw new RuntimeException("unsupported execution type: " + executionType);
            }
        }
    }









这个方法主要用来执行命令，从代码中可以看出这里有三个执行类型，分别是同步、异步、以及响应式。其中，响应式又分为Cold Observable（observable.toObservable()） 和 HotObservable（observable.observe()）默认的executionType=SYNCHRONOUS ，同步请求。

- execute()：同步执行，返回一个单一的对象结果，发生错误时抛出异常。
- queue()：异步执行，返回一个 Future 对象，包含着执行结束后返回的单一结果。
- observe()：这个方法返回一个 Observable 对象，它代表操作的多个结果，但是已经被订阅者消费掉了。
- toObservable()：这个方法返回一个 Observable 对象，它代表操作的多个结果，需要咱们自己手动订阅并消费掉。
  类图关系如下：

通过GenericCommand一层层的往上翻，最终定位到HystrixCommand有个execute()



    public abstract class HystrixCommand<R> extends AbstractCommand<R> implements HystrixExecutable<R>, HystrixInvokableInfo<R>, HystrixObservable<R> {
        //同步执行
        public R execute() {
            try {
                //通过queue().get()来同步执行（封装异步处理的结果）
                return queue().get();
            } catch (Exception e) {
                throw Exceptions.sneakyThrow(decomposeException(e));
            }
        }
       //异步执行，什么时候get()，由调用者决定，get()的时候会阻塞
       public Future<R> queue() {
            //核心处理，最终定位到了AbstractCommand里的toObservable()里
            // toObservable转换为Observable,toBlocking转换为BlockingObservable, 
            // toFuture转换为Future,完成了Observable的创建和订阅
            final Future<R> delegate = toObservable().toBlocking().toFuture();   	
            final Future<R> f = new Future<R>() {
                .....
                @Override
                public R get() throws InterruptedException, ExecutionException {
                    return delegate.get();
                }
                @Override
                public R get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
                    return delegate.get(timeout, unit);
                }       	
            };
            //特殊处理了下，已经执行完了，get()也不会阻塞了
            if (f.isDone()) {
                try {
                    f.get();
                    return f;
                } catch (Exception e) {
                    ...
                }
            }
            return f;
        }
    }









在上述代码中，重点来了，构建了一个
java.util.concurrent.Future ，然后调用 get的时候委派给 delegate，而 delegate来自于 toObservable().toBlocking().toFuture(); 这正是我们上面例子里面得代码。所以现在的重点应该放在 toObservable() 方法中：

3.2 toObservable
通过Observable定义一个被观察者，这个被观察者会被toObservable().toBlocking().toFuture() ，实际上这行代码的核心含义就是去根据一些熔断逻辑判断是执行真实的业务逻辑还是执行fallback的回调方法，然后将结果返回给Future。里面的 run() 方法就是执行正常的业务逻辑。这个方法主要做了以下几件事：

- 创建一堆的动作，我也不知道这些动作是干啥的，不重要。
- 判断是否开启了缓存，如果开了，而且也命中了，就去缓存里面以Observable形式返回一个缓存结果
- 创建一个被观察者，这个被观察者后面会去回调真实业务逻辑或者fallback。

核心逻辑是这个被观察者会去执行applyHystrixSemantics里面的动作



    public Observable<R> toObservable() {
        final AbstractCommand<R> _cmd = this;
        // 命令执行完的回调操作 终止命令清理
        //会在Observable结束前触发回调该call方法，无论是正常还是异常终止
        final Action0 terminateCommandCleanup = new Action0() {
    		...
        };
        // 将命令标记为已取消并存储延迟（除了标准清理）
        //取消订阅时的监听会进行回调该 call方法
        final Action0 unsubscribeCommandCleanup = new Action0() {
            @Override
            public void call() {
    			...
            }
        };
        // 执行命令时的回调
        final Func0<Observable<R>> applyHystrixSemantics = new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                if (commandState.get().equals(CommandState.UNSUBSCRIBED)) {
                    // 立即终止整个流程。
                    return Observable.never();
                }
                //返回执行命令的Observable
                return applyHystrixSemantics(_cmd);
            }
        };
        final Func1<R, R> wrapWithAllOnNextHooks = new Func1<R, R>() {
            @Override
            public R call(R r) {
    			...
            }
        };
        final Action0 fireOnCompletedHook = new Action0() {
            @Override
            public void call() {
    			...
            }
        };
        // 创建Observable,设置各种处理操作
        return Observable.defer(new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                // 设置已启动标志, CAS保证命令只执行一次
                if (!commandState.compareAndSet(CommandState.NOT_STARTED, CommandState.OBSERVABLE_CHAIN_CREATED)) {
                    IllegalStateException ex = new IllegalStateException("This instance can only be executed once. Please instantiate a new instance.");
                    //TODO make a new error type for this
                    throw new HystrixRuntimeException(FailureType.BAD_REQUEST_EXCEPTION, _cmd.getClass(), getLogMessagePrefix() + " command executed multiple times - this is not permitted.", ex, null);
                }
                // 命令开始时间戳
                commandStartTimestamp = System.currentTimeMillis();
                // 打印日志
                if (properties.requestLogEnabled().get()) {
                    // log this command execution regardless of what happened
                    if (currentRequestLog != null) {
                        currentRequestLog.addExecutedCommand(_cmd);
                    }
                }
                // 缓存开关，缓存KEY（这个是Hystrix中请求缓存功能，hystrix支持将一个请求结果缓存起来，
                // 下一个具有相同key的请求将直接从缓存中取出结果，减少请求开销）
                final boolean requestCacheEnabled = isRequestCachingEnabled();
                final String cacheKey = getCacheKey();
                // 如果配置允许缓存，先试图从缓存获取，默认 false
                if (requestCacheEnabled) {
                    HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.get(cacheKey);
                    if (fromCache != null) {
                        isResponseFromCache = true;
                        return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                    }
                }
                // 声明执行命令的Observable
                // 创建Observable, applyHystrixSemantics() 来生成Observable
                Observable<R> hystrixObservable =
                    Observable.defer(applyHystrixSemantics)
                    .map(wrapWithAllOnNextHooks);
                Observable<R> afterCache;
                // put in cache 保存请求结果到缓存中
                if (requestCacheEnabled && cacheKey != null) {
                    // wrap it for caching
                    HystrixCachedObservable<R> toCache = HystrixCachedObservable.from(hystrixObservable, _cmd);
                    HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.putIfAbsent(cacheKey, toCache);
                    if (fromCache != null) {
                        // another thread beat us so we'll use the cached value instead
                        toCache.unsubscribe();
                        isResponseFromCache = true;
                        return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                    } else {
                        // we just created an ObservableCommand so we cast and return it
                        afterCache = toCache.toObservable();
                    }
                } else {
                    afterCache = hystrixObservable;
                }
                // 生命周期回调设置
                return afterCache
                    //会在Observable结束前触发回调，无论是正常还是异常终止
                    .doOnTerminate(terminateCommandCleanup)     
                    //取消订阅时的监听
                    .doOnUnsubscribe(unsubscribeCommandCleanup) 
                    //Observable正常终止时的监听
                    .doOnCompleted(fireOnCompletedHook);
            }
        });
    }









接下来看看核心逻辑applyHystrixSemantics



    Observable<R> hystrixObservable =
        Observable.defer(applyHystrixSemantics)
        .map(wrapWithAllOnNextHooks);











    final Func0<Observable<R>> applyHystrixSemantics = new Func0<Observable<R>>() {
        @Override
        public Observable<R> call() {
            if (commandState.get().equals(CommandState.UNSUBSCRIBED)) {
                return Observable.never();
            }
            return applyHystrixSemantics(_cmd);
        }
    };









这里传入的_cmd是一个GenericCommand，最终会执行到这个GenericCommand中的run方法。

circuitBreaker.allowRequest() 这个是判断是否处于熔断状态的，true表示没有处于熔断状态，正常执行，否则，调用 handleShortCircuitViaFallback 实现服务降级，最终会回调到我们自定义的fallback方法中。

如果当前hystrix处于未熔断状态，则

- getExecutionSemaphore 判断当前策略是否为信号量还是线程池，显然默认是线程池，然后再调用tryAcquire时写死了为true。

调用executeCommandAndObserve。



    private Observable<R> applyHystrixSemantics(final AbstractCommand<R> _cmd) {
    
        executionHook.onStart(_cmd);
    
        // 是否允许请求，即断路器是否开启 ，这里也有好几种情况
        if (circuitBreaker.allowRequest()) {
            // 信号量获取
            final TryableSemaphore executionSemaphore = getExecutionSemaphore();
            final AtomicBoolean semaphoreHasBeenReleased = new AtomicBoolean(false);
    
            // 信号释放回调
            final Action0 singleSemaphoreRelease = new Action0() {
                @Override
                public void call() {
                    if (semaphoreHasBeenReleased.compareAndSet(false, true)) {
                        executionSemaphore.release();
                    }
                }
            };
    
            // 异常回调
            final Action1<Throwable> markExceptionThrown = new Action1<Throwable>() {
                @Override
                public void call(Throwable t) {
                    eventNotifier.markEvent(HystrixEventType.EXCEPTION_THROWN, commandKey);
                }
            };
    
            // 获取信号，并返回对应的 Observable
            // 是否开启信号量资源隔离，未配置走 com.netflix.hystrix.AbstractCommand.TryableSemaphoreNoOp#tryAcquire 默认返回通过
            if (executionSemaphore.tryAcquire()) {
                try {
                    executionResult = executionResult.setInvocationStartTime(System.currentTimeMillis());
                    return executeCommandAndObserve(_cmd)   // 执行命令，以下三个是回调，可以不看
                        .doOnError(markExceptionThrown)
                        .doOnTerminate(singleSemaphoreRelease)
                        .doOnUnsubscribe(singleSemaphoreRelease);
                } catch (RuntimeException e) {
                    return Observable.error(e);
                }
            } else {
                // 获取信号失败则降级
                return handleSemaphoreRejectionViaFallback();
            }
        } else {
            // 断路器已打开，直接降级
            return handleShortCircuitViaFallback();
        }
    }









先来看一下执行失败进入降级的逻辑，这里我们直接进入到 HystrixCommand#getFallbackObservable



    public abstract class HystrixCommand<R> extends AbstractCommand<R> implements HystrixExecutable<R>, HystrixInvokableInfo<R>, HystrixObservable<R> {
        @Override
        final protected Observable<R> getFallbackObservable() {
            return Observable.defer(new Func0<Observable<R>>() {
                @Override
                public Observable<R> call() {
                    try {
                        return Observable.just(getFallback());
                    } catch (Throwable ex) {
                        return Observable.error(ex);
                    }
                }
            });
        }
    }









这里的getFallback最终会回调我们自定的fallback方法。

回到executeCommandAndObserve，这个方法主要做了以下三件事情

- 定义不同的回调，doOnNext、doOnCompleted、onErrorResumeNext、doOnEach。
- 调用executeCommandWithSpecifiedIsolation。

若执行命令超时特性开启，调用 Observable.lift方法实现执行命令超时功能。



    private Observable<R> executeCommandAndObserve(final AbstractCommand<R> _cmd) {
        final HystrixRequestContext currentRequestContext = HystrixRequestContext.getContextForCurrentThread();
        // Action和Func都是定义的一个动作，Action是无返回值，Func是有返回值
        // doOnNext中的回调。即命令执行之前执行的操作
        final Action1<R> markEmits = new Action1<R>() {
            @Override
            public void call(R r) {
                if (shouldOutputOnNextEvents()) {
                    executionResult = executionResult.addEvent(HystrixEventType.EMIT);
                    eventNotifier.markEvent(HystrixEventType.EMIT, commandKey);
                }
                if (commandIsScalar()) {
                    long latency = System.currentTimeMillis() - executionResult.getStartTimestamp();
                    eventNotifier.markCommandExecution(getCommandKey(), properties.executionIsolationStrategy().get(), (int) latency, executionResult.getOrderedList());
                    eventNotifier.markEvent(HystrixEventType.SUCCESS, commandKey);
                    executionResult = executionResult.addEvent((int) latency, HystrixEventType.SUCCESS);
                    circuitBreaker.markSuccess();
                }
            }
        };
        // doOnCompleted中的回调。命令执行完毕后执行的操作
        final Action0 markOnCompleted = new Action0() {
            @Override
            public void call() {
                if (!commandIsScalar()) {
                    long latency = System.currentTimeMillis() - executionResult.getStartTimestamp();
                    eventNotifier.markCommandExecution(getCommandKey(), properties.executionIsolationStrategy().get(), (int) latency, executionResult.getOrderedList());
                    eventNotifier.markEvent(HystrixEventType.SUCCESS, commandKey);
                    executionResult = executionResult.addEvent((int) latency, HystrixEventType.SUCCESS);
                    circuitBreaker.markSuccess();
                }
            }
        };
        // onErrorResumeNext中的回调。命令执行失败后的回退逻辑
        final Func1<Throwable, Observable<R>> handleFallback = new Func1<Throwable, Observable<R>>() {
            @Override
            public Observable<R> call(Throwable t) {
                Exception e = getExceptionFromThrowable(t);
                executionResult = executionResult.setExecutionException(e);
                if (e instanceof RejectedExecutionException) {
                    // 线程调度失败回调
                    return handleThreadPoolRejectionViaFallback(e);
                } else if (t instanceof HystrixTimeoutException) {
                    // 超时回调
                    return handleTimeoutViaFallback();
                } else if (t instanceof HystrixBadRequestException) {
                    // HystrixBadRequestException 异常回调
                    return handleBadRequestByEmittingError(e);
                } else {
                    if (e instanceof HystrixBadRequestException) {
                        eventNotifier.markEvent(HystrixEventType.BAD_REQUEST, commandKey);
                        return Observable.error(e);
                    }
                    // 降级处理
                    return handleFailureViaFallback(e);
                }
            }
        };
        // doOnEach中的回调。`Observable`每发射一个数据都会执行这个回调，设置请求上下文
        final Action1<Notification<? super R>> setRequestContext = new Action1<Notification<? super R>>() {
            @Override
            public void call(Notification<? super R> rNotification) {
                setRequestContextIfNeeded(currentRequestContext);
            }
        };
        // 创建对应的 Observable，实现 线程隔离、请求发送 等操作
        Observable<R> execution;
        // 判断 超时监控功能是否打开
        if (properties.executionTimeoutEnabled().get()) {
            // HystrixObservableTimeoutOperator  转换对应的 Observable
            execution = executeCommandWithSpecifiedIsolation(_cmd)
                .lift(new HystrixObservableTimeoutOperator<R>(_cmd));
        } else {
            execution = executeCommandWithSpecifiedIsolation(_cmd);
        }
        //设置回调
        return execution.doOnNext(markEmits)
            .doOnCompleted(markOnCompleted)
            .onErrorResumeNext(handleFallback)
            .doOnEach(setRequestContext);
    }









3.3 executeCommandWithSpecifiedIsolation
这个方法首先是根据当前不同的资源隔离策略执行不同的逻辑，THREAD、SEMAPHORE。



    private Observable<R> executeCommandWithSpecifiedIsolation(final AbstractCommand<R> _cmd) {
        // 线程隔离, 是否开启 THREAD 资源隔离降级
        if (properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.THREAD) {
            //创建一个Observable
            return Observable.defer(new Func0<Observable<R>>() {
                @Override
                public Observable<R> call() {
                    executionResult = executionResult.setExecutionOccurred();
                    if (!commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.USER_CODE_EXECUTED)) {
                        return Observable.error(new IllegalStateException("execution attempted while in state : " + commandState.get().name()));
                    }
    
                    metrics.markCommandStart(commandKey, threadPoolKey, ExecutionIsolationStrategy.THREAD);
    
                    // 该命令在包装线程中超时，将立即返回，并且不会增加任何计数器或其他此类逻辑
                    if (isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT) {
                        // the command timed out in the wrapping thread so we will return immediately
                        // and not increment any of the counters below or other such logic
                        return Observable.error(new RuntimeException("timed out before executing run()"));
                    }
    
                    // 设置线程启动
                    if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.STARTED)) {
                        //we have not been unsubscribed, so should proceed
                        HystrixCounters.incrementGlobalConcurrentThreads();
                        threadPool.markThreadExecution();
                        // store the command that is being run
                        endCurrentThreadExecutingCommand = Hystrix.startCurrentThreadExecutingCommand(getCommandKey());
                        executionResult = executionResult.setExecutedInThread();
    
                        try {
                            executionHook.onThreadStart(_cmd);
                            executionHook.onRunStart(_cmd);
                            executionHook.onExecutionStart(_cmd);
                            //返回 Observable,这个函数最终会返回一个封装了我们的run()逻辑的Observable
                            return getUserExecutionObservable(_cmd);
                        } catch (Throwable ex) {
                            return Observable.error(ex);
                        }
                    } else {
                        //command has already been unsubscribed, so return immediately
                        return Observable.error(new RuntimeException("unsubscribed before executing run()"));
                    }
                }
            }).doOnTerminate(new Action0() {
                @Override
                public void call() {
                    if (threadState.compareAndSet(ThreadState.STARTED, ThreadState.TERMINAL)) {
                        handleThreadEnd(_cmd);
                    }
                    if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.TERMINAL)) {
                    }
                }
            }).doOnUnsubscribe(new Action0() {
                @Override
                public void call() {
                    if (threadState.compareAndSet(ThreadState.STARTED, ThreadState.UNSUBSCRIBED)) {
                        handleThreadEnd(_cmd);
                    }
                    if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.UNSUBSCRIBED)) {
                    }
                }
            }).subscribeOn(threadPool.getScheduler(new Func0<Boolean>() {
                @Override
                public Boolean call() {
                    return properties.executionIsolationThreadInterruptOnTimeout().get() && _cmd.isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT;
                }
            }));
        } else {
            // 信号量隔离
            return Observable.defer(new Func0<Observable<R>>() {
                @Override
                public Observable<R> call() {
                    executionResult = executionResult.setExecutionOccurred();
                    if (!commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.USER_CODE_EXECUTED)) {
                        return Observable.error(new IllegalStateException("execution attempted while in state : " + commandState.get().name()));
                    }
                    metrics.markCommandStart(commandKey, threadPoolKey, ExecutionIsolationStrategy.SEMAPHORE);
                    endCurrentThreadExecutingCommand = Hystrix.startCurrentThreadExecutingCommand(getCommandKey());
                    try {
                        executionHook.onRunStart(_cmd);
                        executionHook.onExecutionStart(_cmd);
                        // 真正的执行
                        return getUserExecutionObservable(_cmd); 
                    } catch (Throwable ex) {
                        //If the above hooks throw, then use that as the result of the run method
                        return Observable.error(ex);
                    }
                }
            });
        }
    }









- 判断是否允许发送请求，这是基于断路器实现，如果断路器打开，则进行对应回调处理（失败或降级）。
- 如果 断路器 关闭，则进行请求，先获取信号，获取失败则处理对应回调。
- 获取成功，则由方法 executeCommandAndObserve 创建对应的 Observable 实现 线程隔离、请求发送 等操作，同时注册了对应的 生命周期回调。
  3.4 getUserExecutionObservable
  然后会执行 HystrixCommand#getExecutionObservable



    abstract class AbstractCommand<R> implements HystrixInvokableInfo<R>, HystrixObservable<R> {
        private Observable<R> getUserExecutionObservable(final AbstractCommand<R> _cmd) {
            Observable<R> userObservable;
            try {
                userObservable = getExecutionObservable();
            } catch (Throwable ex) {
                userObservable = Observable.error(ex);
            }
            return userObservable
                    .lift(new ExecutionHookApplication(_cmd))
                    .lift(new DeprecatedOnRunHookApplication(_cmd));
        }
    }
    public abstract class HystrixCommand<R> extends AbstractCommand<R> implements HystrixExecutable<R>, HystrixInvokableInfo<R>, HystrixObservable<R> {
        @Override
        final protected Observable<R> getExecutionObservable() {
            return Observable.defer(new Func0<Observable<R>>() {
                @Override
                public Observable<R> call() {
                    try {
                        return Observable.just(run());
                    } catch (Throwable ex) {
                        return Observable.error(ex);
                    }
                }
            }).doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    // Save thread on which we get subscribed so that we can interrupt it later if needed
                    executionThread.set(Thread.currentThread());
                }
            });
        }
    }









这个 run() 方法在上面已经讲过了，就是真正的业务执行方法。



    @ThreadSafe
    public class GenericCommand extends AbstractHystrixCommand<Object> {
        @Override
        protected Object run() throws Exception {
            LOGGER.debug("execute command: {}", getCommandKey().name());
            return process(new Action() {
                @Override
                Object execute() {
                    return getCommandAction().execute(getExecutionType());
                }
            });
        }
    }









最终调用到我们自己的业务逻辑。
# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning
