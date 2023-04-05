# 目录
* [前言](#前言)
* [源码分析](#源码分析)
* [实例](#实例)
* [资源文件映射](#资源文件映射)
* [总结](#总结)


本文转载自互联网，侵删

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Spring和SpringMVC源码分析》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从spring基础入手，一步步地学习spring基础和springmvc的框架知识，并上手进行项目实战，spring框架是每一个Java工程师必须要学习和理解的知识点，进一步来说，你还需要掌握spring甚至是springmvc的源码以及实现原理，才能更完整地了解整个spring技术体系，形成自己的知识框架。

后续还会有springboot和springcloud的技术专题，陆续为大家带来，敬请期待。

为了更好地总结和检验你的学习成果，本系列文章也会提供部分知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->  
## 前言
SpringMVC是目前主流的Web MVC框架之一。

我们使用浏览器通过地址 http://ip:port/contextPath/path 进行访问，SpringMVC是如何得知用户到底是访问哪个Controller中的方法，这期间到底发生了什么。

本文将分析SpringMVC是如何处理请求与Controller之间的映射关系的，让读者知道这个过程中到底发生了什么事情。

本文实际上是在上文基础上，深入分析

HandlerMapping里的

HandlerExecutionChain getHandler(HttpServletRequest var1) throws Exception;

该方法的具体实现，包括它如何找到对应的方法，以及如何把结果保存在map里，以便让请求转发到对应的handler上，同时也分析了handleradaptor具体做了什么事情。

## 源码分析

在分析源码之前，我们先了解一下几个东西。

1.这个过程中重要的接口和类。

**HandlerMethod类：**

Spring3.1版本之后引入的。 是一个封装了方法参数、方法注解，方法返回值等众多元素的类。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/232033308878895.jpg)

它的子类InvocableHandlerMethod有两个重要的属性WebDataBinderFactory和HandlerMethodArgumentResolverComposite， 很明显是对请求进行处理的。

InvocableHandlerMethod的子类ServletInvocableHandlerMethod有个重要的属性HandlerMethodReturnValueHandlerComposite，很明显是对响应进行处理的。

ServletInvocableHandlerMethod这个类在HandlerAdapter对每个请求处理过程中，都会实例化一个出来(上面提到的属性由HandlerAdapter进行设置)，分别对请求和返回进行处理。　　(RequestMappingHandlerAdapter源码，实例化ServletInvocableHandlerMethod的时候分别set了上面提到的重要属性)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/240149411377243.png)

**MethodParameter类：**

HandlerMethod类中的parameters属性类型，是一个MethodParameter数组。MethodParameter是一个封装了方法参数具体信息的工具类，包括参数的的索引位置，类型，注解，参数名等信息。

HandlerMethod在实例化的时候，构造函数中会初始化这个数组，这时只初始化了部分数据，在HandlerAdapter对请求处理过程中会完善其他属性，之后交予合适的HandlerMethodArgumentResolver接口处理。

以类DeptController为例：

```  
@Controller  
@RequestMapping(value = "/dept")  
public class DeptController {  
  
  @Autowired  private IDeptService deptService;  
  @RequestMapping("/update")  @ResponseBody  public String update(Dept dept) {    deptService.saveOrUpdate(dept);    return "success";  }  
}  
  
```  

(刚初始化时的数据)　　  
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241246397157212.png)

(HandlerAdapter处理后的数据)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241246574657656.png)

**RequestCondition接口：**

**Spring3.1版本之后引入的。 是SpringMVC的映射基础中的请求条件，可以进行combine, compareTo，getMatchingCondition操作。这个接口是映射匹配的关键接口，其中getMatchingCondition方法关乎是否能找到合适的映射。**

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241429158878034.png)

**RequestMappingInfo类：**

Spring3.1版本之后引入的。 是一个封装了各种请求映射条件并实现了RequestCondition接口的类。

有各种RequestCondition实现类属性，patternsCondition，methodsCondition，paramsCondition，headersCondition，consumesCondition以及producesCondition，这个请求条件看属性名也了解，分别代表http请求的路径模式、方法、参数、头部等信息。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241556162777007.png)

**RequestMappingHandlerMapping类：**

处理请求与HandlerMethod映射关系的一个类。

2.Web服务器启动的时候，SpringMVC到底做了什么。

先看AbstractHandlerMethodMapping的initHandlerMethods方法中。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241831201065104.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241922304028276.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/241932113242502.png)

我们进入createRequestMappingInfo方法看下是如何构造RequestMappingInfo对象的。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/251242523404424.png)

PatternsRequestCondition构造函数：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/252348515124305.png)

类对应的RequestMappingInfo存在的话，跟方法对应的RequestMappingInfo进行combine操作。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/250107154024892.png)

然后使用符合条件的method来注册各种HandlerMethod。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260023477461660.png)

下面我们来看下各种RequestCondition接口的实现类的combine操作。

PatternsRequestCondition：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/251617000436911.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/251917084021029.png)

RequestMethodsRequestCondition：

方法的请求条件，用个set直接add即可。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/251919265597456.png)

其他相关的RequestConditon实现类读者可自行查看源码。

最终，RequestMappingHandlerMapping中两个比较重要的属性

private final Map<T, HandlerMethod> handlerMethods = new LinkedHashMap<T, HandlerMethod>();

private final MultiValueMap<String, T> urlMap = new LinkedMultiValueMap<String, T>();

T为RequestMappingInfo。

构造完成。

我们知道，SpringMVC的分发器DispatcherServlet会根据浏览器的请求地址获得HandlerExecutionChain。

这个过程我们看是如何实现的。

首先看HandlerMethod的获得(直接看关键代码了)：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/252206079491274.png)

这里的比较器是使用RequestMappingInfo的compareTo方法(RequestCondition接口定义的)。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/252219494658063.png)

然后构造HandlerExecutionChain加上拦截器

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/252208290431970.png)

## 实例

写了这么多，来点例子让我们验证一下吧。

```  
@Controller  
@RequestMapping(value = "/wildcard")  
public class TestWildcardController {  
  
  @RequestMapping("/test/**")  @ResponseBody  public String test1(ModelAndView view) {    view.setViewName("/test/test");    view.addObject("attr", "TestWildcardController -> /test/**");    return view;  }  
  @RequestMapping("/test/*")  @ResponseBody  public String test2(ModelAndView view) {    view.setViewName("/test/test");    view.addObject("attr", "TestWildcardController -> /test*");    return view;  }  
  @RequestMapping("test?")  @ResponseBody  public String test3(ModelAndView view) {    view.setViewName("/test/test");    view.addObject("attr", "TestWildcardController -> test?");    return view;  }  
  @RequestMapping("test/*")  @ResponseBody  public String test4(ModelAndView view) {    view.setViewName("/test/test");    view.addObject("attr", "TestWildcardController -> test/*");    return view;  }  
}  
  
```  

由于这里的每个pattern都带了*因此，都不会加入到urlMap中，但是handlerMethods还是有的。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260032509654870.png)

当我们访问：http://localhost:8888/SpringMVCDemo/wildcard/test1的时候。

会先根据 "/wildcard/test1" 找urlMap对应的RequestMappingInfo集合，找不到的话取handlerMethods集合中所有的key集合(也就是RequestMappingInfo集合)。

然后进行匹配，匹配根据RequestCondition的getMatchingCondition方法。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260046407936253.png)

最终匹配到2个RequestMappingInfo：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260049401063106.png)

然后会使用比较器进行排序。

之前也分析过，比较器是有优先级的。

我们看到，RequestMappingInfo除了pattern，其他属性都是一样的。

我们看下PatternsRequestCondition比较的逻辑：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260125371817100.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260125488223078.png)

因此，/test*的通配符比/test?的多，因此，最终选择了/test?

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260129091195087.png)

直接比较优先于通配符。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260129342756267.png)

```  
@Controller  
@RequestMapping(value = "/priority")  
public class TestPriorityController {  
  
  @RequestMapping(method = RequestMethod.GET)  @ResponseBody  public String test1(ModelAndView view) {    view.setViewName("/test/test");    view.addObject("attr", "其他condition相同，带有method属性的优先级高");  
    return view;  }  
  @RequestMapping()  @ResponseBody  public String test2(ModelAndView view) {    view.setViewName("/test/test");    view.addObject("attr", "其他condition相同，不带method属性的优先级高");  
    return view;  }  
}  
  
```  

这里例子，其他requestCondition都一样，只有RequestMethodCondition不一样。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260151358222728.png)

看出，方法多的优先级越多。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/260152065252948.png)

至于其他的RequestCondition，大家自行查看源码吧。

## 资源文件映射

以上分析均是基于Controller方法的映射(RequestMappingHandlerMapping)。

SpringMVC中还有静态文件的映射，SimpleUrlHandlerMapping。

DispatcherServlet找对应的HandlerExecutionChain的时候会遍历属性handlerMappings，这个一个实现了HandlerMapping接口的集合。

由于我们在*-dispatcher.xml中加入了以下配置：

```  
<mvc:resources location="/static/" mapping="/static/**"/>  
```  

Spring解析配置文件会使用ResourcesBeanDefinitionParser进行解析的时候，会实例化出SimpleUrlHandlerMapping。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/261025451501584.jpg)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/261026409781228.jpg)

其中注册的HandlerMethod为ResourceHttpRequestHandler。

访问地址：http://localhost:8888/SpringMVCDemo/static/js/jquery-1.11.0.js

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/261013551199069.jpg)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/261028504315946.jpg)

地址匹配到/static/**。

最终SimpleUrlHandlerMapping找到对应的Handler -> ResourceHttpRequestHandler。

ResourceHttpRequestHandler进行handleRequest的时候，直接输出资源文件的文本内容。

## 总结

大致上整理了一下SpringMVC对请求的处理，包括其中比较关键的类和接口，希望对读者有帮助。

让自己对SpringMVC有了更深入的认识，也为之后分析数据绑定，拦截器、HandlerAdapter等打下基础。