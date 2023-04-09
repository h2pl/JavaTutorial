# 目录

  * [1.1、SpringMVC引言](#11、springmvc引言)
  * [1.2、SpringMVC的优势](#12、springmvc的优势)
  * [二、SpringMVC入门](#二、springmvc入门)
  * [2.1、环境搭建](#21、环境搭建)
  * [2.1.1、引入依赖](#211、引入依赖)
  * [2.1.2、编写配置文件](#212、编写配置文件)
  * [2.1.3、配置web.xml](#213、配置webxml)
  * [2.1.4、编写控制器](#214、编写控制器)
  * [2.2、注解详解](#22、注解详解)
  * [2.2.1、@Controller](#221、controller)
  * [2.2.2、@RequestMapping](#222、requestmapping)
  * [2.3、SpringMVC的跳转方式](#23、springmvc的跳转方式)
  * [2.3.1、Controller ——>前台页面](#231、controller-前台页面)
  * [2.3.1.1、forward](#2311、forward)
  * [2.3.1.2、redirect](#2312、redirect)
  * [2.3.1Controller ——>Controller](#231controller-controller)
  * [2.3.1.1、forward](#2311、forward-1)
  * [2.3.1.2、redirect](#2312、redirect-1)
  * [2.4、SpringMVC的参数接收](#24、springmvc的参数接收)
  * [2.4.1、Servlet接收参数的方式](#241、servlet接收参数的方式)
  * [2.4.2、SpringMVC的参数接收](#242、springmvc的参数接收)
  * [2.4.2.1、基本数据类型](#2421、基本数据类型)
  * [2.4.2.2、对象类型](#2422、对象类型)
  * [2.4.2.3、数组类型](#2423、数组类型)
  * [2.4.2.4、集合类型](#2424、集合类型)
  * [2.5、SpringMVC接收参数中文乱码问题](#25、springmvc接收参数中文乱码问题)
  * [2.5.1、GET请求](#251、get请求)
  * [2.5.2、POST请求](#252、post请求)
  * [2.5.2.1、自定义过滤器解决POST乱码请求](#2521、自定义过滤器解决post乱码请求)
  * [2.5.2.2、使用CharacterEncodingFilter解决POST乱码请求](#2522、使用characterencodingfilter解决post乱码请求)
  * [2.6、SpringMVC中数据传递机制](#26、springmvc中数据传递机制)
  * [2.6.1、什么事数据传递机制](#261、什么事数据传递机制)
  * [2.6.2、Servlet的数据传递机制](#262、servlet的数据传递机制)
  * [三、前端控制器](#三、前端控制器)
  * [3.1、什么是前端控制器](#31、什么是前端控制器)
  * [3.2、代码实现](#32、代码实现)
  * [3.3、注意](#33、注意)
  * [3.4、映射路径](#34、映射路径)
  * [3.4.1、访问静态资源和 JSP 被拦截的原因](#341、访问静态资源和-jsp-被拦截的原因)
  * [3.4.2、如何解决](#342、如何解决)
  * [3.4.2.1、方式一](#3421、方式一)
  * [3.4.2.2、方式二](#3422、方式二)
  * [3.5、@ModelAttribute 注解](#35、modelattribute-注解)
  * [四、处理响应](#四、处理响应)
  * [4.1、返回 ModelAndView](#41、返回-modelandview)
  * [4.2、返回String](#42、返回string)
  * [4.3、改进](#43、改进)
  * [五、请求转发和重定向](#五、请求转发和重定向)
  * [5.1、请求转发和重定向的区别](#51、请求转发和重定向的区别)
  * [5.2、请求转发](#52、请求转发)
  * [5.3、重定向](#53、重定向)
  * [5.4、请求路径](#54、请求路径)
  * [六、参数处理](#六、参数处理)
  * [6.1、处理简单类型的请求参数](#61、处理简单类型的请求参数)
  * [6.1.1、请求参数名和控制器方法参数列表形参同名](#611、请求参数名和控制器方法参数列表形参同名)
  * [6.1.2、请求参数名和控制器方法参数列表形参不同名](#612、请求参数名和控制器方法参数列表形参不同名)
  * [6.2、处理复杂类型的请求参数](#62、处理复杂类型的请求参数)
  * [6.2.1、数组类型](#621、数组类型)
  * [6.2.2、自定义类型](#622、自定义类型)
  * [6.3、处理日期类型的请求参数](#63、处理日期类型的请求参数)
  * [6.3.1、日期在请求参数上](#631、日期在请求参数上)
  * [6.3.2、在封装的对象上](#632、在封装的对象上)
  * [七、文件上传与下载](#七、文件上传与下载)
  * [7.1、文件上传](#71、文件上传)
  * [7.1.1、编写表单](#711、编写表单)
  * [7.1.2、修改web.xml](#712、修改webxml)
  * [7.1.3、配置上传解析器](#713、配置上传解析器)
  * [7.1.4、配置上传控制器](#714、配置上传控制器)
  * [7.2、文件下载](#72、文件下载)
  * [7.2.1、开发控制器](#721、开发控制器)



## 1.1、SpringMVC引言

为了使Spring有可插入的MVC架构,SpringFrameWork在Spring基础上开发SpringMVC框架,从而在使用Spring进行WEB开发时可以选择使用**Spring的SpringMVC框架作为web开发的控制器框架**。

## 1.2、SpringMVC的优势

SpringMVC是一个**典型的轻量级MVC框架**，在整个MVC架构中充当控制器框架,相对于之前的struts2框架,**SpringMVC运行更快,其注解式开发更高效灵活**。

<figure data-size="normal">

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-3363c79f645a232562a2d6e800a11967_720w.webp)
</figure>

1.  可以和Spring框架无缝整合。
2.  运行效率远远高于struts2框架。
3.  注解式开发更高效。

## 二、SpringMVC入门

## 2.1、环境搭建

## 2.1.1、引入依赖

依赖就忽略了，我放在了评论区！

## 2.1.2、编写配置文件



```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:context="http://www.springframework.org/schema/context"
  xmlns:mvc="http://www.springframework.org/schema/mvc"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd">

<!--  1\. 开启注解扫描-->
  <context:component-scan base-package="com.lin.controller"/>
<!--  2\. 配置处理器映射器-->
<!--  <bean  />-->
<!--  3\. 开启处理器适配器-->
<!--  <bean  />-->
<!--  上面两段配置被下面的一句话所替代（封装）-->
  <mvc:annotation-driven />
  <!--  4\. 开启视图解析器-->
  <bean >
    <property name="prefix" value="/"/>
    <property name="suffix" value=".jsp"/>
  </bean>
</beans>
复制代码
```



## 2.1.3、配置web.xml



```
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>

<!--  配置springmvc的核心servlet-->
  <servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
<!--      告诉springmvc配置文件的位置-->
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:springmvc.xml</param-value>
    </init-param>
  </servlet>
  <servlet-mapping>
<!--    拦截所有请求-->
    <servlet-name>springmvc</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
复制代码
```



## 2.1.4、编写控制器



```
package com.lin.controller;

import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

/**
 * @author XiaoLin
 * @date 2021/2/17 17:09
 */
@Controller
public class HellowController {

  /**
      * @Description:第一个springmvc测试类
      * @author XiaoLin
      * @date 2021/2/17
      * @Param: [username, password]
      * @return java.lang.String
      */
     /*
      RequestMapping的修饰范围：可以用在类上和方法上，他的作用如下：
      1\. 用在方法上可以给当前方法加入指定的请求路径
      2\. 用在类上可以给类中的所有方法都加入一个统一的请求路径，在这个方法访问之前都必须加上
      */
  @RequestMapping("/hello")
  public String hello(String username,String password){
    System.out.println("hello");
    return "index";
  }
}

复制代码
```



## 2.2、注解详解

## 2.2.1、@Controller

该注解用来类上标识这是一个控制器组件类并创建这个类实例，告诉spring我是一个控制器。

## 2.2.2、@RequestMapping

这个注解可以作用在方法上或者是类上，用来指定请求路径。

## 2.3、SpringMVC的跳转方式

传统的Servlet开发跳转方式有两种：

1.  forward：forward跳转，是在服务器内部跳转，所以是一次请求，地址栏不变。跳转时可以携带数据进行传递（使用request作用域进行传递）。
2.  redirect：redirect跳转是客户端跳转，所以是多次请求,地址栏会改变，跳转时不可以携带数据传递。

## 2.3.1、Controller ——>前台页面

## 2.3.1.1、forward

通过测试我们可以发现，SpringMVC默认的就是使用请求转发的方式来进行跳转到前台页面的;



```
@Controller
@RequestMapping("forwoartAndRedirect")
public class TestForwoartAndRedirect {

  @RequestMapping("test")
  public String test(){
    System.out.println("test");
    return"index";
  }
}
复制代码
```



## 2.3.1.2、redirect

如果我们想使用重定向的方式来进行跳转的话，需要使用SpringMVC提供给我们的关键字——redirect:来完成。

语法：return "redirect:/视图全路径名";

**注意：**在redirect:后接页面的不是逻辑名，而是全路径名。因为redirect跳转不会经过视图解析器。

## 2.3.1Controller ——>Controller

## 2.3.1.1、forward

如果我们想使用请求转发的方式跳转到相同(不同)Controller的不同方法的时候，我们也需要使用SpringMVC提供的关键字：forward:。

语法：return:"forward: /需要跳转的类上的@RequestMapping的值/需要跳转的方法上的@RequestMapping的值;"

## 2.3.1.2、redirect

如果我们想使用重定向的方式跳转到相同(不同)Controller的不同方法的时候，我们也需要使用SpringMVC提供的关键字：redirect:。

语法：return:"redirect: /需要跳转的类上的@RequestMapping的值/需要跳转的方法上的@RequestMapping的值;"

## 2.4、SpringMVC的参数接收

## 2.4.1、Servlet接收参数的方式

在传统的Servlet开发，我们一般都是用这种方式来进行接收请求参数的。



```
// 接收名字为name的参数
request.getParameter(name)
复制代码
```



他有几个需要注意的点：

1.  参数要求是表单域的name属性。
2.  getParameter方法用于获取单个值, 返回类型是String。
3.  getParameterValues方法用于获取一组数据, 返回结果是String[]。
4.  冗余代码较多, 使用麻烦, 类型需要自己转换。

## 2.4.2、SpringMVC的参数接收

SpringMVC使用的是控制器中方法形参列表来接收客户端的请求参数，他可以进行自动类型转换，**要求传递参数的key要与对应方法的形参变量名一致才可以完成自动赋值**。他的优势很明显：

1.  简化参数接收形式(不需要调用任何方法, 需要什么参数, 就在控制器方法中提供什么参数)。
2.  参数类型不需要自己转换了。日期时间（默认为yyyy/MM/dd）得注意，需要使用@DateTimeFormat注解声明日期转换时遵循的格式, 否则抛出400异常。

## 2.4.2.1、基本数据类型

**要求传递参数的key要与对应方法的形参变量名一致才可以完成自动赋值。**

## 2.4.2.2、对象类型

如果我们需要接收对象类型的话，直接将需要接收的对象作为控制器的方法参数声明即可。SpringMVC会自动封装对象，若传递参数key与对象中属性名一致，就会自动封装成对象。

## 2.4.2.3、数组类型

如果我们需要接收数组类型的时候，只需将要接收的数组类型直接声明为方法的形式参数即可。

## 2.4.2.4、集合类型

SpringMVC不能直接通过形式参数列表的方式接收集合类型的参数，如果需要接收集合类型的参数必须将集合放入一个对象中，并且提供get/set方法，才可以。推荐放入VO对象中进行封装，进而使用对象类型来进行接收。

## 2.5、SpringMVC接收参数中文乱码问题

## 2.5.1、GET请求

GET请求方式出现乱码需要分Tomcat版本进行讨论：

1.  Tomcat8.x版本之前：默认使用server.xml中的URIEncoding="ISO-8859-1"编码，而不是"UTF-8"编码，进而会出现中文乱码。
2.  Tomcat8.x版本之后：默认使用server.xml中的URIEncoding="UTF-8"，所以不会出现中文乱码问题。

## 2.5.2、POST请求

SpringMVC中默认没有对POST请求进行任何编码处理，所以无论什么版本直接接收POST请求都会出现中文乱码。

## 2.5.2.1、自定义过滤器解决POST乱码请求

在Servlet阶段，我们学过过滤器，我们可以自定义过滤器来进行过滤编码。



```
package com.filter;

import javax.servlet.*;
import java.io.IOException;

//自定义编码filter
public class CharacterEncodingFilter  implements Filter {

    private String encoding;
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        this.encoding = filterConfig.getInitParameter("encoding");
        System.out.println(encoding);
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        request.setCharacterEncoding(encoding);
        response.setCharacterEncoding(encoding);
        chain.doFilter(request,response);
    }

    @Override
    public void destroy() {

    }
}
复制代码
<!--配置post请求方式中文乱码的Filter-->
  <filter>
    <filter-name>charset</filter-name>
    <filter-class>com.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>UTF-8</param-value>
    </init-param>
  </filter>

  <filter-mapping>
    <filter-name>charset</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
复制代码
```



## 2.5.2.2、使用CharacterEncodingFilter解决POST乱码请求



```
<!--配置post请求方式中文乱码的Filter-->
  <filter>
    <filter-name>charset</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
      <param-name>encoding</param-name>
      <param-value>UTF-8</param-value>
    </init-param>
  </filter>

  <filter-mapping>
    <filter-name>charset</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
复制代码
package com.filter;

import javax.servlet.*;
import java.io.IOException;

//自定义编码filter
public class CharacterEncodingFilter  implements Filter {

    private String encoding;
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        this.encoding = filterConfig.getInitParameter("encoding");
        System.out.println(encoding);
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        request.setCharacterEncoding(encoding);
        response.setCharacterEncoding(encoding);
        chain.doFilter(request,response);
    }

    @Override
    public void destroy() {

    }
}
复制代码
```



## 2.6、SpringMVC中数据传递机制

## 2.6.1、什么事数据传递机制

数据传递机制主要包含三个问题：

1.  数据如何存储？
2.  如何在页面中获取数据？
3.  在页面中获取的数据该如何展示？

## 2.6.2、Servlet的数据传递机制

在以前的Servlet开发中，我们一般是将数据放入作用域（request、session、application），如果数据是单个的直接用EL表达式在前端进行展示，如果是集合或者数组，可以用EL表达式➕JSTL标签进行遍历后在前端进行展示。

## 三、前端控制器

## 3.1、什么是前端控制器

在 MVC 框架中都存在一个前端控制器，在 WEB 应用的前端（Front）设置一个入口控制器（Controller），是用来提供一个集中的请求处理机制，所有的请求都被发往该控制器统一处理，然后把请求分发给各自相应的处理程序。一般用来做一个共同的处理，如权限检查，授权，日志记录等。因为前端控制的集中处理请求的能力，因此提高了可重用性和可拓展性。

在没有前端控制器的时候，我们是这样传递和处理请求的。

<figure data-size="normal">

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/52752e7a606d50ae243cf94309de682b_v2-aadb0832d3c878ff17af7dfe66e7f914_720w.webp)
</figure>

有了前端控制器之后，我们变成了这样。

<figure data-size="normal">

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-cb0dcdd3922ebfba4781800dd492dd69_720w.webp)
</figure>

## 3.2、代码实现

Spring MVC 已经提供了一个 DispatcherServlet 类作为前端控制器，所以要使用 Spring MVC 必须在web.xml 中配置前端控制器。



```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                        http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
  version="3.0">
  <!-- Spring MVC 前端控制器-->
  <servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-
      class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- 指定 Spring 容器启动加载的配置文件-->
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:mvc.xml</param-value>
    </init-param>
    <!-- Tomcat 启动初始化 -->
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
复制代码
```



## 3.3、注意

load-on-startup 元素是可选的：若值为 0 或者大于 0 时，表示容器在应用启动时就构建 Servlet 并调用其 init 方法做初始化操作（非负数的值越小，启动该 Servlet 的优先级越高）；若值为一个负数时或者没有指定时，则在第一次请求该 Servlet 才加载。配置的话，就可以让 SpringMVC 初始化的工作在容器启动的时候完成，而不是丢给用户请求去完成，提高用户访问的体验性。

## 3.4、映射路径

配置前端控制器的映射路径一般有以下的三种形式：

1.  配置如 .do、.htm 是最传统方式，可以访问静态文件（图片、 JS、 CSS 等），但不支持 RESTful风格。
2.  配置成 /，可以支持流行的 RESTful 风格，但会导致静态文件（图片、 JS、 CSS 等）被拦截后不能访问。
3.  配置成 /*，是错误的方式，可以请求到 Controller 中，但跳转到调转到 JSP 时被拦截，不能渲染JSP 视图，也会导致静资源访问不了。

## 3.4.1、访问静态资源和 JSP 被拦截的原因



```
Tomcat 容器处理静态资源是交由内置 DefaultServlet 来处理的（拦截路径是 /），处理 JSP 资源是交由内置的 JspServlet 处理的（拦截路径是*.jsp | *.jspx）。
    启动项目时，先加载容器的 web.xml，而后加载项目中的 web.xml。当拦截路径在两者文件中配置的一样，后面会覆盖掉前者。
    所以前端控制器配置拦截路径是 / 的所有静态资源都会交由前端控制器处理，而拦截路径配置 /*，所有静态资源和 JSP 都会交由前端控制器处理。
复制代码
```



## 3.4.2、如何解决

## 3.4.2.1、方式一

在 web.xml 中修改，修改前端控制器的映射路径修改为*.do，**但注意，访问控制器里的处理方法时，请求路径须携带 .do。**



```
<servlet-mapping>
	<servlet-name>dispatcherServlet</servlet-name>
	<url-pattern>*.do</url-pattern>
</servlet-mapping>	
复制代码
```



## 3.4.2.2、方式二

在 mvc.xml中加入一段配置，这个配置会在 Spring MVC 上下文中创建存入一个 DefaultServletHttpRequestHandler 的 bean，它会 对进入DispatcherServlet的请求进行筛查，若不是映射的请求，就将该请求交由容器默认的 Servlet处理。



```
<mvc:default-servlet-handler/>
复制代码
```



## 3.5、@ModelAttribute 注解

在形参中的对象（**必须是自定义类型**），SpringMVC会默认将他存入Model中，名称是参数的类名首字母小写，有些时候，这个类会显得格外长，但是我们又有这种需求，比方说：查询条件的回显。我们只需在自定义类的前面加@ModelAttribute，里面写我们需要修改的key的名称即可。



```
package cn.wolfcode.web.controller;
	@Controller
	public class RequestController {
		@RequestMapping("/req7")
			public String resp7(@ModelAttribute("u") User user) {
			return "m";
		}
}
复制代码
```



## 四、处理响应

SpringMVC的作用是请求和处理响应，响应处理是指怎么编写控制器里面的处理方法接受请求做响应，找视图文件和往作用域中存入数据。要处理方法要做响应，一般处理方法返回的类型为 ModelAndView 和 String。

## 4.1、返回 ModelAndView

方法中返回 ModelAndView 对象，此对象中设置模型数据并指定视图。前端依旧是使用JSTL+CgLib来进行取值。他有两个常用方法：

1.  addObject(String key, Object value)：设置共享数据的 key 和 value。
2.  addObject(Object value)：设置共享数据的 value，key 为该 value 类型首字母小写。



```
package cn.linstudy.web.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class ResponseController {
  // 提供方法处理请求，localhost/resp1
  @RequestMapping("/resp1")
  public ModelAndView resp1() {
// 通过创建这个类对象，告诉 Spring MVC 找什么视图文件， 往作用域或者说往模型中存入什么数据
    ModelAndView mv = new ModelAndView();
// 往作用域或者模型中存入数据
    mv.addObject("msg", "方法返回类型是 ModelAndView");
// 找视图
    mv.setViewName("/WEB-INF/views/resp.jsp");
    return mv;
  }
复制代码
```



## 4.2、返回String

返回 String 类型（使用广泛），此时如果我们需要共享数据，那么就需要用到HttpServlet对象，Spring帮我们封装好了一个对象：Model 。组合使用，用其往作用域或模型中存入数据。



```
package cn.instudy.web.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class ResponseController {
  // 提供方法处理请求，localhost/resp2
  @RequestMapping("/resp2")
  public String resp2(Model model) {
// 往作用域或者模型中存入数据
    model.addAttribute("msg", "方法返回类型是 String");
// 返回视图名
    return "/WEB-INF/views/resp.jsp";
  }
}
复制代码
```



## 4.3、改进

我们会发现，如果我们需要写返回界面的话需要不断的写前缀和后缀，这个时候需要进行消除消除视图前缀和后缀，我们只需在Spring中进行配置视图解析器即可。



```
<!--
配置视图解析器 配置这个Spring MVC 找视图的路径就是：前缀 + 逻辑视图名（处理方法设置或返回视图名）+ 后缀名
-->
<bean >
	<!-- 视图前缀 -->
	<property name="prefix" value="/WEB-INF/views/"/>
	<!-- 视图后缀 -->
	<property name="suffix" value=".jsp"/>
</bean>
复制代码
```



## 五、请求转发和重定向

## 5.1、请求转发和重定向的区别

|  | 几次请求 | 地址栏 | WEB-INF中资源 | 共享请求数据 | 有无表单重复提交 |
| --- | --- | --- | --- | --- | --- |
| 请求转发 | 1次 | 不改变 | 可以访问 | 可以共享 | 有 |
| 重定向 | 多次 | 改变 | 不可访问 | 不可共享 | 无 |

## 5.2、请求转发

加上forward 关键字，表示请求转发，相当于request.getRequestDispatcher().forward(request,response)，转发后浏览器地址栏不变，共享之前请求中的数据。**加了关键字后，配置的视图解析器就不起作用了。如果返回视图必须写全路径**



```
package cn.linstudy.web.controller;
@Controller
public class ResponseController {

	@RequestMapping("/TestForward")
	public String forward() {
		return "forward:/WEB-INF/views/welcome.jsp";
	}
}
复制代码
```



## 5.3、重定向

加上 redirect 关键字，表示重定向，相当于 response.sendRedirect()，重定向后浏览器地址栏变为重定向后的地址，不共享之前请求的数据。



```
package cn.linstudy.web.controller;
@Controller
public class ResponseController {
	// localhost/r
	@RequestMapping("/TestRedirect")
	public String redirect() {
		return "redirect:/static/demo.html";
	}
}
复制代码
```



## 5.4、请求路径

在请求转发和重定向的时候，我们一般有两种方式来写请求路径：

1.  加/：使用是绝对路径（推荐使用），从项目根路径找。(/response/test6 ---> "redirect:/hello.html" ---> localhost:/hello.html)
2.  不加/：使用是相对路径，相对于上一次访问上下文路径的上一级找。(/response/test6 ---> "redirect:hello.html" ---> localhost:/response/hello.html)

## 六、参数处理

## 6.1、处理简单类型的请求参数

我们在控制器的如何获取请求中的简单数据类型的参数参数？简单数据类型包含基本数据类型及其包装类、String 和BigDecimal 等形参接收。

## 6.1.1、请求参数名和控制器方法参数列表形参同名

如果前台传递过来的参数名和控制器方法中参数列表的形参参数名相同就无需做任何操作，SpringMVC会自动帮我们进赋值。



```
// 请求路径为：/req1?username=zs&age=18
package cn.linstudy.web.controller;
	@Controller
	public class RequestController {
		@RequestMapping("/req1")
		public ModelAndView resp1(String username, int age) {
			System.out.println(username);
			System.out.println(age);
			return null;
		}
}
复制代码
```



## 6.1.2、请求参数名和控制器方法参数列表形参不同名

如果前台传递过来的参数名和控制器方法中参数列表的形参参数名不相同的话，我们需要使用一个注解@RequestParam("前台携带的参数名")来告诉SpringMVC我们任何对数据来进行赋值。



```
// 请求路径为：/req1?username=zs&age=18
package cn.linstudy.web.controller;
	@Controller
	public class RequestController {
		@RequestMapping("/req1")
		public ModelAndView resp1(@RequestParam("username") String username1, @RequestParam("age") int age1) {
			System.out.println(username);
			System.out.println(age);
			return null;
		}
}
复制代码
```



## 6.2、处理复杂类型的请求参数

## 6.2.1、数组类型

对于数组类型参数，我们只需在方法参数的形参列表中定义一个同名的数组类型进行接收即可。



```
// 请求路径 /req3?ids=1&ids=2&ids=3
package cn.linstudy.web.controller;
	@Controller
	public class RequestController {
		@RequestMapping("/req3")
		public ModelAndView resp3(Long[] ids) {
			System.out.println(Arrays.toString(ids));
			return null;
		}
}
复制代码
```



## 6.2.2、自定义类型

我们在很多的时候，需要接收的是一个自定义类型的对象。比如说我们进行保存用户，需要将前台传递的数据进行封装成一个自定义的用户类型，那么这个时候，只需要保证自定义的类型里面的字段和前端传过来的字段相同（**注意传递参数名与封装对象的属性名一致**），SpringMVC即可自动进行封装。



```
// /req4?username=hehe&password=666
package cn.linstudy.web.controller;	
	@Controller
	public class RequestController {
		@RequestMapping("/req4")
		public ModelAndView resp4(User user) {
			System.out.println(user);
			return null
		}
}
复制代码
```



底层 Spring MVC 根据请求地址对应调用处理方法，调用方法时发现要传递 User 类型的实参，SpringMVC 会反射创建 User 对象，之后通过请求参数名找对应的属性，给对象的属性设置对应的参数值。

## 6.3、处理日期类型的请求参数

## 6.3.1、日期在请求参数上

如果日期在请求参数上，那么我们需要在处理方法的 Date 类型的形参贴上 @DateTimeFormat注解。



```
package cn.linstudy.controller;
	@Controller
	public class RequestController {
		@RequestMapping("/req5")
		// 注意形参的类型为 java.util.Date
		public ModelAndView resp5(@DateTimeFormat(pattern="yyyy-MM-dd")Date date) {
			System.out.println(date.toLocaleString());
			return null;
		}
}
复制代码
```



## 6.3.2、在封装的对象上

如果日期在封装对象的字段，那么我们需要在字段的上贴@DateTimeFormat注解。



```
package cn.linstudy.domain;
	public class User {
	private Long id;
	private String Username;
	private String password;
	// 增加下面这个字段，并贴注解
	@DateTimeFormat(pattern="yyyy-MM-dd")
	private Date date;
	// 省略 setter getter toString
}
复制代码
package cn.linstudy.controller;
	@Controller
	public class RequestController {
		@RequestMapping("/req6")
		public ModelAndView resp6(User user) {
			System.out.println(user);
			return null;
		}
}
复制代码
```



## 七、文件上传与下载

## 7.1、文件上传

回顾之前使用 Servlet3.0 来解决文件上传的问题，编写上传表单（POST、multipart/form-data），还在处理方法 doPost 中编写解析上传文件的代码。但是在SpringMVC是可以帮我们简化文件上传的步骤和代码。

## 7.1.1、编写表单

注意请求数据类型必须是：multipart/form-data，且请求方式是POST。



```
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>文件上传</title>
</head>
<body>
	<form action="/upload" method="POST" enctype="multipart/form-data">
		文件:<br>
		
	</form>
</body>
</html>
复制代码
```



## 7.1.2、修改web.xml

我们可以在web.xml中指定上传文件的大小。



```
<servlet>
	<servlet-name>dispatcherServlet</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<init-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath:mvc.xml</param-value>
	</init-param>
	<load-on-startup>1</load-on-startup>
	<multipart-config>
		<max-file-size>52428800</max-file-size>
		<max-request-size>52428800</max-request-size>
	</multipart-config>
</servlet>
<servlet-mapping>
	<servlet-name>dispatcherServlet</servlet-name>
	<url-pattern>/</url-pattern>
</servlet-mapping>
复制代码
```



## 7.1.3、配置上传解析器

在mvc.xml中配置上传解析器，**使用springmvc中multipartfile接收客户端上传的文件必须配置文件上传解析器且解析的id必须为multipartResolver**



```
<bean id="multipartResolver" >
  <!--控制文件上传大小单位字节 默认没有大小限制 这里是2-->
  <property name="maxUploadSize" value="2097152"/>
</bean>
复制代码
```



## 7.1.4、配置上传控制器



```
package cn.linstudy.controller;
	@Controller
	public class UploadController {
	// Spring 容器存在 ServletContext 类型的对象，所以定义好 ServletContext 类型字段贴@Autowired 注解即可获取到
	@Autowired
	private ServletContext servletContext;
	@RequestMapping("/upload")
	public ModelAndView upload(Part pic) throws Exception {
		System.out.println(pic.getContentType()); // 文件类型
		System.out.println(pic.getName()); // 文件参数名
		System.out.println(pic.getSize()); // 文件大小
		System.out.println(pic.getInputStream()); // 文件输入流
		// FileCopyUtils.copy(in, out)，一个 Spring 提供的拷贝方法
		// 获取项目 webapp 目录下 uploadDir 目录的绝对路径
		System.out.println(servletContext.getRealPath("/uploadDir"));
		return null;
	}
}
复制代码
```



## 7.2、文件下载

文件下载:将服务器上的文件下载到当前用户访问的计算机的过程称之为文件下载

## 7.2.1、开发控制器

下载时必须设置响应的头信息,指定文件以何种方式保存,另外下载文件的控制器不能存在返回值,代表响应只用来下载文件信息`



```
/**
     * 测试文件下载
     * @param fileName 要下载文件名
     * @return
     */
    @RequestMapping("download")
    public String download(String fileName, HttpServletRequest request, HttpServletResponse response) throws IOException {
        //获取下载服务器上文件的绝对路径
        String realPath = request.getSession().getServletContext().getRealPath("/down");
        //根据文件名获取服务上指定文件
        FileInputStream is = new FileInputStream(new File(realPath, fileName));
        //获取响应对象设置响应头信息
        response.setHeader("content-disposition","attachment;fileName="+ URLEncoder.encode(fileName,"UTF-8"));
        ServletOutputStream os = response.getOutputStream();
        IOUtils.copy(is,os);
        IOUtils.closeQuietly(is);
        IOUtils.closeQuietly(os);
        return null;
    }
```
