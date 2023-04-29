要想了解spring cloud config，我们就必须了解springboot中environment环境的加载过程。



```
SpringApplication.run(AppApiApplication.class, args);

public static ConfigurableApplicationContext run(Class<?>[] primarySources,
            String[] args) {
    return new SpringApplication(primarySources).run(args);
}

```



在`SpringApplication的构造方法中`，有这么一段代码



```
setInitializers((Collection) getSpringFactoriesInstances(
                ApplicationContextInitializer.class));
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

```





```
Set<String> names = new LinkedHashSet<>(
                SpringFactoriesLoader.loadFactoryNames(type, classLoader));

```



这段代码中就是从META-INF目录下面获取`ApplicationListener和ApplicationContextInitializer`类型的所有类。
在spring-boot.jar下面spring.factories中配置了`ConfigFileApplicationListener`这个类，配置环境也就是application文件的加载都会通过这个类



```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.context.config.ConfigFileApplicationListener,\

# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

```



继续进入springboot启动阶段



```
    public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        configureHeadlessProperty();
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                    args);
            ConfigurableEnvironment environment = prepareEnvironment(listeners,
                    applicationArguments);
            configureIgnoreBeanInfo(environment);
            Banner printedBanner = printBanner(environment);
            context = createApplicationContext();
            exceptionReporters = getSpringFactoriesInstances(
                    SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
            prepareContext(context, environment, listeners, applicationArguments,
                    printedBanner);
            refreshContext(context);
            afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass)
                        .logStarted(getApplicationLog(), stopWatch);
            }
            listeners.started(context);
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            listeners.running(context);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, exceptionReporters, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }

```





```
ConfigurableEnvironment environment = prepareEnvironment(listeners,
                    applicationArguments);
prepareContext(context, environment, listeners, applicationArguments,
                    printedBanner);

```



上面这两段代码就是装载环境的代码
在`prepareEnvironment`方法中初始化了`environment`。



```
    private ConfigurableEnvironment getOrCreateEnvironment() {
        if (this.environment != null) {
            return this.environment;
        }
        switch (this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
        }
    }

```



因为我们servlet服务，所以`Environment`为`StandardServletEnvironment`，并且在父类的的构造其中执行了`customizePropertySources`方法



```
    public AbstractEnvironment() {
        customizePropertySources(this.propertySources);
        if (logger.isDebugEnabled()) {
            logger.debug("Initialized " + getClass().getSimpleName() + " with PropertySources " + this.propertySources);
        }
    }

```





```
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
        propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
        if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
            propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
        }
        super.customizePropertySources(propertySources);
    }

```





```
    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
        propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
    }

```



到这里,我们得到了`StandardServletEnvironment`并且加载的配置顺序是

1.  servletConfigInitParams
2.  servletContextInitParams
3.  jndiProperties
4.  systemProperties
5.  systemEnvironment



```
    protected void configureEnvironment(ConfigurableEnvironment environment,
            String[] args) {
        configurePropertySources(environment, args);
        configureProfiles(environment, args);
    }

    protected void configurePropertySources(ConfigurableEnvironment environment,
            String[] args) {
        MutablePropertySources sources = environment.getPropertySources();
        if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
            sources.addLast(
                    new MapPropertySource("defaultProperties", this.defaultProperties));
        }
        if (this.addCommandLineProperties && args.length > 0) {
            String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
            if (sources.contains(name)) {
                PropertySource<?> source = sources.get(name);
                CompositePropertySource composite = new CompositePropertySource(name);
                composite.addPropertySource(new SimpleCommandLinePropertySource(
                        "springApplicationCommandLineArgs", args));
                composite.addPropertySource(source);
                sources.replace(name, composite);
            }
            else {
                sources.addFirst(new SimpleCommandLinePropertySource(args));
            }
        }
    }

```



而这里我们可以看到又加入了`defaultProperties`配置在最后面以及`SimpleCommandLinePropertySource`命令行参数在最前面。因此，现在的顺序是

1.  SimpleCommandLinePropertySourcem命令行配置
2.  servletConfigInitParams
3.  servletContextInitParams
4.  jndiProperties
5.  systemProperties
6.  systemEnvironment
7.  defaultProperties



```
    public void environmentPrepared(ConfigurableEnvironment environment) {
        for (SpringApplicationRunListener listener : this.listeners) {
            listener.environmentPrepared(environment);
        }
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(
                this.application, this.args, environment));
    }

```



在这里`SpringApplicationRunListener`为`EventPublishingRunListener`,来自于前面的`getRunListeners`



```
    private SpringApplicationRunListeners getRunListeners(String[] args) {
        Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
        return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
                SpringApplicationRunListener.class, types, this, args));
    }

```



而通过构造方法初始化的时候传入了this，所以最终`initialMulticaster`中的listener为application中的listener



```
    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        for (ApplicationListener<?> listener : application.getListeners()) {
            this.initialMulticaster.addApplicationListener(listener);
        }
    }

```



所以这里会发送个事件通知，来到了`ConfigFileApplicationListener`



```
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationEnvironmentPreparedEvent) {
            onApplicationEnvironmentPreparedEvent(
                    (ApplicationEnvironmentPreparedEvent) event);
        }
        if (event instanceof ApplicationPreparedEvent) {
            onApplicationPreparedEvent(event);
        }
    }

```





```
    private void onApplicationEnvironmentPreparedEvent(
            ApplicationEnvironmentPreparedEvent event) {
        List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
        postProcessors.add(this);
        AnnotationAwareOrderComparator.sort(postProcessors);
        for (EnvironmentPostProcessor postProcessor : postProcessors) {
            postProcessor.postProcessEnvironment(event.getEnvironment(),
                    event.getSpringApplication());
        }
    }

    List<EnvironmentPostProcessor> loadPostProcessors() {
        return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class,
                getClass().getClassLoader());
    }

```



在这里有有个扩展点，通过获取`spring.factories`文件中配置的`EnvironmentPostProcessor`类调用`postProcessEnvironment`方法，自己本身也加入到了`postProcessors`集合中，所以最终又会走到`postProcessors`方法



```
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,
            SpringApplication application) {
        addPropertySources(environment, application.getResourceLoader());
    }

```





```
    protected void addPropertySources(ConfigurableEnvironment environment,
            ResourceLoader resourceLoader) {
        RandomValuePropertySource.addToEnvironment(environment);
        new Loader(environment, resourceLoader).load();
    }

    public static void addToEnvironment(ConfigurableEnvironment environment) {
        environment.getPropertySources().addAfter(
                StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                new RandomValuePropertySource(RANDOM_PROPERTY_SOURCE_NAME));
        logger.trace("RandomValuePropertySource add to Environment");
    }

        public void load() {
            this.profiles = new LinkedList<>();
            this.processedProfiles = new LinkedList<>();
            this.activatedProfiles = false;
            this.loaded = new LinkedHashMap<>();
            initializeProfiles();
            while (!this.profiles.isEmpty()) {
                Profile profile = this.profiles.poll();
                if (profile != null && !profile.isDefaultProfile()) {
                    addProfileToEnvironment(profile.getName());
                }
                load(profile, this::getPositiveProfileFilter,
                        addToLoaded(MutablePropertySources::addLast, false));
                this.processedProfiles.add(profile);
            }
            resetEnvironmentProfiles(this.processedProfiles);
            load(null, this::getNegativeProfileFilter,
                    addToLoaded(MutablePropertySources::addFirst, true));
            addLoadedPropertySources();
        }

```



在这里，又将random配置加入到systemEnvironment之后，所以我们的配置加载顺序成了
defaultProperties

1.  SimpleCommandLinePropertySourcem命令行配置
2.  servletConfigInitParams
3.  servletContextInitParams
4.  jndiProperties
5.  systemProperties
6.  systemEnvironment
7.  RandomValuePropertySource
8.  defaultProperties

路径

> classpath:/,classpath:/config/,file:./,file:./config/
> spring.config.location=
> spring.config.additional-location=

文件名默认为application

> 如果配置了spring.config.name=
> 文件名则为spring.config.name配置的名称

并且通过`load`方法将上述路径和文件名的文件加入到`environment中`，如果配置了`spring.profiles.active`，并且加载prefix + "-" + profile + fileExtension文件，也就是`application-<spring.profiles.active>.properties`和`y`文件



```
        private void load(Profile profile, DocumentFilterFactory filterFactory,
                DocumentConsumer consumer) {
            getSearchLocations().forEach((location) -> {
                boolean isFolder = location.endsWith("/");
                Set<String> names = isFolder ? getSearchNames() : NO_SEARCH_NAMES;
                names.forEach(
                        (name) -> load(location, name, profile, filterFactory, consumer));
            });
        }

        private void load(String location, String name, Profile profile,
                DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
            if (!StringUtils.hasText(name)) {
                for (PropertySourceLoader loader : this.propertySourceLoaders) {
                    if (canLoadFileExtension(loader, location)) {
                        load(loader, location, profile,
                                filterFactory.getDocumentFilter(profile), consumer);
                        return;
                    }
                }
            }
            Set<String> processed = new HashSet<>();
            for (PropertySourceLoader loader : this.propertySourceLoaders) {
                for (String fileExtension : loader.getFileExtensions()) {
                    if (processed.add(fileExtension)) {
                        loadForFileExtension(loader, location + name, "." + fileExtension,
                                profile, filterFactory, consumer);
                    }
                }
            }
        }

```



在这里通过`this.propertySourceLoaders`遍历判断是否符合该SourceLoader的后缀，如果符合，就将资源加载进来,而`propertySourceLoaders`来自于`Loader`构造方法



```
        Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
            this.environment = environment;
            this.resourceLoader = (resourceLoader != null) ? resourceLoader
                    : new DefaultResourceLoader();
            this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(
                    PropertySourceLoader.class, getClass().getClassLoader());
        }

```





```
# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader

```



因此可以加载`yml`文件和`properties`文件



```
        private void addLoadedPropertySources() {
            MutablePropertySources destination = this.environment.getPropertySources();
            List<MutablePropertySources> loaded = new ArrayList<>(this.loaded.values());
            Collections.reverse(loaded);
            String lastAdded = null;
            Set<String> added = new HashSet<>();
            for (MutablePropertySources sources : loaded) {
                for (PropertySource<?> source : sources) {
                    if (added.add(source.getName())) {
                        addLoadedPropertySource(destination, lastAdded, source);
                        lastAdded = source.getName();
                    }
                }
            }
        }

```



最终放入到了`environment`

接下来我们就来看下Spring cloud config

在`spring-cloud-context`的jar中，`spring.factories`中引入了个`BootstrapApplicationListener`
并且该配置优先于`ConfigFileApplicationListener`，因此如果我们引入了springcloud组件，那么
事件监听会优先走`BootstrapApplicationListener`



```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.cloud.bootstrap.BootstrapApplicationListener,\

```





```
    @Override
    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        ConfigurableEnvironment environment = event.getEnvironment();
        if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,
                true)) {
            return;
        }
        // don't listen to events in a bootstrap context
        if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
            return;
        }
        ConfigurableApplicationContext context = null;
        String configName = environment
                .resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
        for (ApplicationContextInitializer<?> initializer : event.getSpringApplication()
                .getInitializers()) {
            if (initializer instanceof ParentContextApplicationContextInitializer) {
                context = findBootstrapContext(
                        (ParentContextApplicationContextInitializer) initializer,
                        configName);
            }
        }
        if (context == null) {
            context = bootstrapServiceContext(environment, event.getSpringApplication(),
                    configName);
        }
        apply(context, event.getSpringApplication(), environment);
    }

```





```
    private ConfigurableApplicationContext bootstrapServiceContext(
            ConfigurableEnvironment environment, final SpringApplication application,
            String configName) {
        StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
        MutablePropertySources bootstrapProperties = bootstrapEnvironment
                .getPropertySources();
        for (PropertySource<?> source : bootstrapProperties) {
            bootstrapProperties.remove(source.getName());
        }
        String configLocation = environment
                .resolvePlaceholders("${spring.cloud.bootstrap.location:}");
        Map<String, Object> bootstrapMap = new HashMap<>();
        bootstrapMap.put("spring.config.name", configName);
        // if an app (or test) uses spring.main.web-application-type=reactive, bootstrap will fail
        // force the environment to use none, because if though it is set below in the builder
        // the environment overrides it
        bootstrapMap.put("spring.main.web-application-type", "none");
        if (StringUtils.hasText(configLocation)) {
            bootstrapMap.put("spring.config.location", configLocation);
        }
        bootstrapProperties.addFirst(
                new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
        for (PropertySource<?> source : environment.getPropertySources()) {
            if (source instanceof StubPropertySource) {
                continue;
            }
            bootstrapProperties.addLast(source);
        }
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        // Use names and ensure unique to protect against duplicates
        List<String> names = new ArrayList<>(SpringFactoriesLoader
                .loadFactoryNames(BootstrapConfiguration.class, classLoader));
        for (String name : StringUtils.commaDelimitedListToStringArray(
                environment.getProperty("spring.cloud.bootstrap.sources", ""))) {
            names.add(name);
        }
        // TODO: is it possible or sensible to share a ResourceLoader?
        SpringApplicationBuilder builder = new SpringApplicationBuilder()
                .profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
                .environment(bootstrapEnvironment)
                // Don't use the default properties in this builder
                .registerShutdownHook(false).logStartupInfo(false)
                .web(WebApplicationType.NONE);
        final SpringApplication builderApplication = builder.application();
        if(builderApplication.getMainApplicationClass() == null){
            // gh_425:
            // SpringApplication cannot deduce the MainApplicationClass here
            // if it is booted from SpringBootServletInitializer due to the
            // absense of the "main" method in stackTraces.
            // But luckily this method's second parameter "application" here
            // carries the real MainApplicationClass which has been explicitly
            // set by SpringBootServletInitializer itself already.
            builder.main(application.getMainApplicationClass());
        }
        if (environment.getPropertySources().contains("refreshArgs")) {
            // If we are doing a context refresh, really we only want to refresh the
            // Environment, and there are some toxic listeners (like the
            // LoggingApplicationListener) that affect global static state, so we need a
            // way to switch those off.
            builderApplication
                    .setListeners(filterListeners(builderApplication.getListeners()));
        }
        List<Class<?>> sources = new ArrayList<>();
        for (String name : names) {
            Class<?> cls = ClassUtils.resolveClassName(name, null);
            try {
                cls.getDeclaredAnnotations();
            }
            catch (Exception e) {
                continue;
            }
            sources.add(cls);
        }
        AnnotationAwareOrderComparator.sort(sources);
        builder.sources(sources.toArray(new Class[sources.size()]));
        final ConfigurableApplicationContext context = builder.run();
        // gh-214 using spring.application.name=bootstrap to set the context id via
        // `ContextIdApplicationContextInitializer` prevents apps from getting the actual
        // spring.application.name
        // during the bootstrap phase.
        context.setId("bootstrap");
        // Make the bootstrap context a parent of the app context
        addAncestorInitializer(application, context);
        // It only has properties in it now that we don't want in the parent so remove
        // it (and it will be added back later)
        bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
        mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
        return context;
    }

```



这里主要做的事情有两点

1.  创建Environment对象，设置spring.config.name=bootstrap
2.  创建一个新的SpringApplication启动器，并且设置sources为扩展点下的BootstrapConfiguration配置类



```
List<String> names = new ArrayList<>(SpringFactoriesLoader
                .loadFactoryNames(BootstrapConfiguration.class, classLoader));

```



简单的就是可以理解为创建了一个新的启动器，启动类为BootstrapConfiguration配置的类。并且通过run方法得到了初始化后的BeanFactory容器并且将contenxt封装成AncestorInitializer加入到了我们自己的SpringApplication中



```
application.addInitializers(new AncestorInitializer(context));

```



而在`apply`方法中,获取了`ApplicationContextInitializer`类型的所有对象加入到了我们当前的`SpringApplication`中



```
    private void apply(ConfigurableApplicationContext context,
            SpringApplication application, ConfigurableEnvironment environment) {
        @SuppressWarnings("rawtypes")
        List<ApplicationContextInitializer> initializers = getOrderedBeansOfType(context,
                ApplicationContextInitializer.class);
        application.addInitializers(initializers
                .toArray(new ApplicationContextInitializer[initializers.size()]));
        addBootstrapDecryptInitializer(application);
    }

```



在`spring-cloud-context`包中，`spring.factories`配置下面这个类



```
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration,\

```



到这里我们启动类`SpringApplication`中已经有了`AncestorInitializer,PropertySourceBootstrapConfiguration`两个ApplicationContextInitializer

返回到我们启动类



```
    private void prepareContext(ConfigurableApplicationContext context,
            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
            ApplicationArguments applicationArguments, Banner printedBanner) {
        context.setEnvironment(environment);
        postProcessApplicationContext(context);
        applyInitializers(context);
        listeners.contextPrepared(context);
        if (this.logStartupInfo) {
            logStartupInfo(context.getParent() == null);
            logStartupProfileInfo(context);
        }

        // Add boot specific singleton beans
        context.getBeanFactory().registerSingleton("springApplicationArguments",
                applicationArguments);
        if (printedBanner != null) {
            context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
        }

        // Load the sources
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        load(context, sources.toArray(new Object[0]));
        listeners.contextLoaded(context);
    }

```





```
    protected void applyInitializers(ConfigurableApplicationContext context) {
        for (ApplicationContextInitializer initializer : getInitializers()) {
            Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                    initializer.getClass(), ApplicationContextInitializer.class);
            Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
            initializer.initialize(context);
        }
    }

```



在这里就会调用到`ApplicationContextInitializer.initialize`方法
就会先来到`AncestorInitializer`



```
        @Override
        public void initialize(ConfigurableApplicationContext context) {
            while (context.getParent() != null && context.getParent() != context) {
                context = (ConfigurableApplicationContext) context.getParent();
            }
            reorderSources(context.getEnvironment());
            new ParentContextApplicationContextInitializer(this.parent)
                    .initialize(context);
        }

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        if (applicationContext != this.parent) {
            applicationContext.setParent(this.parent);
            applicationContext.addApplicationListener(EventPublisher.INSTANCE);
        }
    }

```



可以看到这里将BootStrap创建的容器作为我们当前容器的父容器，并且父容器中的对象都初始化好了，`PropertySourceBootstrapConfiguration`的也初始化好了,并且该`ApplicationContextInitializer`加入到了我们自己的启动类里面，因此会调用初始化好了的`PropertySourceBootstrapConfiguration.initialize`,而`PropertySourceLocator`是注入进来的



```
# Bootstrap components
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.config.client.ConfigServiceBootstrapConfiguration,\

```





```
    @Bean
    @ConditionalOnMissingBean(ConfigServicePropertySourceLocator.class)
    @ConditionalOnProperty(value = "spring.cloud.config.enabled", matchIfMissing = true)
    public ConfigServicePropertySourceLocator configServicePropertySource(ConfigClientProperties properties) {
        ConfigServicePropertySourceLocator locator = new ConfigServicePropertySourceLocator(
                properties);
        return locator;
    }

```





```
    @Autowired(required = false)
    private List<PropertySourceLocator> propertySourceLocators = new ArrayList<>();

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        CompositePropertySource composite = new CompositePropertySource(
                BOOTSTRAP_PROPERTY_SOURCE_NAME);
        AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
        boolean empty = true;
        ConfigurableEnvironment environment = applicationContext.getEnvironment();
        for (PropertySourceLocator locator : this.propertySourceLocators) {
            PropertySource<?> source = null;
            source = locator.locate(environment);
            if (source == null) {
                continue;
            }
            logger.info("Located property source: " + source);
            composite.addPropertySource(source);
            empty = false;
        }
        if (!empty) {
            MutablePropertySources propertySources = environment.getPropertySources();
            String logConfig = environment.resolvePlaceholders("${logging.config:}");
            LogFile logFile = LogFile.get(environment);
            if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
                propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
            }
            insertPropertySources(propertySources, composite);
            reinitializeLoggingSystem(environment, logConfig, logFile);
            setLogLevels(applicationContext, environment);
            handleIncludedProfiles(environment);
        }
    }

```



因此最终会来到`ConfigServicePropertySourceLocator`中



```
    private Environment getRemoteEnvironment(RestTemplate restTemplate,
            ConfigClientProperties properties, String label, String state) {
        String path = "/{name}/{profile}";
        String name = properties.getName();
        String profile = properties.getProfile();
        String token = properties.getToken();
        int noOfUrls = properties.getUri().length;
        if (noOfUrls > 1) {
            logger.info("Multiple Config Server Urls found listed.");
        }

        Object[] args = new String[] { name, profile };
        if (StringUtils.hasText(label)) {
            if (label.contains("/")) {
                label = label.replace("/", "(_)");
            }
            args = new String[] { name, profile, label };
            path = path + "/{label}";
        }
        ResponseEntity<Environment> response = null;

        for (int i = 0; i < noOfUrls; i++) {
            Credentials credentials = properties.getCredentials(i);
            String uri = credentials.getUri();
            String username = credentials.getUsername();
            String password = credentials.getPassword();

            logger.info("Fetching config from server at : " + uri);

            try {
                HttpHeaders headers = new HttpHeaders();
                addAuthorizationToken(properties, headers, username, password);
                if (StringUtils.hasText(token)) {
                    headers.add(TOKEN_HEADER, token);
                }
                if (StringUtils.hasText(state) && properties.isSendState()) {
                    headers.add(STATE_HEADER, state);
                }

                final HttpEntity<Void> entity = new HttpEntity<>((Void) null, headers);
                response = restTemplate.exchange(uri + path, HttpMethod.GET, entity,
                        Environment.class, args);
            }
            catch (HttpClientErrorException e) {
                if (e.getStatusCode() != HttpStatus.NOT_FOUND) {
                    throw e;
                }
            }
            catch (ResourceAccessException e) {
                logger.info("Connect Timeout Exception on Url - " + uri
                        + ". Will be trying the next url if available");
                if (i == noOfUrls - 1)
                    throw e;
                else
                    continue;
            }

            if (response == null || response.getStatusCode() != HttpStatus.OK) {
                return null;
            }

            Environment result = response.getBody();
            return result;
        }

        return null;
    }

```



可以看到最后就是调用服务端的接口获取最新的配置。
而这个外部配置最后会放到systemEnvironment之前，因此就会覆盖本地配置，但是可以通过参数控制



```
@ConfigurationProperties("spring.cloud.config")
public class PropertySourceBootstrapProperties {

    /**
     * Flag to indicate that the external properties should override system properties.
     * Default true.
     */
    private boolean overrideSystemProperties = true;

    /**
     * Flag to indicate that {@link #isOverrideSystemProperties()
     * systemPropertiesOverride} can be used. Set to false to prevent users from changing
     * the default accidentally. Default true.
     */
    private boolean allowOverride = true;

    /**
     * Flag to indicate that when {@link #setAllowOverride(boolean) allowOverride} is
     * true, external properties should take lowest priority, and not override any
     * existing property sources (including local config files). Default false.
     */
    private boolean overrideNone = false;

```





```
if (!remoteProperties.isAllowOverride() || (!remoteProperties.isOverrideNone()
                && remoteProperties.isOverrideSystemProperties())) {
            propertySources.addFirst(composite);
            return;
        }
        if (remoteProperties.isOverrideNone()) {
            propertySources.addLast(composite);
            return;
        }
        if (propertySources
                .contains(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME)) {
            if (!remoteProperties.isOverrideSystemProperties()) {
                propertySources.addAfter(
                        StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                        composite);
            }
            else {
                propertySources.addBefore(
                        StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
                        composite);
            }
        }
        else {
            propertySources.addLast(composite);
        }

```



**总结**

1.  扩展点
    1.1 `EnvironmentPostProcessor`
    对环境做处理
    1.2 `PropertySourceLoader`
    解析不同的格式的文件
    1.2 `ApplicationListener`
    spring-cloud包中通过新增扩展BootstrapApplicationListener
    1.3 `BootstrapConfiguration`
    通过扩展点加载启动类

2.  文件加载过程
    通过ApplicationListener事件创建一个新的SpringApliction启动类将BootstrapConfiguration扩展类作为启动配置类，然后获得了一个初始化了的BootStrap容器，将该容器封装成AncestorInitializer类加入到我们自己的启动类中，作用就是将我们自己容器的父容器设置为BootStrap容器。通过BootStrap容器获得初始化好后的ApplicationContextInitializer类型对象，而该类来自于PropertySourceBootstrapConfiguration启动配置类，该类中又注入了PropertySourceLocator类，而在ConfigServiceBootstrapConfiguration启动配置类中配置bean对象
    ConfigServicePropertySourceLocator，所以最终会将BootStrap中初始好的PropertySourceBootstrapConfiguration加入到我们自己的启动类中调用，最终调用initialize方法然后调用ConfigServicePropertySourceLocator.locate方法去Config server服务获取配置。



作者：拥抱孤独_to
链接：https://www.jianshu.com/p/60c6ab0e79d5
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。