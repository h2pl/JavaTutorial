



# Spring MVC 使用@Controller注解定义一个控制器



2018-07-26 14:02 更新







<section>

> [Original] The `@Controller` annotation indicates that a particular class serves the role of a controller. Spring does not require you to extend any controller base class or reference the Servlet API. However, you can still reference Servlet-specific features if you need to.

`@Controller`注解表明了一个类是作为控制器的角色而存在的。Spring不要求你去继承任何控制器基类，也不要求你去实现Servlet的那套API。当然，如果你需要的话也可以去使用任何与Servlet相关的特性和设施。

> [Original] The `@Controller` annotation acts as a stereotype for the annotated class, indicating its role. The dispatcher scans such annotated classes for mapped methods and detects `@RequestMapping` annotations (see the next section).

`@Controller`注解可以认为是被标注类的原型（stereotype），表明了这个类所承担的角色。分派器（`DispatcherServlet`）会扫描所有注解了`@Controller`的类，检测其中通过`@RequestMapping`注解配置的方法（详见下一小节）。

> [Original] You can define annotated controller beans explicitly, using a standard Spring bean definition in the dispatcher’s context. However, the `@Controller` stereotype also allows for autodetection, aligned with Spring general support for detecting component classes in the classpath and auto-registering bean definitions for them.

当然，你也可以不使用`@Controller`注解而显式地去定义被注解的bean，这点通过标准的Spring bean的定义方式，在dispather的上下文属性下配置即可做到。但是`@Controller`原型是可以被框架自动检测的，Spring支持classpath路径下组件类的自动检测，以及对已定义bean的自动注册。

> [Original] To enable autodetection of such annotated controllers, you add component scanning to your configuration. Use the spring-context schema as shown in the following XML snippet:

你需要在配置中加入组件扫描的配置代码来开启框架对注解控制器的自动检测。请使用下面XML代码所示的spring-context schema：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.springframework.samples.petclinic.web"/>

    <!-- ... -->

</beans>
```

</section>







# Spring MVC 使用@RequestMapping注解映射请求路径



2018-07-26 15:29 更新







你可以使用`@RequestMapping`注解来将请求URL，如`/appointments`等，映射到整个类上或某个特定的处理器方法上。一般来说，类级别的注解负责将一个特定（或符合某种模式）的请求路径映射到一个控制器上，同时通过方法级别的注解来细化映射，即根据特定的HTTP请求方法（“GET”“POST”方法等）、HTTP请求中是否携带特定参数等条件，将请求映射到匹配的方法上。

<section>

下面这段代码示例来自Petcare，它展示了在Spring MVC中如何在控制器上使用`@RequestMapping`注解：

```
@Controller
@RequestMapping("/appointments")
public class AppointmentsController {

    private final AppointmentBook appointmentBook;

    @Autowired
    public AppointmentsController(AppointmentBook appointmentBook) {
        this.appointmentBook = appointmentBook;
    }

    @RequestMapping(method = RequestMethod.GET)
    public Map<String, Appointment> get() {
        return appointmentBook.getAppointmentsForToday();
    }

    @RequestMapping(path = "/{day}", method = RequestMethod.GET)
    public Map<String, Appointment> getForDay(@PathVariable @DateTimeFormat(iso=ISO.DATE) Date day, Model model) {
        return appointmentBook.getAppointmentsForDay(day);
    }

    @RequestMapping(path = "/new", method = RequestMethod.GET)
    public AppointmentForm getNewForm() {
        return new AppointmentForm();
    }

    @RequestMapping(method = RequestMethod.POST)
    public String add(@Valid AppointmentForm appointment, BindingResult result) {
        if (result.hasErrors()) {
            return "appointments/new";
        }
        appointmentBook.addAppointment(appointment);
        return "redirect:/appointments";
    }
}

```

在上面的示例中，许多地方都使用到了`@RequestMapping`注解。第一次使用点是作用于类级别的，它指示了所有`/appointments`开头的路径都会被映射到控制器下。`get()`方法上的`@RequestMapping`注解对请求路径进行了进一步细化：它仅接受GET方法的请求。这样，一个请求路径为`/appointments`、HTTP方法为GET的请求，将会最终进入到这个方法被处理。`add()`方法也做了类似的细化，而`getNewForm()`方法则同时注解了能够接受的请求的HTTP方法和路径。这种情况下，一个路径为`appointments/new`、HTTP方法为GET的请求将会被这个方法所处理。

`getForDay()`方法则展示了使用`@RequestMapping`注解的另一个技巧：URI模板。（关于URI模板，请见下小节）

类级别的`@RequestMapping`注解并不是必须的。不配置的话则所有的路径都是绝对路径，而非相对路径。以下的代码示例来自PetClinic，它展示了一个具有多个处理器方法的控制器：

```
@Controller
public class ClinicController {

    private final Clinic clinic;

    @Autowired
    public ClinicController(Clinic clinic) {
        this.clinic = clinic;
    }

    @RequestMapping("/")
    public void welcomeHandler() {
    }

    @RequestMapping("/vets")
    public ModelMap vetsHandler() {
        return new ModelMap(this.clinic.getVets());
    }
}

```

以上代码没有指定请求必须是GET方法还是PUT/POST或其他方法，`@RequestMapping`注解默认会映射所有的HTTP请求方法。如果仅想接收某种请求方法，请在注解中指定之`@RequestMapping(method=GET)`以缩小范围。

## @Controller和面向切面（AOP）代理

有时，我们希望在运行时使用AOP代理来装饰控制器，比如当你直接在控制器上使用`@Transactional`注解时。这种情况下，我们推荐使用类级别（在控制器上使用）的代理方式。这一般是代理控制器的默认做法。如果控制器必须实现一些接口，而该接口又不支持Spring Context的回调（比如`InitializingBean`, `*Aware`等接口），那要配置类级别的代理就必须手动配置了。比如，原来的配置文件`<tx:annotation-driven/>`需要显式配置为`<tx:annotation-driven proxy-target-/>`。

## Spring MVC 3.1中新增支持@RequestMapping的一些类

> They are recommended for use and even required to take advantage of new features in Spring MVC 3.1 and going forward.

Spring 3.1中新增了一组类用以增强`@RequestMapping`，分别是`RequestMappingHandlerMapping`和`RequestMappingHandlerAdapter`。我们推荐你用一用。有部分Spring MVC 3.1之后新增的特性，这两个注解甚至是必须的。在MVC命名空间和MVC Java编程配置方式下，这组类及其新特性默认是开启的。但若你使用其他配置方式，则该特性必须手动配置才能使用。本小节将简要介绍一下，新类相比之前的一些重要变化。

在Spring 3.1之前，框架会在两个不同的阶段分别检查类级别和方法级别的请求映射——首先，`DefaultAnnotationHanlderMapping`会先在类级别上选中一个控制器，然后再通过`AnnotationMethodHandlerAdapter`定位到具体要调用的方法。

> [Original] With the new support classes in Spring 3.1, the `RequestMappingHandlerMapping` is the only place where a decision is made about which method should process the request. Think of controller methods as a collection of unique endpoints with mappings for each method derived from type and method-level `@RequestMapping` information.

现在有了Spring 3.1后引入的这组新类，`RequestMappingHandlerMapping`成为了这两个决策实际发生的唯一一个地方。你可以把控制器中的一系列处理方法当成是一系列独立的服务节点，每个从类级别和方法级别的`@RequestMapping`注解中获取到足够请求1路径映射信息。

> [Original] This enables some new possibilities. For once a `HandlerInterceptor` or a `HandlerExceptionResolver` can now expect the Object-based handler to be a `HandlerMethod`, which allows them to examine the exact method, its parameters and associated annotations. The processing for a URL no longer needs to be split across different controllers.

这种新的处理方式带来了新的可能性。之前的`HandlerInterceptor`或`HandlerExceptionResolver`现在可以确定拿到的这个处理器肯定是一个`HandlerMethod`类型，因此它能够精确地了解这个方法的所有信息，包括它的参数、应用于其上的注解等。这样，内部对于一个URL的处理流程再也不需要分隔到不同的控制器里面去执行了。

> [Original] There are also several things no longer possible: [Original] _Select a controller first with a `SimpleUrlHandlerMapping` or `BeanNameUrlHandlerMapping` and then narrow the method based on `@RequestMapping` annotations. [Original] _Rely on method names as a fall-back mechanism to disambiguate between two `@RequestMapping` methods that don’t have an explicit path mapping URL path but otherwise match equally, e.g. by HTTP method. In the new support classes `@RequestMapping` methods have to be mapped uniquely. [Original] * Have a single default method (without an explicit path mapping) with which requests are processed if no other controller method matches more concretely. In the new support classes if a matching method is not found a 404 error is raised.

同时，也有其他的一些变化，比如有些事情就没法这么玩儿了：

*   先通过`SimpleUrlHandlerMapping`或`BeanNameUrlHandlerMapping`来拿到负责处理请求的控制器，然后通过`@RequestMapping`注解配置的信息来定位到具体的处理方法；
*   依靠方法名称来作为选择处理方法的标准。比如说，两个注解了`@RequestMapping`的方法除了方法名称拥有完全相同的URL映射和HTTP请求方法。在新版本下，`@RequestMapping`注解的方法必须具有唯一的请求映射；
*   定义一个默认方法（即没有声明路径映射），在请求路径无法被映射到控制器下更精确的方法上去时，为该请求提供默认处理。在新版本中，如果无法为一个请求找到合适的处理方法，那么一个404错误将被抛出；

> [Original] The above features are still supported with the existing support classes. However to take advantage of new Spring MVC 3.1 features you’ll need to use the new support classes.

如果使用原来的类，以上的功能还是可以做到。但是，如果要享受Spring MVC 3.1版本带来的方便特性，你就需要去使用新的类。

> [Original] ## URI Template Patterns

## URI模板

> [Original] URI templates can be used for convenient access to selected parts of a URL in a `@RequestMapping` method.

URI模板可以为快速访问`@RequestMapping`中指定的URL的一个特定的部分提供很大的便利。

> [Original] A URI Template is a URI-like string, containing one or more variable names. When you substitute values for these variables, the template becomes a URI. The proposed RFC for URI Templates defines how a URI is parameterized. For example, the URI Template `http://www.example.com/users/{userId}` contains the variable userId. Assigning the value fred to the variable yields `http://www.example.com/users/fred`.

URI模板是一个类似于URI的字符串，只不过其中包含了一个或多个的变量名。当你使用实际的值去填充这些变量名的时候，模板就退化成了一个URI。在URI模板的RFC提议中定义了一个URI是如何进行参数化的。比如说，一个这个URI模板`http://www.example.com/users/{userId}`就包含了一个变量名_userId_。将值_fred_赋给这个变量名后，它就变成了一个URI：`http://www.example.com/users/fred`。

> [Original] In Spring MVC you can use the `@PathVariable` annotation on a method argument to bind it to the value of a URI template variable:

在Spring MVC中你可以在方法参数上使用`@PathVariable`注解，将其与URI模板中的参数绑定起来：

```
@RequestMapping(path="/owners/{ownerId}", method=RequestMethod.GET)
public String findOwner(@PathVariable String ownerId, Model model) {
    Owner owner = ownerService.findOwner(ownerId);
    model.addAttribute("owner", owner);
    return "displayOwner";
}

```

> [Original] The URI Template "`/owners/{ownerId}`" specifies the variable name `ownerId`. When the controller handles this request, the value of `ownerId` is set to the value found in the appropriate part of the URI. For example, when a request comes in for `/owners/fred`, the value of `ownerId` is `fred`.

URI模板"`/owners/{ownerId}`"指定了一个变量，名为`ownerId`。当控制器处理这个请求的时候，`ownerId`的值就会被URI模板中对应部分的值所填充。比如说，如果请求的URI是`/owners/fred`，此时变量`ownerId`的值就是`fred`. `

> 为了处理`@PathVariables`注解，Spring MVC必须通过变量名来找到URI模板中相对应的变量。你可以在注解中直接声明：
>
> ```
> @RequestMapping(path="/owners/{ownerId}}", method=RequestMethod.GET)
> public String findOwner(@PathVariable("ownerId") String theOwner, Model model) {
>     // 具体的方法代码…
> }
> 
> ```
>
> 或者，如果URI模板中的变量名与方法的参数名是相同的，则你可以不必再指定一次。只要你在编译的时候留下debug信息，Spring MVC就可以自动匹配URL模板中与方法参数名相同的变量名。
>
> ```
> @RequestMapping(path="/owners/{ownerId}", method=RequestMethod.GET)
> public String findOwner(@PathVariable String ownerId, Model model) {
>     // 具体的方法代码…
> }
> 
> ```
>
> [Original] A method can have any number of `@PathVariable` annotations:

一个方法可以拥有任意数量的`@PathVariable`注解：

```
@RequestMapping(path="/owners/{ownerId}/pets/{petId}", method=RequestMethod.GET)
public String findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {
    Owner owner = ownerService.findOwner(ownerId);
    Pet pet = owner.getPet(petId);
    model.addAttribute("pet", pet);
    return "displayPet";
}

```

> [Original] When a `@PathVariable` annotation is used on a `Map<String, String>` argument, the map is populated with all URI template variables.

当`@PathVariable`注解被应用于`Map<String, String>`类型的参数上时，框架会使用所有URI模板变量来填充这个map。

> [Original] A URI template can be assembled from type and path level _@RequestMapping_ annotations. As a result the `findPet()` method can be invoked with a URL such as `/owners/42/pets/21`.

URI模板可以从类级别和方法级别的 _@RequestMapping_ 注解获取数据。因此，像这样的`findPet()`方法可以被类似于`/owners/42/pets/21`这样的URL路由并调用到：

```
_@Controller_
@RequestMapping("/owners/{ownerId}")
public class RelativePathUriTemplateController {

    @RequestMapping("/pets/{petId}")
    public void findPet(_@PathVariable_ String ownerId, _@PathVariable_ String petId, Model model) {
        // 方法实现体这里忽略
    }

}

```

> [Original] A `@PathVariable` argument can be of _any simple type_ such as int, long, Date, etc. Spring automatically converts to the appropriate type or throws a `TypeMismatchException` if it fails to do so. You can also register support for parsing additional data types. See [the section called "Method Parameters And Type Conversion"](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-ann-typeconversion) and [the section called "Customizing WebDataBinder initialization"](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-ann-webdatabinder).

`@PathVariable`可以被应用于所有 _简单类型_ 的参数上，比如int、long、Date等类型。Spring会自动地帮你把参数转化成合适的类型，如果转换失败，就抛出一个`TypeMismatchException`。如果你需要处理其他数据类型的转换，也可以注册自己的类。若需要更详细的信息可以参考[“方法参数与类型转换”一节](http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/mvc.html#mvc-ann-typeconversion)和[“定制WebDataBinder初始化过程”一节](http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/mvc.html#mvc-ann-webdatabinder)

## 带正则表达式的URI模板

> [Original] Sometimes you need more precision in defining URI template variables. Consider the URL `"/spring-web/spring-web-3.0.5.jar"`. How do you break it down into multiple parts?

有时候你可能需要更准确地描述一个URI模板的变量，比如说这个URL：`"/spring-web/spring-web-3.0.5.jar`。你要怎么把它分解成几个有意义的部分呢？

> [Original] The `@RequestMapping` annotation supports the use of regular expressions in URI template variables. The syntax is `{varName:regex}` where the first part defines the variable name and the second - the regular expression.For example:

`@RequestMapping`注解支持你在URI模板变量中使用正则表达式。语法是`{varName:regex}`，其中第一部分定义了变量名，第二部分就是你所要应用的正则表达式。比如下面的代码样例：

```
@RequestMapping("/spring-web/{symbolicName:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{extension:\\.[a-z]+}")
    public void handle(@PathVariable String version, @PathVariable String extension) {
        // 代码部分省略...
    }
}

```

## Path Patterns（不好翻，容易掉韵味）

> [Original] In addition to URI templates, the `@RequestMapping` annotation also supports Ant-style path patterns (for example, `/myPath/*.do`). A combination of URI template variables and Ant-style globs is also supported (e.g. `/owners/*/pets/{petId}`).

除了URI模板外，`@RequestMapping`注解还支持Ant风格的路径模式（如`/myPath/*.do`等）。不仅如此，还可以把URI模板变量和Ant风格的glob组合起来使用（比如`/owners/*/pets/{petId}`这样的用法等）。

## 路径样式的匹配(Path Pattern Comparison)

> [Original] When a URL matches multiple patterns, a sort is used to find the most specific match.

当一个URL同时匹配多个模板（pattern）时，我们将需要一个算法来决定其中最匹配的一个。

> [Original] A pattern with a lower count of URI variables and wild cards is considered more specific. For example `/hotels/{hotel}/*` has 1 URI variable and 1 wild card and is considered more specific than `/hotels/{hotel}/**` which as 1 URI variable and 2 wild cards.

URI模板变量的数目和通配符数量的总和最少的那个路径模板更准确。举个例子，`/hotels/{hotel}/*`这个路径拥有一个URI变量和一个通配符，而`/hotels/{hotel}/**`这个路径则拥有一个URI变量和两个通配符，因此，我们认为前者是更准确的路径模板。

> [Original] If two patterns have the same count, the one that is longer is considered more specific. For example `/foo/bar*` is longer and considered more specific than `/foo/*`.

如果两个模板的URI模板数量和通配符数量总和一致，则路径更长的那个模板更准确。举个例子，`/foo/bar*`就被认为比`/foo/*`更准确，因为前者的路径更长。

> [Original] When two patterns have the same count and length, the pattern with fewer wild cards is considered more specific. For example `/hotels/{hotel}` is more specific than `/hotels/*`.

如果两个模板的数量和长度均一致，则那个具有更少通配符的模板是更加准确的。比如，`/hotels/{hotel}`就比`/hotels/*`更精确。

> [Original] There are also some additional special rules:

除此之外，还有一些其他的规则：

> [Original] _The **default mapping pattern** `/*_`is less specific than any other pattern. For example`/api/{a}/{b}/{c}` is more specific.
>
> [Original] _A **prefix pattern** such as `/public/*_`is less specific than any other pattern that doesn't contain double wildcards. For example`/public/path3/{a}/{b}/{c}` is more specific.

*   **默认的通配模式**`/**`比其他所有的模式都更“不准确”。比方说，`/api/{a}/{b}/{c}`就比默认的通配模式`/**`要更准确
*   **前缀通配**（比如`/public/**`)被认为比其他任何不包括双通配符的模式更不准确。比如说，`/public/path3/{a}/{b}/{c}`就比`/public/**`更准确

> [Original] For the full details see `AntPatternComparator` in `AntPathMatcher`. Note that the PathMatcher can be customized (see [Section 21.16.11, "Path Matching"](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-config-path-matching) in the section on configuring Spring MVC).

更多的细节请参考这两个类：`AntPatternComparator`和`AntPathMatcher`。值得一提的是，PathMatcher类是可以配置的（见“配置Spring MVC”一节中的[路径的匹配](https://www.w3cschool.cn/spring_mvc_documentation_linesh_translation/spring_mvc_documentation_linesh_translation-cgvo27t2.html)一节)。

## 带占位符的路径模式（path patterns）

> [Original] Patterns in `@RequestMapping` annotations support ${…} placeholders against local properties and/or system properties and environment variables. This may be useful in cases where the path a controller is mapped to may need to be customized through configuration. For more information on placeholders, see the javadocs of the `PropertyPlaceholderConfigurer` class.

`@RequestMapping`注解支持在路径中使用占位符，以取得一些本地配置、系统配置、环境变量等。这个特性有时很有用，比如说控制器的映射路径需要通过配置来定制的场景。如果想了解更多关于占位符的信息，可以参考`PropertyPlaceholderConfigurer`这个类的文档。

## Suffix Pattern Matching

## 后缀模式匹配

> [Original] By default Spring MVC performs `".*"` suffix pattern matching so that a controller mapped to `/person` is also implicitly mapped to `/person.*`. This makes it easy to request different representations of a resource through the URL path (e.g. `/person.pdf`, `/person.xml`).

Spring MVC默认采用`".*"`的后缀模式匹配来进行路径匹配，因此，一个映射到`/person`路径的控制器也会隐式地被映射到`/person.*`。这使得通过URL来请求同一资源文件的不同格式变得更简单（比如`/person.pdf`，`/person.xml`）。

> [Original] Suffix pattern matching can be turned off or restricted to a set of path extensions explicitly registered for content negotiation purposes. This is generally recommended to minimize ambiguity with common request mappings such as `/person/{id}` where a dot might not represent a file extension, e.g. `/person/joe@email.com` vs `/person/joe@email.com.json)`. Furthermore as explained in the note below suffix pattern matching as well as content negotiation may be used in some circumstances to attempt malicious attacks and there are good reasons to restrict them meaningfully.

你可以关闭默认的后缀模式匹配，或者显式地将路径后缀限定到一些特定格式上for content negotiation purpose。我们推荐这样做，这样可以减少映射请求时可以带来的一些二义性，比如请求以下路径`/person/{id}`时，路径中的点号后面带的可能不是描述内容格式，比如`/person/joe@email.com` vs `/person/joe@email.com.json`。而且正如下面马上要提到的，后缀模式通配以及内容协商有时可能会被黑客用来进行攻击，因此，对后缀通配进行有意义的限定是有好处的。

> [Original] See [Section 21.16.11, "Path Matching"](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-config-path-matching) for suffix pattern matching configuration and also [Section 21.16.6, "Content Negotiation"](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-config-content-negotiation) for content negotiation configuration.

关于后缀模式匹配的配置问题，可以参考[Spring MVC路径匹配配置](https://www.w3cschool.cn/spring_mvc_documentation_linesh_translation/spring_mvc_documentation_linesh_translation-cgvo27t2.html)；关于内容协商的配置问题，可以参考[Spring MVC 内容协商"](https://www.w3cschool.cn/spring_mvc_documentation_linesh_translation/spring_mvc_documentation_linesh_translation-h8br27sx.html)的内容。

## 后缀模式匹配与RFD

> [Original] Reflected file download (RFD) attack was first described in a [paper by Trustwave](https://www.trustwave.com/Resources/SpiderLabs-Blog/Reflected-File-Download---A-New-Web-Attack-Vector/) in 2014\. The attack is similar to XSS in that it relies on input (e.g. query parameter, URI variable) being reflected in the response. However instead of inserting JavaScript into HTML, an RFD attack relies on the browser switching to perform a download and treating the response as an executable script if double-clicked based on the file extension (e.g. .bat, .cmd).

RFD(Reflected file download)攻击最先是2014年在[Trustwave的一篇论文](https://www.trustwave.com/Resources/SpiderLabs-Blog/Reflected-File-Download---A-New-Web-Attack-Vector/)中被提出的。它与XSS攻击有些相似，因为这种攻击方式也依赖于某些特征，即需要你的输入（比如查询参数，URI变量等）等也在输出（response）中以某种形式出现。不同的是，RFD攻击并不是通过在HTML中写入JavaScript代码进行，而是依赖于浏览器来跳转到下载页面，并把特定格式（比如.bat，.cmd等）的response当成是可执行脚本，双击它就会执行。

> [Original] In Spring MVC `@ResponseBody` and `ResponseEntity` methods are at risk because they can render different content types which clients can request including via URL path extensions. Note however that neither disabling suffix pattern matching nor disabling the use of path extensions for content negotiation purposes alone are effective at preventing RFD attacks.

Spring MVC的`@ResponseBody`和`ResponseEntity`方法是有风险的，因为它们会根据客户的请求——包括URL的路径后缀，来渲染不同的内容类型。因此，禁用后缀模式匹配或者禁用仅为内容协商开启的路径文件后缀名携带，都是防范RFD攻击的有效方式。

> [Original] For comprehensive protection against RFD, prior to rendering the response body Spring MVC adds a `Content-Disposition:inline;filename=f.txt` header to suggest a fixed and safe download file filename. This is done only if the URL path contains a file extension that is neither whitelisted nor explicitly registered for content negotiation purposes. However it may potentially have side effects when URLs are typed directly into a browser.

若要开启对RFD更高级的保护模式，可以在Spring MVC渲染开始请求正文之前，在请求头中增加一行配置`Content-Disposition:inline;filename=f.txt`，指定固定的下载文件的文件名。这仅在URL路径中包含了一个文件符合以下特征的拓展名时适用：该扩展名既不在信任列表（白名单）中，也没有被显式地被注册于内容协商时使用。并且这种做法还可以有一些副作用，比如，当URL是通过浏览器手动输入的时候。

> [Original] Many common path extensions are whitelisted by default. Furthermore REST API calls are typically not meant to be used as URLs directly in browsers. Nevertheless applications that use custom `HttpMessageConverter` implementations can explicitly register file extensions for content negotiation and the Content-Disposition header will not be added for such extensions. See [Section 21.16.6, "Content Negotiation"](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-config-content-negotiation).

很多常用的路径文件后缀默认是被信任的。另外，REST的API一般是不应该直接用做URL的。不过，你可以自己定制`HttpMessageConverter`的实现，然后显式地注册用于内容协商的文件类型，这种情形下Content-Disposition头将不会被加入到请求头中。详见[Spring MVC 内容协商](https://www.w3cschool.cn/spring_mvc_documentation_linesh_translation/spring_mvc_documentation_linesh_translation-h8br27sx.html)。

> [Original] This was originally introduced as part of work for [CVE-2015-5211](http://pivotal.io/security/cve-2015-5211). Below are additional recommendations from the report:
>
> *   Encode rather than escape JSON responses. This is also an OWASP XSS recommendation. For an example of how to do that with Spring see [spring-jackson-owasp](https://github.com/rwinch/spring-jackson-owasp).
> *   Configure suffix pattern matching to be turned off or restricted to explicitly registered suffixes only.
> *   Configure content negotiation with the properties "useJaf" and "ignoreUnknownPathExtensions" set to false which would result in a 406 response for URLs with unknown extensions. Note however that this may not be an option if URLs are naturally expected to have a dot towards the end.
> *   Add `X-Content-Type-Options: nosniff` header to responses. Spring Security 4 does this by default.

感觉这节的翻译质量还有限，需要继续了解XSS攻击和RFD攻击的细节再翻。

## 矩阵变量

> [Original] The URI specification [RFC 3986](http://tools.ietf.org/html/rfc3986#section-3.3) defines the possibility of including name-value pairs within path segments. There is no specific term used in the spec. The general "URI path parameters" could be applied although the more unique ["Matrix URIs"](http://www.w3.org/DesignIssues/MatrixURIs.html), originating from an old post by Tim Berners-Lee, is also frequently used and fairly well known. Within Spring MVC these are referred to as matrix variables.

原来的URI规范[RFC 3986](http://tools.ietf.org/html/rfc3986#section-3.3)中允许在路径段落中携带键值对，但规范没有明确给这样的键值对定义术语。有人叫“URI路径参数”，也有叫[“矩阵URI”](http://www.w3.org/DesignIssues/MatrixURIs.html)的。后者是Tim Berners-Lee首先在其博客中提到的术语，被使用得要更加频繁一些，知名度也更高些。而在Spring MVC中，我们称这样的键值对为矩阵变量。

> [Original] Matrix variables can appear in any path segment, each matrix variable separated with a ";" (semicolon). For example: `"/cars;color=red;year=2012"`. Multiple values may be either "," (comma) separated `"color=red,green,blue"` or the variable name may be repeated `"color=red;color=green;color=blue"`.

矩阵变量可以在任何路径段落中出现，每对矩阵变量之间使用一个分号“;”隔开。比如这样的URI：`"/cars;color=red;year=2012"`。多个值可以用逗号隔开`"color=red,green,blue"`，或者重复变量名多次`"color=red;color=green;color=blue"`。

> [Original] If a URL is expected to contain matrix variables, the request mapping pattern must represent them with a URI template. This ensures the request can be matched correctly regardless of whether matrix variables are present or not and in what order they are provided.

如果一个URL有可能需要包含矩阵变量，那么在请求路径的映射配置上就需要使用URI模板来体现这一点。这样才能确保请求可以被正确地映射，而不管矩阵变量在URI中是否出现、出现的次序是怎样等。

> [Original] Below is an example of extracting the matrix variable "q":

下面是一个例子，展示了我们如何从矩阵变量中获取到变量“q”的值：

```
// GET /pets/42;q=11;r=22

@RequestMapping(path = "/pets/{petId}", method = RequestMethod.GET)
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11

}

```

> [Original] Since all path segments may contain matrix variables, in some cases you need to be more specific to identify where the variable is expected to be:

由于任意路径段落中都可以含有矩阵变量，在某些场景下，你需要用更精确的信息来指定一个矩阵变量的位置：

```
// GET /owners/42;q=11/pets/21;q=22

@RequestMapping(path = "/owners/{ownerId}/pets/{petId}", method = RequestMethod.GET)
public void findPet(
    @MatrixVariable(name="q", pathVar="ownerId") int q1,
    @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22

}

```

> [Original] A matrix variable may be defined as optional and a default value specified:

你也可以声明一个矩阵变量不是必须出现的，并给它赋一个默认值：

```
// GET /pets/42

@RequestMapping(path = "/pets/{petId}", method = RequestMethod.GET)
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1

}

```

> [Original] All matrix variables may be obtained in a Map:

也可以通过一个Map来存储所有的矩阵变量：

```
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@RequestMapping(path = "/owners/{ownerId}/pets/{petId}", method = RequestMethod.GET)
public void findPet(
    @MatrixVariable Map<String, String> matrixVars,
    @MatrixVariable(pathVar="petId") Map<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 11, "s" : 23]

}

```

> [Original] Note that to enable the use of matrix variables, you must set the `removeSemicolonContent`property of `RequestMappingHandlerMapping` to `false`. By default it is set to `true`.

如果要允许矩阵变量的使用，你必须把`RequestMappingHandlerMapping`类的`removeSemicolonContent`属性设置为`false`。该值默认是`true`的。

> [Original] The MVC Java config and the MVC namespace both provide options for enabling the use of matrix variables.
>
> MVC的Java编程配置和命名空间配置都提供了启用矩阵变量的方式。
>
> [Original] If you are using Java config, The [Advanced Customizations with MVC Java Config](https://linesh.gitbooks.io/spring-mvc-documentation-linesh-translation/content/publish/21-3/mvc.html#mvc-config-advanced-java) section describes how the `RequestMappingHandlerMapping` can be customized.
>
> 如果你是使用Java编程的方式，[“MVC Java高级定制化配置”一节](http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/mvc.html#mvc-config-advanced-java)描述了如何对`RequestMappingHandlerMapping`进行定制。
>
> [Original] In the MVC namespace, the `<mvc:annotation-driven>` element has an `enable-matrix-variables` attribute that should be set to `true`. By default it is set to `false`.
>
> 而使用MVC的命名空间配置时，你可以把`<mvc:annotation-driven>`元素下的`enable-matrix-variables`属性设置为`true`。该值默认情况下是配置为`false`的。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven enable-matrix-variables="true"/>

</beans>

```

## 可消费的媒体类型

> [Original] You can narrow the primary mapping by specifying a list of consumable media types. The request will be matched only if the _Content-Type_ request header matches the specified media type. For example:

你可以指定一组可消费的媒体类型，缩小映射的范围。这样只有当请求头中 _Content-Type_ 的值与指定可消费的媒体类型中有相同的时候，请求才会被匹配。比如下面这个例子：

```
@Controller
@RequestMapping(path = "/pets", method = RequestMethod.POST, consumes="application/json")
public void addPet(@RequestBody Pet pet, Model model) {
    // 方法实现省略
}

```

> [Original] Consumable media type expressions can also be negated as in _!text/plain_ to match to all requests other than those with _Content-Type_ of _text/plain_. Also consider using constants provided in `MediaType` such as `APPLICATION_JSON_VALUE` and `APPLICATION_JSON_UTF8_VALUE`.

指定可消费媒体类型的表达式中还可以使用否定，比如，可以使用 _!text/plain_ 来匹配所有请求头 _Content-Type_ 中不含 _text/plain_ 的请求。同时，在`MediaType`类中还定义了一些常量，比如`APPLICATION_JSON_VALUE`、`APPLICATION_JSON_UTF8_VALUE`等，推荐更多地使用它们。

> [Original] The _consumes_ condition is supported on the type and on the method level. Unlike most other conditions, when used at the type level, method-level consumable types override rather than extend type-level consumable types.
>
> _consumes_ 属性提供的是方法级的类型支持。与其他属性不同，当在类型级使用时，方法级的消费类型将覆盖类型级的配置，而非继承关系。

## 可生产的媒体类型

> [Original] You can narrow the primary mapping by specifying a list of producible media types. The request will be matched only if the _Accept_ request header matches one of these values. Furthermore, use of the _produces_ condition ensures the actual content type used to generate the response respects the media types specified in the _produces_ condition. For example:

你可以指定一组可生产的媒体类型，缩小映射的范围。这样只有当请求头中 _Accept_ 的值与指定可生产的媒体类型中有相同的时候，请求才会被匹配。而且，使用 _produces_ 条件可以确保用于生成响应（response）的内容与指定的可生产的媒体类型是相同的。举个例子：

```
@Controller
@RequestMapping(path = "/pets/{petId}", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
@ResponseBody
public Pet getPet(@PathVariable String petId, Model model) {
    // 方法实现省略
}

```

> [Original] Be aware that the media type specified in the _produces_ condition can also optionally specify a character set. For example, in the code snippet above we specify the same media type than the default one configured in `MappingJackson2HttpMessageConverter`, including the `UTF-8`charset.
>
> 要注意的是，通过 _condition_ 条件指定的媒体类型也可以指定字符集。比如在上面的小段代码中，我们还是覆写了`MappingJackson2HttpMessageConverter`类中默认配置的媒体类型，同时，还指定了使用`UTF-8`的字符集。
>
> [Original] Just like with _consumes_, producible media type expressions can be negated as in _!text/plain_ to match to all requests other than those with an _Accept_ header value of _text/plain_. Also consider using constants provided in `MediaType` such as `APPLICATION_JSON_VALUE` and `APPLICATION_JSON_UTF8_VALUE`.

与 _consumes_ 条件类似，可生产的媒体类型表达式也可以使用否定。比如，可以使用 _!text/plain_ 来匹配所有请求头 _Accept_ 中不含 _text/plain_ 的请求。同时，在`MediaType`类中还定义了一些常量，比如`APPLICATION_JSON_VALUE`、`APPLICATION_JSON_UTF8_VALUE`等，推荐更多地使用它们。

> [Original] The _produces_ condition is supported on the type and on the method level. Unlike most other conditions, when used at the type level, method-level producible types override rather than extend type-level producible types.
>
> _produces_ 属性提供的是方法级的类型支持。与其他属性不同，当在类型级使用时，方法级的消费类型将覆盖类型级的配置，而非继承关系。

## 请求参数与请求头的值

> [Original] You can narrow request matching through request parameter conditions such as `"myParam"`, `"!myParam"`, or `"myParam=myValue"`. The first two test for request parameter presence/absence and the third for a specific parameter value. Here is an example with a request parameter value condition:

你可以筛选请求参数的条件来缩小请求匹配范围，比如`"myParam"`、`"!myParam"`及`"myParam=myValue"`等。前两个条件用于筛选存在/不存在某些请求参数的请求，第三个条件筛选具有特定参数值的请求。下面有个例子，展示了如何使用请求参数值的筛选条件：

```
@Controller
@RequestMapping("/owners/{ownerId}")
public class RelativePathUriTemplateController {

    @RequestMapping(path = "/pets/{petId}", method = RequestMethod.GET, params="myParam=myValue")
    public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {
        // 实际实现省略
    }

}

```

> [Original] The same can be done to test for request header presence/absence or to match based on a specific request header value:

同样，你可以用相同的条件来筛选请求头的出现与否，或者筛选出一个具有特定值的请求头：

```
@Controller
@RequestMapping("/owners/{ownerId}")
public class RelativePathUriTemplateController {

    @RequestMapping(path = "/pets", method = RequestMethod.GET, headers="myHeader=myValue")
    public void findPet(@PathVariable String ownerId, @PathVariable String petId, Model model) {
        // 方法体实现省略
    }

}

```

> [Original] Although you can match to _Content-Type_ and _Accept_ header values using media type wild cards (for example _"content-type=text/*"_ will match to _"text/plain"_ and _"text/html"_), it is recommended to use the _consumes_ and _produces_ conditions respectively instead. They are intended specifically for that purpose.
>
> 尽管，你可以使用媒体类型的通配符（比如 _"content-type=text/*"_）来匹配请求头 _Content-Type_和 _Accept_的值，但我们更推荐独立使用 _consumes_和 _produces_条件来筛选各自的请求。因为它们就是专门为区分这两种不同的场景而生的。

</section>



