



# Spring MVC 使用HandlerInterceptor拦截请求



2018-07-26 14:07 更新







Spring的处理器映射机制包含了处理器拦截器。拦截器在你需要为特定类型的请求应用一些功能时可能很有用，比如，检查用户身份等。

<section>

处理器映射处理过程配置的拦截器，必须实现 `org.springframework.web.servlet`包下的 `HandlerInterceptor`接口。这个接口定义了三个方法： `preHandle(..)`，它在处理器实际执行 _之前_ 会被执行； `postHandle(..)`，它在处理器执行 _完毕_ 以后被执行； `afterCompletion(..)`，它在 _整个请求处理完成_ 之后被执行。这三个方法为各种类型的前处理和后处理需求提供了足够的灵活性。

`preHandle(..)`方法返回一个boolean值。你可以通过这个方法来决定是否继续执行处理链中的部件。当方法返回 `true`时，处理器链会继续执行；若方法返回 `false`， `DispatcherServlet`即认为拦截器自身已经完成了对请求的处理（比如说，已经渲染了一个合适的视图），那么其余的拦截器以及执行链中的其他处理器就不会再被执行了。

拦截器可以通过`interceptors`属性来配置，该选项在所有继承了`AbstractHandlerMapping`的处理器映射类`HandlerMapping`都提供了配置的接口。如下面代码样例所示：

```
<beans>
    <bean id="handlerMapping" class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
        <property name="interceptors">
            <list>
                <ref bean="officeHoursInterceptor"/>
            </list>
        </property>
    </bean>

    <bean id="officeHoursInterceptor" class="samples.TimeBasedAccessInterceptor">
        <property name="openingTime" value="9"/>
        <property name="closingTime" value="18"/>
    </bean>
<beans>

```

```
package samples;

public class TimeBasedAccessInterceptor extends HandlerInterceptorAdapter {

    private int openingTime;
    private int closingTime;

    public void setOpeningTime(int openingTime) {
        this.openingTime = openingTime;
    }

    public void setClosingTime(int closingTime) {
        this.closingTime = closingTime;
    }

    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler) throws Exception {
        Calendar cal = Calendar.getInstance();
        int hour = cal.get(HOUR_OF_DAY);
        if (openingTime <= hour && hour < closingTime) {
            return true;
        }
        response.sendRedirect("http://host.com/outsideOfficeHours.html");
        return false;
    }
}

```

在上面的例子中，所有被此处理器处理的请求都会被`TimeBasedAccessInterceptor`拦截器拦截。如果当前时间在工作时间以外，那么用户就会被重定向到一个HTML文件提示用户，比如显示“你只有在工作时间才可以访问本网站”之类的信息。

> 使用`RequestMappingHandlerMapping`时，实际的处理器是一个处理器方法`HandlerMethod`的实例，它标识了一个将被用于处理该请求的控制器方法。

如你所见，Spring的拦截器适配器`HandlerInterceptorAdapter`让继承`HandlerInterceptor`接口变得更简单了。

> 上面的例子中，所有控制器方法处理的请求都会被配置的拦截器先拦截到。如果你想进一步缩小拦截的URL范围，你可以通过MVC命名空间或MVC Java编程的方式来配置，或者，声明一个`MappedInterceptor`类型的bean实例来处理。具体请见 [21.16.1 启用MVC Java编程配置或MVC命名空间配置](https://www.w3cschool.cn/spring_mvc_documentation_linesh_translation/spring_mvc_documentation_linesh_translation-ouxg27ss.html)一小节。

需要注意的是，`HandlerInterceptor`的后拦截`postHandle`方法不一定总是适用于注解了`@ResponseBody`或`ResponseEntity`的方法。这些场景中，`HttpMessageConverter`会在拦截器的`postHandle`方法被调之前就把信息写回响应中。这样拦截器就无法再改变响应了，比如要增加一个响应头之类的。如果有这种需求，请让你的应用实现`ResponseBodyAdvice`接口，并将其定义为一个`@ControllerAdvice`bean或直接在`RequestMappingHandlerMapping`中配置。

</section>



