#### 国际化

* * *

在开发应用程序的时候，经常会遇到支持多语言的需求，这种支持多语言的功能称之为国际化，英文是internationalization，缩写为i18n（因为首字母i和末字母n中间有18个字母）。

还有针对特定地区的本地化功能，英文是localization，缩写为L10n，本地化是指根据地区调整类似姓名、日期的显示等。

也有把上面两者合称为全球化，英文是globalization，缩写为g11n。

在Java中，支持多语言和本地化是通过`MessageFormat`配合`Locale`实现的：


对于Web应用程序，要实现国际化功能，主要是渲染View的时候，要把各种语言的资源文件提出来，这样，不同的用户访问同一个页面时，显示的语言就是不同的。

我们来看看在Spring MVC应用程序中如何实现国际化。

### 获取Locale

实现国际化的第一步是获取到用户的`Locale`。在Web应用程序中，HTTP规范规定了浏览器会在请求中携带`Accept-Language`头，用来指示用户浏览器设定的语言顺序，如：

```  
Accept-Language: zh-CN,zh;q=0.8,en;q=0.2  
  
```

上述HTTP请求头表示优先选择简体中文，其次选择中文，最后选择英文。`q`表示权重，解析后我们可获得一个根据优先级排序的语言列表，把它转换为Java的`Locale`，即获得了用户的`Locale`。大多数框架通常只返回权重最高的`Locale`。

Spring MVC通过`LocaleResolver`来自动从`HttpServletRequest`中获取`Locale`。有多种`LocaleResolver`的实现类，其中最常用的是`CookieLocaleResolver`：

```  
@Primary  
@Bean  
LocaleResolver createLocaleResolver() {  
    var clr = new CookieLocaleResolver();    clr.setDefaultLocale(Locale.ENGLISH);    clr.setDefaultTimeZone(TimeZone.getDefault());    return clr;}  
  
```

`CookieLocaleResolver`从`HttpServletRequest`中获取`Locale`时，首先根据一个特定的Cookie判断是否指定了`Locale`，如果没有，就从HTTP头获取，如果还没有，就返回默认的`Locale`。

当用户第一次访问网站时，`CookieLocaleResolver`只能从HTTP头获取`Locale`，即使用浏览器的默认语言。通常网站也允许用户自己选择语言，此时，`CookieLocaleResolver`就会把用户选择的语言存放到Cookie中，下一次访问时，就会返回用户上次选择的语言而不是浏览器默认语言。

### 提取资源文件

第二步是把写死在模板中的字符串以资源文件的方式存储在外部。对于多语言，主文件名如果命名为`messages`，那么资源文件必须按如下方式命名并放入classpath中：

*   默认语言，文件名必须为`messages.properties`；
*   简体中文，Locale是`zh_CN`，文件名必须为`messages_zh_CN.properties`；
*   日文，Locale是`ja_JP`，文件名必须为`messages_ja_JP.properties`；
*   其它更多语言……

每个资源文件都有相同的key，例如，默认语言是英文，文件`messages.properties`内容如下：

```  
language.select=Language  
home=Home  
signin=Sign In  
copyright=Copyright?{0,number,#}  
  
```

文件`messages_zh_CN.properties`内容如下：

```  
language.select=语言  
home=首页  
signin=登录  
copyright=版权所有?{0,number,#}  
  
```

### 创建MessageSource

第三步是创建一个Spring提供的`MessageSource`实例，它自动读取所有的`.properties`文件，并提供一个统一接口来实现“翻译”：

```  
// code, arguments, locale:  
String text = messageSource.getMessage("signin", null, locale);  
  
```

其中，`signin`是我们在`.properties`文件中定义的key，第二个参数是`Object[]`数组作为格式化时传入的参数，最后一个参数就是获取的用户`Locale`实例。

创建`MessageSource`如下：

```  
@Bean("i18n")  
MessageSource createMessageSource() {  
    var messageSource = new ResourceBundleMessageSource();    // 指定文件是UTF-8编码:  
    messageSource.setDefaultEncoding("UTF-8");    // 指定主文件名:  
    messageSource.setBasename("messages");    return messageSource;}  
  
```

注意到`ResourceBundleMessageSource`会自动根据主文件名自动把所有相关语言的资源文件都读进来。

再注意到Spring容器会创建不只一个`MessageSource`实例，我们自己创建的这个`MessageSource`是专门给页面国际化使用的，因此命名为`i18n`，不会与其它`MessageSource`实例冲突。

### 实现多语言

要在View中使用`MessageSource`加上`Locale`输出多语言，我们通过编写一个`MvcInterceptor`，把相关资源注入到`ModelAndView`中：

```  
@Component  
public class MvcInterceptor implements HandlerInterceptor {  
    @Autowired    LocaleResolver localeResolver;  
    // 注意注入的MessageSource名称是i18n:  
    @Autowired    @Qualifier("i18n")    MessageSource messageSource;  
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {        if (modelAndView != null) {            // 解析用户的Locale:  
            Locale locale = localeResolver.resolveLocale(request);            // 放入Model:  
            modelAndView.addObject("__messageSource__", messageSource);            modelAndView.addObject("__locale__", locale);        }    }}  
  
```

不要忘了在`WebMvcConfigurer`中注册`MvcInterceptor`。现在，就可以在View中调用`MessageSource.getMessage()`方法来实现多语言：

```  
{{ __messageSource__.getMessage('signin', null, __locale__) }}  
  
```

上述这种写法虽然可行，但格式太复杂了。使用View时，要根据每个特定的View引擎定制国际化函数。在Pebble中，我们可以封装一个国际化函数，名称就是下划线`_`，改造一下创建`ViewResolver`的代码：

```  
@Bean  
ViewResolver createViewResolver(@Autowired ServletContext servletContext, @Autowired @Qualifier("i18n") MessageSource messageSource) {  
    var engine = new PebbleEngine.Builder()            .autoEscaping(true)            .cacheActive(false)            .loader(new Servlet5Loader(servletContext))            // 添加扩展:  
            .extension(createExtension(messageSource))            .build();    var viewResolver = new PebbleViewResolver();    viewResolver.setPrefix("/WEB-INF/templates/");    viewResolver.setSuffix("");    viewResolver.setPebbleEngine(engine);    return viewResolver;}  
  
private Extension createExtension(MessageSource messageSource) {  
    return new AbstractExtension() {        @Override        public Map<String, Function> getFunctions() {            return Map.of("_", new Function() {                public Object execute(Map<String, Object> args, PebbleTemplate self, EvaluationContext context, int lineNumber) {                    String key = (String) args.get("0");                    List<Object> arguments = this.extractArguments(args);                    Locale locale = (Locale) context.getVariable("__locale__");                    return messageSource.getMessage(key, arguments.toArray(), "???" + key + "???", locale);                }                private List<Object> extractArguments(Map<String, Object> args) {                    int i = 1;                    List<Object> arguments = new ArrayList<>();                    while (args.containsKey(String.valueOf(i))) {                        Object param = args.get(String.valueOf(i));                        arguments.add(param);                        i++;                    }                    return arguments;                }                public List<String> getArgumentNames() {                    return null;                }            });        }    };}  
  
```

这样，我们可以把多语言页面改写为：

```  
{{ _('signin') }}  
  
```

如果是带参数的多语言，需要把参数传进去：

```  
<h5>{{ _('copyright', 2020) }}</h5>  
  
```

使用其它View引擎时，也应当根据引擎接口实现更方便的语法。

### 切换Locale

最后，我们需要允许用户手动切换`Locale`，编写一个`LocaleController`来实现该功能：

```  
@Controller  
public class LocaleController {  
    final Logger logger = LoggerFactory.getLogger(getClass());  
    @Autowired    LocaleResolver localeResolver;  
    @GetMapping("/locale/{lo}")    public String setLocale(@PathVariable("lo") String lo, HttpServletRequest request, HttpServletResponse response) {        // 根据传入的lo创建Locale实例:  
        Locale locale = null;        int pos = lo.indexOf('_');        if (pos > 0) {            String lang = lo.substring(0, pos);            String country = lo.substring(pos + 1);            locale = new Locale(lang, country);        } else {            locale = new Locale(lo);        }        // 设定此Locale:  
        localeResolver.setLocale(request, response, locale);        logger.info("locale is set to {}.", locale);        // 刷新页面:  
        String referer = request.getHeader("Referer");        return "redirect:" + (referer == null ? "/" : referer);    }}  
  
```

在页面设计中，通常在右上角给用户提供一个语言选择列表，来看看效果：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416194544.png)

切换到中文：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416194602.png)

### 小结

多语言支持需要从HTTP请求中解析用户的Locale，然后针对不同Locale显示不同的语言；

Spring MVC应用程序通过`MessageSource`和`LocaleResolver`，配合View实现国际化。