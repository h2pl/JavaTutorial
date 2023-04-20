



# Spring MVC 文件上传概述










Spring内置对多路上传的支持，专门用于处理web应用中的文件上传。你可以通过注册一个可插拔的`MultipartResolver`对象来启用对文件多路上传的支持。该接口在定义于`org.springframework.web.multipart`包下。Spring为[_一般的文件上传_](http://jakarta.apache.org/commons/fileupload)提供了`MultipartResolver`接口的一个实现，为Servlet 3.0多路请求的转换提供了另一个实现。

<section>

默认情况下，Spring的多路上传支持是不开启的，因为有些开发者希望由自己来处理多路请求。如果想启用Spring的多路上传支持，你需要在web应用的上下文中添加一个多路传输解析器。每个进来的请求，解析器都会检查是不是一个多部分请求。若发现请求是完整的，则请求按正常流程被处理；如果发现请求是一个多路请求，则你在上下文中注册的`MultipartResolver`解析器会被用来处理该请求。之后，请求中的多路上传属性就与其他属性一样被正常对待了。【最后一句翻的不好，multipart翻译成多路还是多部分还在斟酌中。望阅读者注意此处。】

</section>







# Spring MVC 使用MultipartResolver与Commons FileUpload传输文件



2018-07-26 14:28 更新







下面的代码展示了如何使用一个通用的多路上传解析器`CommonsMultipartResolver`：

<section>

```
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">

    <!-- 支持的其中一个属性，支持的最大文件大小，以字节为单位 -->
    <property name="maxUploadSize" value="100000"/>

</bean>

```

当然，要让多路解析器正常工作，你需要在classpath路径下准备必须的jar包。如果使用的是通用的多路上传解析器`CommonsMultipartResolver`，你所需要的jar包是`commons-fileupload.jar`。

当Spring的`DispatcherServlet`检测到一个多部分请求时，它会激活你在上下文中声明的多路解析器并把请求交给它。解析器会把当前的`HttpServletRequest`请求对象包装成一个支持多路文件上传的请求对象`MultipartHttpServletRequest`。有了`MultipartHttpServletRequest`对象，你不仅可以获取该多路请求中的信息，还可以在你的控制器中获得该多路请求的内容本身。

</section>







# Spring MVC 处理Servlet 3.0下的MultipartResolver



2018-07-26 14:29 更新







要使用基于Servlet 3.0的多路传输转换功能，你必须在`web.xml`中为`DispatcherServlet`添加一个`multipart-config`元素，或者通过Servlet编程的方法使用`javax.servlet.MultipartConfigElement`进行注册，或你自己定制了自己的Servlet类，那你必须使用`javax.servlet.annotation.MultipartConfig`对其进行注解。其他诸如最大文件大小或存储位置等配置选项都必须在这个Servlet级别进行注册，因为Servlet 3.0不允许在解析器MultipartResolver的层级配置这些信息。

<section>

当你通过以上任一种方式启用了Servlet 3.0多路传输转换功能，你就可以把一个`StandardServletMultipartResolver`解析器添加到你的Spring配置中去了：

```
<bean id="multipartResolver" class="org.springframework.web.multipart.support.StandardServletMultipartResolver">
</bean>
```

</section>






# Spring MVC 处理表单中的文件上传



2018-07-26 14:30 更新







当解析器`MultipartResolver`完成处理时，请求便会像其他请求一样被正常流程处理。首先，创建一个接受文件上传的表单将允许用于直接上传整个表单。编码属性（`enctype="multipart/form-data"`）能让浏览器知道如何对多路上传请求的表单进行编码（encode）。

<section>

```
<html>
    <head>
        <title>Upload a file please</title>
    </head>
    <body>
        <h1>Please upload a file</h1>
        <form method="post" action="/form" enctype="multipart/form-data">
            
            
            
        </form>
    </body>
</html>

```

下一步是创建一个能处理文件上传的控制器。这里需要的控制器与[一般注解了`@Controller`的控制器](http://docs.spring.io/spring-framework/docs/4.2.4.RELEASE/spring-framework-reference/html/mvc.html#mvc-ann-controller)基本一样，除了它接受的方法参数类型是`MultipartHttpServletRequest`，或`MultipartFile`。

```
@Controller
public class FileUploadController {

    @RequestMapping(path = "/form", method = RequestMethod.POST)
    public String handleFormUpload(@RequestParam("name") String name, @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }

        return "redirect:uploadFailure";
    }

}

```

请留意`@RequestParam`注解是如何将方法参数对应到表单中的定义的输入字段的。在上面的例子中，我们拿到了`byte[]`文件数据，只是没对它做任何事。在实际应用中，你可能会将它保存到数据库、存储在文件系统上，或做其他的处理。

当使用Servlet 3.0的多路传输转换时，你也可以使用`javax.servlet.http.Part`作为方法参数：

```
@Controller
public class FileUploadController {

    @RequestMapping(path = "/form", method = RequestMethod.POST)
    public String handleFormUpload(@RequestParam("name") String name, @RequestParam("file") Part file) {

        InputStream inputStream = file.getInputStream();
        // store bytes from uploaded file somewhere

        return "redirect:uploadSuccess";
    }

}
```

</section>







# Spring MVC 处理客户端发起的文件上传请求



2018-07-26 14:30 更新







在使用了RESTful服务的场景下，非浏览器的客户端也可以直接提交多路文件请求。上一节讲述的所有例子与配置在这里也都同样适用。但与浏览器不同的是，提交的文件和简单的表单字段，客户端发送的数据可以更加复杂，数据可以指定为某种特定的内容类型（content type）——比如，一个多路上传请求可能第一部分是个文件，而第二部分是个JSON格式的数据：

<section>

```
    POST /someUrl
    Content-Type: multipart/mixed

    --edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
    Content-Disposition: form-data; name="meta-data"
    Content-Type: application/json; charset=UTF-8
    Content-Transfer-Encoding: 8bit

    {
        "name": "value"
    }
    --edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
    Content-Disposition: form-data; name="file-data"; filename="file.properties"
    Content-Type: text/xml
    Content-Transfer-Encoding: 8bit
    ... File Data ...

```

对于名称为`meta-data`的部分，你可以通过控制器方法上的`@RequestParam("meta-data") String metadata`参数来获得。但对于那部分请求体中为JSON格式数据的请求，你可能更想通过接受一个对应的强类型对象，就像`@RequestBody`通过`HttpMessageConverter`将一般请求的请求体转换成一个对象一样。

这是可能的，你可以使用`@RequestPart`注解来实现，而非`@RequestParam`。该注解将使得特定多路请求的请求体被传给`HttpMessageConverter`，并且在转换时考虑多路请求中不同的内容类型参数`'Content-Type'`：

```
@RequestMapping(path = "/someUrl", method = RequestMethod.POST)
public String onSubmit(@RequestPart("meta-data") MetaData metadata, @RequestPart("file-data") MultipartFile file) {

    // ...

}

```

请注意`MultipartFile`方法参数是如何能够在`@RequestParam`或`@RequestPart`注解下互用的，两种方法都能拿到数据。但，这里的方法参数`@RequestPart("meta-data") MetaData`则会因为请求中的内容类型请求头`'Content-Type'`被读入成为JSON数据，然后再通过`MappingJackson2HttpMessageConverter`被转换成特定的对象。

</section>




