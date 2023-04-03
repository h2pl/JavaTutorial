# Table of Contents

  * [目录](#目录)
  * [前言](#前言)
  * [现象](#现象)
  * [源码分析](#源码分析)
  * [实例讲解](#实例讲解)
  * [关于配置](#关于配置)
  * [总结](#总结)
  * [详解RequestBody和@ResponseBody注解](#详解requestbody和responsebody注解)
  * [参考资料](#参考资料)


转自 [SpringMVC关于json、xml自动转换的原理研究[附带源码分析]](https://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html)

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

## 目录

*   [前言](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#preface)
*   [现象](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#phenomenon)
*   [源码分析](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#analysis)
*   [实例讲解](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#demo)
*   [关于配置](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#config)
*   [总结](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#summary)
*   [参考资料](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html#reference)

## 前言

SpringMVC是目前主流的Web MVC框架之一。 

如果有同学对它不熟悉，那么请参考它的入门blog：[http://www.cnblogs.com/fangjian0423/p/springMVC-introduction.html](http://www.cnblogs.com/fangjian0423/p/springMVC-introduction.html)

## 现象

本文使用的demo基于maven，是根据入门blog的例子继续写下去的。

我们先来看一看对应的现象。 我们这里的配置文件 *-dispatcher.xml中的关键配置如下(其他常规的配置文件不在讲解，可参考本文一开始提到的入门blog)：

(视图配置省略)

```
<mvc:resources location="/static/" mapping="/static/**"/>
<mvc:annotation-driven/>
<context:component-scan base-package="org.format.demo.controller"/>
```

pom中需要有以下依赖(Spring依赖及其他依赖不显示)：

```
<dependency>
  <groupId>org.codehaus.jackson</groupId>
  jackson-core-asl
  <version>1.9.13</version>
</dependency>
<dependency>
  <groupId>org.codehaus.jackson</groupId>
  jackson-mapper-asl
  <version>1.9.13</version>
</dependency>

```

这个依赖是json序列化的依赖。

ok。我们在Controller中添加一个method：


<pre>@RequestMapping("/xmlOrJson")
@ResponseBody public Map <string, object="">xmlOrJson() {
    Map <string, object="">map = new HashMap<string, object="">();
    map.put("list", employeeService.list()); return map;
}</string,></string,></string,></pre>


直接访问地址：

![](https://images0.cnblogs.com/i/411512/201405/101449596675807.png)

我们看到，短短几行配置。使用@ResponseBody注解之后，Controller返回的对象 自动被转换成对应的json数据，在这里不得不感叹SpringMVC的强大。

我们好像也没看到具体的配置，唯一看到的就是*-dispatcher.xml中的一句配置：<mvc:annotation-driven>。其实就是这个配置，导致了java对象自动转换成json对象的现象。</mvc:annotation-driven>

那么spring到底是如何实现java对象到json对象的自动转换的呢？ 为什么转换成了json数据，如果想转换成xml数据，那该怎么办？

## 源码分析

**本文使用的spring版本是4.0.2。**

在讲解<mvc:annotation-driven>这个配置之前，我们先了解下Spring的消息转换机制。@ResponseBody这个注解就是使用消息转换机制，最终通过json的转换器转换成json数据的。</mvc:annotation-driven>

HttpMessageConverter接口就是Spring提供的http消息转换接口。有关这方面的知识大家可以参考"参考资料"中的[第二条链接](http://my.oschina.net/lichhao/blog/172562)，里面讲的很清楚。

![](https://images0.cnblogs.com/i/411512/201405/101510002604230.png)

下面开始分析<mvc:annotation-driven>这句配置:</mvc:annotation-driven>

这句代码在spring中的解析类是：

![](https://images0.cnblogs.com/i/411512/201405/101606162131470.png)

在AnnotationDrivenBeanDefinitionParser源码的152行parse方法中：

分别实例化了RequestMappingHandlerMapping，ConfigurableWebBindingInitializer，RequestMappingHandlerAdapter等诸多类。

其中**RequestMappingHandlerMapping和RequestMappingHandlerAdapter**这两个类比较重要。

RequestMappingHandlerMapping处理请求映射的，处理@RequestMapping跟请求地址之间的关系。

RequestMappingHandlerAdapter是请求处理的适配器，也就是请求之后处理具体逻辑的执行，关系到哪个类的哪个方法以及转换器等工作，这个类是我们讲的重点，其中它的属性messageConverters是本文要讲的重点。

![](https://images0.cnblogs.com/i/411512/201405/101611179016436.png)

私有方法:getMessageConverters

![](https://images0.cnblogs.com/i/411512/201405/101630232136603.png)

从代码中我们可以，RequestMappingHandlerAdapter设置messageConverters的逻辑：

1.如果<mvc:annotation-driven>节点有子节点message-converters，那么它的转换器属性messageConverters也由这些子节点组成。</mvc:annotation-driven>

message-converters的子节点配置如下：

```
<mvc:annotation-driven>
  <mvc:message-converters>
    <bean class="org.example.MyHttpMessageConverter"/>
    <bean class="org.example.MyOtherHttpMessageConverter"/>
  </mvc:message-converters>
</mvc:annotation-driven>
```

2.message-converters子节点不存在或它的属性register-defaults为true的话，加入其他的转换器：ByteArrayHttpMessageConverter、StringHttpMessageConverter、ResourceHttpMessageConverter等。

我们看到这么一段：

![](https://images0.cnblogs.com/i/411512/201405/101640298384297.png)

这些boolean属性是哪里来的呢，它们是AnnotationDrivenBeanDefinitionParser的静态变量。

![](https://images0.cnblogs.com/i/411512/201405/101641297132356.png)

 其中ClassUtils中的isPresent方法如下：

![](https://images0.cnblogs.com/i/411512/201405/101643277139672.png)

看到这里，读者应该明白了为什么本文一开始在pom文件中需要加入对应的jackson依赖，为了让json转换器jackson成为默认转换器之一。

<mvc:annotation-driven>的作用读者也明白了。</mvc:annotation-driven>

下面我们看如何通过消息转换器将java对象进行转换的。

RequestMappingHandlerAdapter在进行handle的时候，会委托给HandlerMethod（具体由子类ServletInvocableHandlerMethod处理）的invokeAndHandle方法进行处理，这个方法又转接给HandlerMethodReturnValueHandlerComposite处理。

HandlerMethodReturnValueHandlerComposite维护了一个HandlerMethodReturnValueHandler列表。**HandlerMethodReturnValueHandler是一个对返回值进行处理的策略接口，这个接口非常重要。关于这个接口的细节，请参考楼主的另外一篇博客：**[http://www.cnblogs.com/fangjian0423/p/springMVC-request-param-analysis.html](http://www.cnblogs.com/fangjian0423/p/springMVC-request-param-analysis.html)。然后找到对应的HandlerMethodReturnValueHandler对结果值进行处理。

最终找到RequestResponseBodyMethodProcessor这个Handler（由于使用了@ResponseBody注解）。

RequestResponseBodyMethodProcessor的supportsReturnType方法：

![](https://images0.cnblogs.com/i/411512/201405/101803027605809.png)

然后使用handleReturnValue方法进行处理：

![](https://images0.cnblogs.com/i/411512/201405/101803105889900.png)

我们看到，这里使用了转换器。　　

具体的转换方法：

![](https://images0.cnblogs.com/i/411512/201405/101809037135949.png)

![](https://images0.cnblogs.com/i/411512/201405/102031439173571.png)

至于为何是请求头部的**Accept**数据，读者可以进去debug这个**getAcceptableMediaTypes**方法看看。 我就不罗嗦了～～～

 ok。至此，我们走遍了所有的流程。

现在，回过头来看。为什么一开始的demo输出了json数据？

我们来分析吧。

由于我们只配置了<mvc:annotation-driven>，因此使用spring默认的那些转换器。</mvc:annotation-driven>

![](https://images0.cnblogs.com/i/411512/201405/101816581047144.png)

很明显，我们看到了2个xml和1个json转换器。 **要看能不能转换，得看HttpMessageConverter接口的public boolean canWrite(Class<?> clazz, MediaType mediaType)方法是否返回true来决定的。**

我们先分析SourceHttpMessageConverter：

它的canWrite方法被父类AbstractHttpMessageConverter重写了。

![](https://images0.cnblogs.com/i/411512/201405/101830573234896.png)

![](https://images0.cnblogs.com/i/411512/201405/101832284176592.png)

![](https://images0.cnblogs.com/i/411512/201405/101832352929525.png)

发现SUPPORTED_CLASSES中没有Map类(本文demo返回的是Map类)，因此不支持。

下面看Jaxb2RootElementHttpMessageConverter：

这个类直接重写了canWrite方法。

![](https://images0.cnblogs.com/i/411512/201405/101838053851073.png)

需要有XmlRootElement注解。 很明显，Map类当然没有。

最终MappingJackson2HttpMessageConverter匹配，进行json转换。（为何匹配，请读者自行查看源码）

## 实例讲解

 我们分析了转换器的转换过程之后，下面就通过实例来验证我们的结论吧。

首先，我们先把xml转换器实现。

之前已经分析，默认的转换器中是支持xml的。下面我们加上注解试试吧。

由于Map是jdk源码中的部分，因此我们用Employee来做demo。

因此，Controller加上一个方法：

<pre>@RequestMapping("/xmlOrJsonSimple")
@ResponseBody public Employee xmlOrJsonSimple() { return employeeService.getById(1);
}</pre>

实体中加上@XmlRootElement注解

![](https://images0.cnblogs.com/i/411512/201405/101903141989122.png)



![](https://images0.cnblogs.com/i/411512/201405/101904598389030.png)

我们发现，解析成了xml。

这里为什么解析成xml，而不解析成json呢？

之前分析过，消息转换器是根据class和mediaType决定的。

我们使用firebug看到：

![](https://images0.cnblogs.com/i/411512/201405/102222464019898.png)

我们发现Accept有xml，没有json。因此解析成xml了。

我们再来验证，同一地址，HTTP头部不同Accept。看是否正确。


<pre>$.ajax({
    url: "${request.contextPath}/employee/xmlOrJsonSimple",
    success: function(res) {
        console.log(res);
    },
    headers: { "Accept": "application/xml" }
});</pre>

<pre>$.ajax({
    url: "${request.contextPath}/employee/xmlOrJsonSimple",
    success: function(res) {
        console.log(res);
    },
    headers: { "Accept": "application/json" }
});</pre>


验证成功。

## 关于配置

如果不想使用<mvc:annotation-driven>中默认的RequestMappingHandlerAdapter的话，我们可以在重新定义这个bean，spring会覆盖掉默认的RequestMappingHandlerAdapter。</mvc:annotation-driven>

为何会覆盖，请参考楼主的另外一篇博客：[http://www.cnblogs.com/fangjian0423/p/spring-Ordered-interface.html](http://www.cnblogs.com/fangjian0423/p/spring-Ordered-interface.html)

<pre>    ` <bean><property name="messageConverters"><list><bean><bean><bean></bean></bean></bean></list></property></bean> ` </pre>

或者如果只想换messageConverters的话。

```
<mvc:annotation-driven>
  <mvc:message-converters>
    <bean class="org.example.MyHttpMessageConverter"/>
    <bean class="org.example.MyOtherHttpMessageConverter"/>
  </mvc:message-converters>
</mvc:annotation-driven>

```

如果还想用其他converters的话。

![](https://images0.cnblogs.com/i/411512/201405/102311480731629.png)

以上是spring-mvc jar包中的converters。

这里我们使用转换xml的MarshallingHttpMessageConverter。

这个converter里面使用了marshaller进行转换

![](https://images0.cnblogs.com/i/411512/201405/102313161827280.png)

我们这里使用XStreamMarshaller。　　

![](https://images0.cnblogs.com/i/411512/201405/102319292603758.png)

![](https://images0.cnblogs.com/i/411512/201405/102319412294581.png)

json没有转换器，返回406.

至于xml格式的问题，大家自行解决吧。 这里用的是XStream～。

使用这种方式，pom别忘记了加入xstream的依赖：

```
<dependency>
  <groupId>com.thoughtworks.xstream</groupId>
  xstream
  <version>1.4.7</version>
</dependency>
```

## 总结

 写了这么多，可能读者觉得有点罗嗦。 毕竟这也是自己的一些心得，希望都能说出来与读者共享。

刚接触SpringMVC的时候，发现这种自动转换机制很牛逼，但是一直没有研究它的原理，目前，算是了了一个小小心愿吧，SpringMVC还有很多内容，以后自己研究其他内容的时候还会与大家一起共享的。

文章难免会出现一些错误，希望读者们能指明出来。

## 详解RequestBody和@ResponseBody注解

概述 在SpringMVC中，可以使用@RequestBody和@ResponseBody两个注解，分别完成请求报文到对象和对象到响应报文的转换，底层这种灵活的消息转换机制，就是Spring3.x中新引入的HttpMessageConverter即消息转换器机制。

Http请求的抽象 还是回到请求-响应，也就是解析请求体，然后返回响应报文这个最基本的Http请求过程中来。我们知道，在servlet标准中，可以用javax.servlet.ServletRequest接口中的以下方法：

```
public ServletInputStream getInputStream() throws IOException; 

```

来得到一个ServletInputStream。这个ServletInputStream中，可以读取到一个原始请求报文的所有内容。同样的，在javax.servlet.ServletResponse接口中，可以用以下方法：

```
public ServletOutputStream getOutputStream() throws IOException;

```

来得到一个ServletOutputStream，这个ServletOutputSteam，继承自java中的OutputStream，可以让你输出Http的响应报文内容。

让我们尝试着像SpringMVC的设计者一样来思考一下。我们知道，Http请求和响应报文本质上都是一串字符串，当请求报文来到java世界，它会被封装成为一个ServletInputStream的输入流，供我们读取报文。响应报文则是通过一个ServletOutputStream的输出流，来输出响应报文。

我们从流中，只能读取到原始的字符串报文，同样，我们往输出流中，也只能写原始的字符。而在java世界中，处理业务逻辑，都是以一个个有业务意义的**对象**为处理维度的，那么在报文到达SpringMVC和从SpringMVC出去，都存在一个字符串到java对象的阻抗问题。这一过程，不可能由开发者手工转换。我们知道，在Struts2中，采用了OGNL来应对这个问题，而在SpringMVC中，它是HttpMessageConverter机制。我们先来看两个接口。

HttpInputMessage 这个类是SpringMVC内部对一次Http请求报文的抽象，在HttpMessageConverter的read()方法中，有一个HttpInputMessage的形参，它正是SpringMVC的消息转换器所作用的受体“请求消息”的内部抽象，消息转换器从“请求消息”中按照规则提取消息，转换为方法形参中声明的对象。

```
package org.springframework.http;

import java.io.IOException;
import java.io.InputStream;

public interface HttpInputMessage extends HttpMessage {

    InputStream getBody() throws IOException;

}

```

HttpOutputMessage 这个类是SpringMVC内部对一次Http响应报文的抽象，在HttpMessageConverter的write()方法中，有一个HttpOutputMessage的形参，它正是SpringMVC的消息转换器所作用的受体“响应消息”的内部抽象，消息转换器将“响应消息”按照一定的规则写到响应报文中。

```
package org.springframework.http;

import java.io.IOException;
import java.io.OutputStream;

public interface HttpOutputMessage extends HttpMessage {

    OutputStream getBody() throws IOException;

}

```

HttpMessageConverter 对消息转换器最高层次的接口抽象，描述了一个消息转换器的一般特征，我们可以从这个接口中定义的方法，来领悟Spring3.x的设计者对这一机制的思考过程。

```
package org.springframework.http.converter;

import java.io.IOException;
import java.util.List;

import org.springframework.http.HttpInputMessage;
import org.springframework.http.HttpOutputMessage;
import org.springframework.http.MediaType;

public interface HttpMessageConverter<T> {

    boolean canRead(Class<?> clazz, MediaType mediaType);

    boolean canWrite(Class<?> clazz, MediaType mediaType);

    List<MediaType> getSupportedMediaTypes();

    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException;

    void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException;

}

```

HttpMessageConverter接口的定义出现了成对的canRead()，read()和canWrite()，write()方法，MediaType是对请求的Media Type属性的封装。举个例子，当我们声明了下面这个处理方法。

```
@RequestMapping(value="/string", method=RequestMethod.POST)
public @ResponseBody String readString(@RequestBody String string) {
    return "Read string '" + string + "'";
}

```

在SpringMVC进入readString方法前，会根据@RequestBody注解选择适当的HttpMessageConverter实现类来将请求参数解析到string变量中，具体来说是使用了StringHttpMessageConverter类，它的canRead()方法返回true，然后它的read()方法会从请求中读出请求参数，绑定到readString()方法的string变量中。

当SpringMVC执行readString方法后，由于返回值标识了@ResponseBody，SpringMVC将使用StringHttpMessageConverter的write()方法，将结果作为String值写入响应报文，当然，此时canWrite()方法返回true。

我们可以用下面的图，简单描述一下这个过程。

![消息转换图](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825151641382-1716038917.png)

RequestResponseBodyMethodProcessor 将上述过程集中描述的一个类是org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor，这个类同时实现了HandlerMethodArgumentResolver和HandlerMethodReturnValueHandler两个接口。前者是将请求报文绑定到处理方法形参的策略接口，后者则是对处理方法返回值进行处理的策略接口。两个接口的源码如下：

```
package org.springframework.web.method.support;

import org.springframework.core.MethodParameter;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;

public interface HandlerMethodArgumentResolver {

    boolean supportsParameter(MethodParameter parameter);

    Object resolveArgument(MethodParameter parameter,
                           ModelAndViewContainer mavContainer,
                           NativeWebRequest webRequest,
                           WebDataBinderFactory binderFactory) throws Exception;

}

package org.springframework.web.method.support;

import org.springframework.core.MethodParameter;
import org.springframework.web.context.request.NativeWebRequest;

public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(MethodParameter returnType);

    void handleReturnValue(Object returnValue,
                           MethodParameter returnType,
                           ModelAndViewContainer mavContainer,
                           NativeWebRequest webRequest) throws Exception;

}

```

RequestResponseBodyMethodProcessor这个类，同时充当了方法参数解析和返回值处理两种角色。我们从它的源码中，可以找到上面两个接口的方法实现。

对HandlerMethodArgumentResolver接口的实现：

```
public boolean supportsParameter(MethodParameter parameter) {
    return parameter.hasParameterAnnotation(RequestBody.class);
}

public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

    Object argument = readWithMessageConverters(webRequest, parameter, parameter.getGenericParameterType());

    String name = Conventions.getVariableNameForParameter(parameter);
    WebDataBinder binder = binderFactory.createBinder(webRequest, argument, name);

    if (argument != null) {
        validate(binder, parameter);
    }

    mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());

    return argument;
}

```

对HandlerMethodReturnValueHandler接口的实现

```
public boolean supportsReturnType(MethodParameter returnType) {
    return returnType.getMethodAnnotation(ResponseBody.class) != null;
}

    public void handleReturnValue(Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
        throws IOException, HttpMediaTypeNotAcceptableException {

    mavContainer.setRequestHandled(true);
    if (returnValue != null) {
        writeWithMessageConverters(returnValue, returnType, webRequest);
    }
}

```

看完上面的代码，整个HttpMessageConverter消息转换的脉络已经非常清晰。因为两个接口的实现，分别是以是否有@RequestBody和@ResponseBody为条件，然后分别调用HttpMessageConverter来进行消息的读写。

如果你想问，怎么样跟踪到RequestResponseBodyMethodProcessor中，请你按照前面几篇博文的思路，然后到这里[spring-mvc-showcase](https://github.com/spring-projects/spring-mvc-showcase)下载源码回来，对其中HttpMessageConverter相关的例子进行debug，只要你肯下功夫，相信你一定会有属于自己的收获的。

思考 张小龙在谈微信的本质时候说：“微信只是个平台，消息在其中流转”。在我们对SpringMVC源码分析的过程中，我们可以从HttpMessageConverter机制中领悟到类似的道理。在SpringMVC的设计者眼中，一次请求报文和一次响应报文，分别被抽象为一个请求消息HttpInputMessage和一个响应消息HttpOutputMessage。

处理请求时，由合适的消息转换器将请求报文绑定为方法中的形参对象，在这里，同一个对象就有可能出现多种不同的消息形式，比如json和xml。同样，当响应请求时，方法的返回值也同样可能被返回为不同的消息形式，比如json和xml。

在SpringMVC中，针对不同的消息形式，我们有不同的HttpMessageConverter实现类来处理各种消息形式。但是，只要这些消息所蕴含的“有效信息”是一致的，那么各种不同的消息转换器，都会生成同样的转换结果。至于各种消息间解析细节的不同，就被屏蔽在不同的HttpMessageConverter实现类中了。


## 参考资料

[http://my.oschina.net/HeliosFly/blog/205343](http://my.oschina.net/HeliosFly/blog/205343)

[http://my.oschina.net/lichhao/blog/172562](http://my.oschina.net/lichhao/blog/172562)

[http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html)
