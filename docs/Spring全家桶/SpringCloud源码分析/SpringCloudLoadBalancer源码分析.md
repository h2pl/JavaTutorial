# Spring Cloud LoadBalancer

## 概述

> Spring Cloud LoadBalancer目前Spring官方是放在spring-cloud-commons里，Spring Cloud最新版本为2021.0.2
>
> [Spring Cloud LoadBalancer 官网文档地址](https://docs.spring.io/spring-cloud-commons/docs/3.1.2/reference/html/#spring-cloud-loadbalancer) [https://docs.spring.io/spring-cloud-commons/docs/3.1.2/reference/html/#spring-cloud-loadbalancer](https://docs.spring.io/spring-cloud-commons/docs/3.1.2/reference/html/#spring-cloud-loadbalancer)
>
> [Spring Cloud官网文档地址](https://docs.spring.io/spring-cloud/docs/current/reference/html/) [https://docs.spring.io/spring-cloud/docs/current/reference/html/](https://docs.spring.io/spring-cloud/docs/current/reference/html/)

一方面Netflix Ribbon停止更新，Spring Cloud LoadBalancer是Spring Cloud官方自己提供的客户端负载均衡器,抽象和实现，用来替代Ribbon。

*   常见负载均衡器分为服务端负载均衡器(如网关层均衡负载)和客户端层均衡负载。
    *   网关层如硬件层面的F5或软件层面的LVS、或者nginx等。
    *   客户端层就如Spring Cloud LoadBalancer，作为一个客户端去发现更新维护服务列表，自定义服务的均衡负载策略（随机、轮询、小流量的金丝雀等等）。

Spring Cloud提供了自己的客户端负载平衡器抽象和实现。对于负载均衡机制，增加了ReactiveLoadBalancer接口，并提供了基于round-robin轮询和Random随机的实现。为了从响应式ServiceInstanceListSupplier中选择实例，需要使用ServiceInstanceListSupplier。目前支持ServiceInstanceListSupplier的基于服务发现的实现，该实现使用类路径中的发现客户端从Service Discovery中检索可用的实例。

可以通过如下配置来禁用Spring Cloud LoadBalance

```
spring:
  cloud:
    loadbalancer:
      enabled: false

```

## 入门示例

前面simple-ecommerce项目创建已在父Pom引入三大父依赖，详细可以看下前面的文章<<SpringCloudAlibaba注册中心与配置中心之利器Nacos实战与源码分析>>，其中Spring Cloud的版本为2021.0.1，前面文章也已说过，Spring Cloud Alibaba整合在spring-cloud-starter-alibaba-nacos-discovery本身就依赖spring-cloud-loadbalancer。

> 注意如果是Hoxton之前的版本，默认负载均衡器为Ribbon，需要移除Ribbon引用和增加配置spring.cloud.loadbalancer.ribbon.enabled: false。

如果是在Spring Boot项目中添加下面的启动器依赖，该starter也包含了Spring Boot Caching and Evictor.

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            spring-cloud-starter-loadbalancer
        </dependency>

```

我们使用Spring官方提供了负载均衡的客户端之一RestTemplate，RestTemplate是Spring提供的用于访问Rest服务的客户端，RestTemplate提供了多种便捷访问远程Http服务的方法，能够大大提高客户端的编写效率。默认情况下，RestTemplate默认依赖jdk的HTTP连接工具。创建RestTemplateConfig配置类，标注 @LoadBalanced注解，默认使用的ReactiveLoadBalancer实现是RoundRobinLoadBalancer。

```
package cn.itxs.ecom.order.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {
    @LoadBalanced
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate() ;
    }
}

```

订单微服务中订单控制器增加deductRest方法

```
package cn.itxs.ecom.order.controller;

import cn.itxs.ecom.commons.service.OrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

/**
 * @Name ：OrderController
 * @Description ：订单控制器
 * @Author ：itxs
 * @Date ：2022/4/10 20:15
 * @Version ：1.0
 * @History ：
 */
@RestController
public class OrderController {

    @Autowired
    OrderService orderService;

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("/create/{userId}/{commodityCode}/{count}")
    public String create(@PathVariable("userId") String userId,@PathVariable("commodityCode") String commodityCode, @PathVariable("count") int count){
        return orderService.create(userId,commodityCode,count).toString();
    }

    @RequestMapping("/deductRest/{commodityCode}/{count}")
    public String deductRest(@PathVariable("commodityCode") String commodityCode, @PathVariable("count") int count){
        String url = "http://ecom-storage-service/deduct/"+commodityCode+"/"+count;
        return restTemplate.getForObject(url, String.class);
    }
}

```

前面server.port我们是放在Nacos配置中心里，这里我们注释Nacos配置中心的配置放在本地配置文件bootstrap.yml里，分别配置为4080、4081、4082启动3个库存服务实例，并启动订单微服务

```
server:
  port: 4080

```

![image-20220505191143684](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/938570a0b63b56671a862e8bda11577a.png)

查看nacos服务管理-服务列表里服务详情，可以看到3个健康的库存实例和1个订单微服务实例

![image-20220505182432182](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/33fa03d937354fe00bb0bb2f3dd5c805.png)

访问6次订单dedect接口：[http://localhost:4070/deductRest/1001/1](http://localhost:4070/deductRest/1001/1) ，从测试的结果也验证了LoadBalancer默认是轮询负载均衡策略。

![image-20220505192217715](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1fc20b1d482d2a4e3924c5707ac2ff19.png)

## 负载均衡算法切换

创建自定义负载均衡配置类CustomLoadBalancerConfiguration

```
package cn.itxs.ecom.order.config;

import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.core.RandomLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ReactorLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.core.env.Environment;

public class CustomLoadBalancerConfiguration {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
                                                            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(loadBalancerClientFactory
                .getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}

```

RestTemplateConfig配置类LoadBalancerClient指定随机的配置类，value的值为提供者也即是库存微服务名称。

```
package cn.itxs.ecom.order.config;

import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
@LoadBalancerClient(value = "ecom-storage-service", configuration = CustomLoadBalancerConfiguration.class)
public class RestTemplateConfig {

    @LoadBalanced
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build() ;
    }
}

```

多次访问订单dedect接口测试确认已切换为随机负载均衡策略。

## 集成方式

官网提供3中集成Spring Cloud LoadBalancer的方式，除了第一种上面已使用过，还支持Spring Web Flux响应式编程，WebClient是从Spring WebFlux 5.0版本开始提供的一个非阻塞的基于响应式编程的进行Http请求的客户端工具。它的响应式编程的基于Reactor的。WebClient中提供了标准Http请求方式对应的get、post、put、delete等方法，可以用来发起相应的请求。

![image-20220506233248585](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/ad44f363342f540b1cb1805b20be6ef0.png)

在订单微服务中引入spring-boot-starter-webflux依赖

```
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-webflux
        </dependency>

```

订单微服务中增加WebClientConfig配置类

```
package cn.itxs.ecom.order.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.client.WebClient;

@Configuration
public class WebClientConfig {
    @LoadBalanced
    @Bean
    WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }

    @Bean
    WebClient webClient() {
        return webClientBuilder().build();
    }
}

```

订单微服务订单控制器中添加WebClient接口实现，代码如下

```
    @Autowired
    private WebClient webClient;   

	@RequestMapping(value = "/deductWebClient/{commodityCode}/{count}")
    public Mono<String> deductWebClient(@PathVariable("commodityCode") String commodityCode, @PathVariable("count") int count) {
        String url = "http://ecom-storage-service/deduct/"+commodityCode+"/"+count;
        // 基于WebClient
        Mono<String> result = webClient.get().uri(url)
                .retrieve().bodyToMono(String.class);
        return result;
    }

```

重新启动订单微服务

![image-20220506234934948](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/ec48e022accfb3298069c96b7e3799e7.png)

访问订单控制器中的减库存WebClient接口，[http://localhost:4070/deductWebClient/1001/1](http://localhost:4070/deductWebClient/1001/1) ，结果返回成功

![image-20220506234627330](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/5f737fc9d8806dc072ce3e1fdf983de6.png)

我们还可以配置基于过滤器的方式，通过WebClient使用ReactiveLoadBalancer。如果项目中添加了Spring Cloud LoadBalancer starter，并且Spring -webflux在类路径中，ReactorLoadBalancerExchangeFilterFunction则是自动配置的。

订单微服务订单控制器中添加WebClient使用ReactiveLoadBalancer接口实现，代码如下

```
    @Autowired
    private ReactorLoadBalancerExchangeFilterFunction lbFunction;   

    @RequestMapping(value = "/deductWebFluxReactor/{commodityCode}/{count}")
    public Mono<String> deductWebFluxReactor(@PathVariable("commodityCode") String commodityCode, @PathVariable("count") int count) {
        String url = "/deduct/"+commodityCode+"/"+count;
        Mono<String> result = WebClient.builder().baseUrl("http://ecom-storage-service")
                .filter(lbFunction)
                .build()
                .get()
                .uri(url)
                .retrieve()
                .bodyToMono(String.class);
        return result;
    }

```

重新启动订单微服务

![image-20220507000930179](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/3546b9913f1253a1b23039ad6f7546d1.png)

访问订单控制器中的减库存WebClient接口，[http://localhost:4070/deductWebFluxReactor/1001/1](http://localhost:4070/deductWebFluxReactor/1001/1) ，结果返回成功

![image-20220507000746900](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0bc2b04d23d11bc85009c7e040900b2b.png)

关于LoadBalancer官网还提供很多其他功能，有兴趣可自行详细查阅和动手实验

![image-20220507001132987](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1d8f35933945b317a2dd71627fdfa748.png)

## 原理

### RestTemplate

Spring Cloud LoadBalancer源码分析我们先从RestTemplate负载均衡的简单实现来分析入手，除此之外其支持Spring Web Flux响应式编程的实现原理思想也是相同，都是通过客户端添加拦截器，在拦截器中实现负载均衡。从RestTemplate的源码中可以知道其继承自InterceptingHttpAccessor抽象类

![image-20220508142236428](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c3ee7af69f54aeed5f1026823779fee8.png)

而InterceptingHttpAccessor抽象类则提供了一个方法setInterceptors，用于设置拦截器，拦截器需要实现ClientHttpRequestInterceptor接口即可，在实际远程请求服务端接口之前会先调用拦截器的intercept方法。这里的拦截器相当于Servlet技术中的Filter功能

```
	// 代码实现在抽象父类InterceptingHttpAccessor里
	// RestTemplate.InterceptingHttpAccessor#setInterceptors
	public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
		Assert.noNullElements(interceptors, "'interceptors' must not contain null elements");
		// Take getInterceptors() List as-is when passed in here
		if (this.interceptors != interceptors) {
			this.interceptors.clear();
			this.interceptors.addAll(interceptors);
			AnnotationAwareOrderComparator.sort(this.interceptors);
		}
	}

```

![image-20220508142443637](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/5062f793bb2bd7f996eb944f77af7137.png)

### **LoadBalancerAutoConfiguration**

从官网可以知道Spring Cloud LoadBalancer放在spring-cloud-commons，因此也作为其核心的@LoadBalanced注解也就是由spring-cloud-commons来实现，依据SpringBoot自动装配的原理先查看依赖包的实现逻辑，不难发现spring-cloud-commons引入了自动配置类LoadBalancerAutoConfiguration和ReactorLoadBalancerClientAutoConfiguration。

![image-20220509001530634](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b5be12e0dd8d515aa07613738efab122.png)

当满足上述的条件时（@Conditional为条件注解），将自动创建LoadBalancerInterceptor并注入到RestTemplate中。

![image-20220508143752218](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/676f6d81529228511c862c780b0c4532.png)

### **LoadBalancerLnterceptor**

LoadBalancerInterceptor实现了ClientHttpRequestInterceptor接口，因此也实现intercept方法，用于实现负载均衡的拦截处理。

![image-20220508144048248](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/bb587d8122fee4973eba2fc00ed1af3f.png)

### **LoadBalancerClient**

LoadBalancerClient用于进行负载均衡逻辑，继承自ServiceInstanceChooser接口，从服务列表中选择出一个服务地址进行调用。在LoadBalancerClient种存在两个execute()方法，均是用来执行请求的，reconstructURI()是用来重构URL。

![image-20220508144435104](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/a3590c0ab24cb5ee04cf1cf513724fc0.png)

对于LoadBalancerClient接口Spring Cloud LoadBalancer的提供默认实现为BlockingLoadBalancerClient

![image-20220508144750601](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/256e4815a982966d7d6bfb82e0e97f67.png)

```
@SuppressWarnings({ "unchecked", "rawtypes" })
public class BlockingLoadBalancerClient implements LoadBalancerClient {

	private final ReactiveLoadBalancer.Factory<ServiceInstance> loadBalancerClientFactory;

	/**
	 * @deprecated in favour of
	 * {@link BlockingLoadBalancerClient#BlockingLoadBalancerClient(ReactiveLoadBalancer.Factory)}
	 */
	@Deprecated
	public BlockingLoadBalancerClient(LoadBalancerClientFactory loadBalancerClientFactory,
			LoadBalancerProperties properties) {
		this.loadBalancerClientFactory = loadBalancerClientFactory;
	}

	public BlockingLoadBalancerClient(ReactiveLoadBalancer.Factory<ServiceInstance> loadBalancerClientFactory) {
		this.loadBalancerClientFactory = loadBalancerClientFactory;
	}

	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
		String hint = getHint(serviceId);
		LoadBalancerRequestAdapter<T, DefaultRequestContext> lbRequest = new LoadBalancerRequestAdapter<>(request,
				new DefaultRequestContext(request, hint));
		Set<LoadBalancerLifecycle> supportedLifecycleProcessors = getSupportedLifecycleProcessors(serviceId);
		supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onStart(lbRequest));
		ServiceInstance serviceInstance = choose(serviceId, lbRequest);
        // 选择服务
		if (serviceInstance == null) {
			supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onComplete(
					new CompletionContext<>(CompletionContext.Status.DISCARD, lbRequest, new EmptyResponse())));
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		return execute(serviceId, serviceInstance, lbRequest);
	}

	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request)
			throws IOException {
		DefaultResponse defaultResponse = new DefaultResponse(serviceInstance);
		Set<LoadBalancerLifecycle> supportedLifecycleProcessors = getSupportedLifecycleProcessors(serviceId);
		Request lbRequest = request instanceof Request ? (Request) request : new DefaultRequest<>();
		supportedLifecycleProcessors
				.forEach(lifecycle -> lifecycle.onStartRequest(lbRequest, new DefaultResponse(serviceInstance)));
		try {
			T response = request.apply(serviceInstance);
			Object clientResponse = getClientResponse(response);
			supportedLifecycleProcessors
					.forEach(lifecycle -> lifecycle.onComplete(new CompletionContext<>(CompletionContext.Status.SUCCESS,
							lbRequest, defaultResponse, clientResponse)));
			return response;
		}
		catch (IOException iOException) {
			supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onComplete(
					new CompletionContext<>(CompletionContext.Status.FAILED, iOException, lbRequest, defaultResponse)));
			throw iOException;
		}
		catch (Exception exception) {
			supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onComplete(
					new CompletionContext<>(CompletionContext.Status.FAILED, exception, lbRequest, defaultResponse)));
			ReflectionUtils.rethrowRuntimeException(exception);
		}
		return null;
	}

	private <T> Object getClientResponse(T response) {
		ClientHttpResponse clientHttpResponse = null;
		if (response instanceof ClientHttpResponse) {
			clientHttpResponse = (ClientHttpResponse) response;
		}
		if (clientHttpResponse != null) {
			try {
				return new ResponseData(clientHttpResponse, null);
			}
			catch (IOException ignored) {
			}
		}
		return response;
	}

	private Set<LoadBalancerLifecycle> getSupportedLifecycleProcessors(String serviceId) {
		return LoadBalancerLifecycleValidator.getSupportedLifecycleProcessors(
				loadBalancerClientFactory.getInstances(serviceId, LoadBalancerLifecycle.class),
				DefaultRequestContext.class, Object.class, ServiceInstance.class);
	}

	@Override
	public URI reconstructURI(ServiceInstance serviceInstance, URI original) {
		return LoadBalancerUriTools.reconstructURI(serviceInstance, original);
	}

	@Override
	public ServiceInstance choose(String serviceId) {
		return choose(serviceId, REQUEST);
	}

    // 通过不同的负载均衡客户端实现选择不同的服务
	@Override
	public <T> ServiceInstance choose(String serviceId, Request<T> request) {
		ReactiveLoadBalancer<ServiceInstance> loadBalancer = loadBalancerClientFactory.getInstance(serviceId);
		if (loadBalancer == null) {
			return null;
		}
		Response<ServiceInstance> loadBalancerResponse = Mono.from(loadBalancer.choose(request)).block();
		if (loadBalancerResponse == null) {
			return null;
		}
		return loadBalancerResponse.getServer();
	}

	private String getHint(String serviceId) {
		LoadBalancerProperties properties = loadBalancerClientFactory.getProperties(serviceId);
		String defaultHint = properties.getHint().getOrDefault("default", "default");
		String hintPropertyValue = properties.getHint().get(serviceId);
		return hintPropertyValue != null ? hintPropertyValue : defaultHint;
	}

}

```

### **LoadBalancerClientFactory**

BlockingLoadBalancerClient中持有LoadBalancerClientFactory通过调用其getInstance方法获取具体的负载均衡客户端。通过工厂类LoadBalancerClientFactory获取具体的负载均衡器实例，后面的loadBalancer.choose(request)调用其接口choose()方法实现根据负载均衡算法选择下一个服务器完成负载均衡，而ReactiveLoadBalancer<t> getInstance(String serviceId) 有默认实现LoadBalancerClientFactory
![image-20220508190132565](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4014445b0c2ea7189a232b5d257fb938.png)</t>

LoadBalancerClientFactory客户端实现了不同的负载均衡算法，比如轮询、随机等。LoadBalancerClientFactory继承自NamedContextFactory，NamedContextFactory继承ApplicationContextAware，实现Spring ApplicationContext容器操作。

![image-20220508190412076](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/18f62338fdb636327fcdb2d08d33661b.png)

### ReactiveLoadBalancer

ReactiveLoadBalancer负载均衡器实现服务选择，Spring Cloud Balancer中实现了轮询RoundRobinLoadBalancer、随机RandomLoadBalancer、NacosLoadBalancer算法。

![image-20220508235128931](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f19b9fd0d246dbae37b05e67b1a79805.png)

### LoadBalancerClientConfiguration

如果没有显式指定负载均衡算法，默认缺省值为RoundRobinLoadBalancer

```
	@Bean
	@ConditionalOnMissingBean
	public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(Environment environment,
			LoadBalancerClientFactory loadBalancerClientFactory) {
		String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
		return new RoundRobinLoadBalancer(
				loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
	}

```

![image-20220508235645313](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c8ae46d4cedb33f40939edb4f6fde542.png)

### **LoadBalancerRequestFactory**

LoadBalancerRequest工厂类调用createRequest方法用于创建LoadBalancerRequest。其内部持有LoadBalancerClient对象也即持有BlockingLoadBalancerClient。

![image-20220509000049541](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c928dcb312df87a44c33ee77a296ee84.png)

在日常项目中，一般负载均衡都是结合Feign使用，后续我们有时间再来分析Feign整合LoadBalancer的自动配置类FeignLoadBalancerAutoConfiguration的实现

### ReactorLoadBalancerClientAutoConfiguration

我们也抛一下基于WebClient的@Loadbalanced的流程的引入，首先声明负载均衡过滤器ReactorLoadBalancerClientAutoConfiguration是一个自动装配器类，在项目中引入了 WebClient 和 ReactiveLoadBalancer 类之后，自动装配流程就开始运行，它会初始化一个实现了 ExchangeFilterFunction 的实例，在后面该实例将作为过滤器被注入到WebClient。后续流程有兴趣再自行研究

![image-20220509001650781](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/869a4ea9a6e1409b84c927db08cebea1.png)

## 自定义负载均衡器

从上面可以知道LoadBalancerClientFactory是创建客户机、负载均衡器和客户机配置实例的工厂。它根据客户端名称创建一个Spring ApplicationContext，并从中提取所需的bean。因此进入到LoadBalancerClientFactory类中，需要去实现它的子接口ReactorServiceInstanceLoadBalancer，因为去获取负载均衡器实例的时候，是通过去容器中查找ReactorServiceInstanceLoadBalancer类型的bean来实现的，可以参照RandomLoadBalancer实现代码

```
package org.springframework.cloud.loadbalancer.core;

import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import reactor.core.publisher.Mono;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.DefaultResponse;
import org.springframework.cloud.client.loadbalancer.EmptyResponse;
import org.springframework.cloud.client.loadbalancer.Request;
import org.springframework.cloud.client.loadbalancer.Response;

public class RandomLoadBalancer implements ReactorServiceInstanceLoadBalancer {

	private static final Log log = LogFactory.getLog(RandomLoadBalancer.class);

	private final String serviceId;

	private ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

	/**
	 * @param serviceInstanceListSupplierProvider a provider of
	 * {@link ServiceInstanceListSupplier} that will be used to get available instances
	 * @param serviceId id of the service for which to choose an instance
	 */
	public RandomLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
			String serviceId) {
		this.serviceId = serviceId;
		this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
	}

	@SuppressWarnings("rawtypes")
	@Override
	public Mono<Response<ServiceInstance>> choose(Request request) {
		ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
				.getIfAvailable(NoopServiceInstanceListSupplier::new);
		return supplier.get(request).next()
				.map(serviceInstances -> processInstanceResponse(supplier, serviceInstances));
	}

	private Response<ServiceInstance> processInstanceResponse(ServiceInstanceListSupplier supplier,
			List<ServiceInstance> serviceInstances) {
		Response<ServiceInstance> serviceInstanceResponse = getInstanceResponse(serviceInstances);
		if (supplier instanceof SelectedInstanceCallback && serviceInstanceResponse.hasServer()) {
			((SelectedInstanceCallback) supplier).selectedServiceInstance(serviceInstanceResponse.getServer());
		}
		return serviceInstanceResponse;
	}

	private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
		if (instances.isEmpty()) {
			if (log.isWarnEnabled()) {
				log.warn("No servers available for service: " + serviceId);
			}
			return new EmptyResponse();
		}
		int index = ThreadLocalRandom.current().nextInt(instances.size());

		ServiceInstance instance = instances.get(index);

		return new DefaultResponse(instance);
	}

}

```

保留核心实现进行简单仿写如下

```
package cn.itxs.ecom.order.config;

import java.util.List;
import java.util.Random;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.client.loadbalancer.DefaultResponse;
import org.springframework.cloud.client.loadbalancer.EmptyResponse;
import org.springframework.cloud.client.loadbalancer.Request;
import org.springframework.cloud.client.loadbalancer.Response;
import reactor.core.publisher.Mono;

public class ItxsRandomLoadBalancerClient implements ReactorServiceInstanceLoadBalancer {
    // 服务列表
    private ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    public ItxsRandomLoadBalancerClient(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider) {
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider.getIfAvailable();
        return supplier.get().next().map(this::getInstanceResponse);
    }

    /**
     * 使用随机数获取服务
     * @param instances
     * @return
     */
    private Response<ServiceInstance> getInstanceResponse(
            List<ServiceInstance> instances) {
        System.out.println("ItxsRandomLoadBalancerClient start");
        if (instances.isEmpty()) {
            return new EmptyResponse();
        }

        System.out.println("ItxsRandomLoadBalancerClient random");
        // 随机算法
        int size = instances.size();
        Random random = new Random();
        ServiceInstance instance = instances.get(random.nextInt(size));

        return new DefaultResponse(instance);
    }
}

```

将上面CustomLoadBalancerConfiguration替换为如下内容

```
package cn.itxs.ecom.order.config;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.context.annotation.Bean;

public class CustomLoadBalancerConfiguration {

    @Bean
    public ReactorServiceInstanceLoadBalancer customLoadBalancer(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider) {
        return new ItxsRandomLoadBalancerClient(serviceInstanceListSupplierProvider);
    }
}

```

启动库存微服务和订单微服务，访问http://localhost:4070/deductRest/1001/1 ，控制台已打印自定义ItxsRandomLoadBalancerClient中的日志和成功访问结果

![image-20220509003807968](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c72e3f02f7e0d5d3343f8ae9c464b69c.png)

![image-20220509003927550](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/59796bbbd6b3524e32759d42b622f1bc.png)

# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning