



# Spring MVC 处理器异常解析器



2018-07-26 14:32 更新







Spring的处理器异常解析器`HandlerExceptionResolver`接口的实现负责处理各类控制器执行过程中出现的异常。某种程度上讲，`HandlerExceptionResolver`与你在web应用描述符`web.xml`文件中能定义的异常映射（exception mapping）很相像，不过它比后者提供了更灵活的方式。比如它能提供异常被抛出时正在执行的是哪个处理器这样的信息。并且，一个更灵活（programmatic）的异常处理方式可以为你提供更多选择，使你在请求被直接转向到另一个URL之前（与你使用Servlet规范的异常映射是一样的）有更多的方式来处理异常。

<section>

实现`HandlerExceptionResolver`接口并非实现异常处理的唯一方式，它只是提供了`resolveException(Exception, Hanlder)`方法的一个实现而已，方法会返回一个`ModelAndView`。除此之外，你还可以框架提供的`SimpleMappingExceptionResolver`或在异常处理方法上注解`@ExceptionHandler`。`SimpleMappingExceptionResolver`允许你获取可能抛出的异常类的名字，并把它映射到一个视图名上去。这与Servlet API提供的异常映射特性是功能等价的，但你也可以基于此实现粒度更精细的异常映射。而`@ExceptionHandler`注解的方法则会在异常抛出时被调用以处理该异常。这样的方法可以定义在`@Controller`注解的控制器类里，也可以定义在`@ControllerAdvice`类中，后者可以使该异常处理方法被应用到更多的`@Controller`控制器中。下一小节将提供更为详细的信息。

</section>







# Spring MVC 使用@ExceptionHandler注解



2018-07-26 14:33 更新







<section>

`HandlerExceptionResolver`接口以及`SimpleMappingExceptionResolver`解析器类的实现使得你能声明式地将异常映射到特定的视图上，还可以在异常被转发（forward）到对应的视图前使用Java代码做些判断和逻辑。不过在一些场景，特别是依靠`@ResponseBody`返回响应而非依赖视图解析机制的场景下，直接设置响应的状态码并将客户端需要的错误信息直接写回响应体中，可能是更方便的方法。

你也可以使用`@ExceptionHandler`方法来做到这点。如果`@ExceptionHandler`方法是在控制器内部定义的，那么它会接收并处理由控制器（或其任何子类）中的`@RequestMapping`方法抛出的异常。如果你将`@ExceptionHandler`方法定义在`@ControllerAdvice`类中，那么它会处理相关控制器中抛出的异常。下面的代码就展示了一个定义在控制器内部的`@ExceptionHandler`方法：

```
@Controller
public class SimpleController {

    // @RequestMapping methods omitted ...

    @ExceptionHandler(IOException.class)
    public ResponseEntity<String> handleIOException(IOException ex) {
        // prepare responseEntity
        return responseEntity;
    }

}

```

此外，`@ExceptionHandler`注解还可以接受一个异常类型的数组作为参数值。若抛出了已在列表中声明的异常，那么相应的`@ExceptionHandler`方法将会被调用。如果没有给注解任何参数值，那么默认处理的异常类型将是方法参数所声明的那些异常。

与标准的控制器`@RequestMapping`注解处理方法一样，`@ExceptionHandler`方法的方法参数和返回值也可以很灵活。比如，在Servlet环境下方法可以接收`HttpServletRequest`参数，而Portlet环境下方法可以接收`PortletRequest`参数。返回值可以是`String`类型——这种情况下会被解析为视图名——可以是`ModelAndView`类型的对象，也可以是`ResponseEntity`。或者你还可以在方法上添加`@ResponseBody`注解以使用消息转换器会转换信息为特定类型的数据，然后把它们写回到响应流中。

</section>







# Spring MVC 处理一般的异常



2018-07-26 14:34 更新







处理请求的过程中，Spring MVC可能会抛出一些的异常。`SimpleMappingExceptionResolver`可以根据需要很方便地将任何异常映射到一个默认的错误视图。但，如果客户端是通过自动检测响应的方式来分发处理异常的，那么后端就需要为响应设置对应的状态码。根据抛出异常的类型不同，可能需要设置不同的状态码来标识是客户端错误（4xx）还是服务器端错误（5xx）。

<section>

默认处理器异常解析器`DefaultHandlerExceptionResolver`会将Spring MVC抛出的异常转换成对应的错误状态码。该解析器在MVC命名空间配置或MVC Java配置的方式下默认已经被注册了，另外，通过`DispatcherServlet`注册也是可行的（即不使用MVC命名空间或Java编程方式进行配置的时候）。下表列出了该解析器能处理的一些异常，及他们对应的状态码。

| 异常 | HTTP状态码 |
| --- | --- |
| `BindException` | 400 (无效请求) |
| `ConversionNotSupportedException` | 500 (服务器内部错误) |
| `HttpMediaTypeNotAcceptableException` | 406 (不接受) |
| `HttpMediaTypeNotSupportedException` | 415 (不支持的媒体类型) |
| `HttpMessageNotReadableException` | 400 (无效请求) |
| `HttpMessageNotWritableException` | 500 (服务器内部错误) |
| `HttpRequestMethodNotSupportedException` | 405 (不支持的方法) |
| `MethodArgumentNotValidException` | 400 (无效请求) |
| `MissingServletRequestParameterException` | 400 (无效请求) |
| `MissingServletRequestPartException` | 400 (无效请求) |
| `NoHandlerFoundException` | 404 (请求未找到) |
| `NoSuchRequestHandlingMethodException` | 404 (请求未找到) |
| `TypeMismatchException` | 400 (无效请求) |
| `MissingPathVariableException` | 500 (服务器内部错误) |
| `NoHandlerFoundException` | 404 (请求未找到) |

以下待翻译。

The `DefaultHandlerExceptionResolver` works transparently by setting the status of the response. However, it stops short of writing any error content to the body of the response while your application may need to add developer- friendly content to every error response for example when providing a REST API. You can prepare a `ModelAndView` and render error content through view resolution?--?i.e. by configuring a `ContentNegotiatingViewResolver`, `MappingJackson2JsonView`, and so on. However, you may prefer to use`@ExceptionHandler` methods instead.

If you prefer to write error content via `@ExceptionHandler` methods you can extend `ResponseEntityExceptionHandler` instead. This is a convenient base for `@ControllerAdvice` classes providing an `@ExceptionHandler` method to handle standard Spring MVC exceptions and return `ResponseEntity`. That allows you to customize the response and write error content with message converters. See the `ResponseEntityExceptionHandler` javadocs for more details.

</section>







# Spring MVC 使用@ResponseStatus注解业务异常



2020-07-31 10:52 更新







业务异常可以使用`@ResponseStatus`来注解。当异常被抛出时，`ResponseStatusExceptionResolver`会设置相应的响应状态码。`DispatcherServlet`会默认注册一个`ResponseStatusExceptionResolver` 以供使用。

ResponseStatus注解的使用非常简单，我们创建一个异常类，加上注解

```
package com.zj.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value=HttpStatus.FORBIDDEN,reason="用户不匹配")
public class UserNotMatchException extends RuntimeException{
}
```

> ResponseStatus注解是修饰类的
> 它有两个属性，value属性是http状态码，比如404，500等。reason是错误信息

写一个目标方法抛出该异常

```
@RequestMapping("/testResponseStatus")
public String testResponseStatus(int i){
    if(i==0)
        throw new UserNotMatchException();
    return "hello";
}
```

> 使用了ResponseStatus注解之后，用户看到的异常界面正是我们自己定义的异常，而不再是一大堆用户看不懂的代码。







# Spring MVC 对Servlet默认容器错误页面的定制化



2018-07-26 14:36 更新







当响应的状态码被设置为错误状态码，并且响应体中没有内容时，Servlet容器通常会渲染一个HTML错误页。若需要定制容器默认提供的错误页，你可以在`web.xml`中定义一个错误页面`<error-page>`元素。在Servlet 3规范出来之前，该错误页元素必须被显式指定映射到一个具体的错误码或一个异常类型。从Servlet 3开始，错误页不再需要映射到其他信息了，这意味着，你指定的位置就是对Servlet容器默认错误页的自定制了。

<section>

```
<error-page>
    <location>/error</location>
</error-page>

```

这里错误页的位置所在可以是一个JSP页面，或者其他的一些URL，只要它指定容器里任意一个`@Controller`控制器下的处理器方法：

写回`HttpServletResponse`的错误信息和错误状态码可以在控制器中通过请求属性来获取：

```
@Controller
public class ErrorController {

    @RequestMapping(path = "/error", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    @ResponseBody
    public Map<String, Object> handle(HttpServletRequest request) {

        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));

        return map;
    }

}

```

或者在JSP中这么使用:

```
<%@ page contentType="application/json" pageEncoding="UTF-8"%>
{
    status:<%=request.getAttribute("javax.servlet.error.status_code") %>,
    reason:<%=request.getAttribute("javax.servlet.error.message") %>
}
```

</section>



