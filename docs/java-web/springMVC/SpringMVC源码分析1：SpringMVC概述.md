# Table of Contents

  * [Web MVC简介](#web-mvc简介)
    * [Web开发中的请求-响应模型：](#web开发中的请求-响应模型：)
    * [标准MVC模型概述](#标准mvc模型概述)
    * [Web MVC概述](#web-mvc概述)
    * [Web端开发发展历程](#web端开发发展历程)
  * [Spring Web MVC是什么](#spring-web-mvc是什么)
  * [Spring Web MVC能帮我们做什么](#spring-web-mvc能帮我们做什么)
  * [Spring Web MVC架构](#spring-web-mvc架构)
    * [Spring Web MVC处理请求的流程](#spring-web-mvc处理请求的流程)
    * [Spring Web MVC架构](#spring-web-mvc架构-1)
  * [Spring Web MVC优势](#spring-web-mvc优势)
  * [DispatcherServlet作用](#dispatcherservlet作用)


转自：[跟开涛学SpringMVC](http://jinnianshilongnian.iteye.com/category/231099)

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







## Web MVC简介





### Web开发中的请求-响应模型：





![](http://sishuok.com/forum/upload/2012/7/1/2fc5edc55fa262fddccecc1816f5ff7b__1.JPG)





在Web世界里，具体步骤如下：





1、  Web浏览器（如IE）发起请求，如访问[http://sishuok.com](http://sishuok.com/)





2、  Web服务器（如Tomcat）接收请求，处理请求（比如用户新增，则将把用户保存一下），最后产生响应（一般为html）。





3、web服务器处理完成后，返回内容给web客户端（一般就是我们的浏览器），客户端对接收的内容进行处理（如web浏览器将会对接收到的html内容进行渲染以展示给客户）。





**因此，在Web世界里：**





都是Web客户端发起请求，Web服务器接收、处理并产生响应。





一般Web服务器是不能主动通知Web客户端更新内容。虽然现在有些技术如服务器推（如Comet）、还有现在的HTML5 websocket可以实现Web服务器主动通知Web客户端。





到此我们了解了在web开发时的请求/响应模型，接下来我们看一下标准的MVC模型是什么。





### 标准MVC模型概述





**MVC模型：**是一种架构型的模式，本身不引入新功能，只是帮助我们将开发的结构组织的更加合理，使展示与模型分离、流程控制逻辑、业务逻辑调用与展示逻辑分离。如图1-2





![](http://sishuok.com/forum/upload/2012/7/1/a633b31a42f0224c9bf66cd3cc886e04__2.JPG)





图1-2





**首先让我们了解下MVC（Model-View-Controller）三元组的概念：**





**Model（模型）：**数据模型，提供要展示的数据，因此包含数据和行为，可以认为是领域模型或JavaBean组件（包含数据和行为），不过现在一般都分离开来：Value Object（数据） 和 服务层（行为）。也就是模型提供了模型数据查询和模型数据的状态更新等功能，包括数据和业务。





**View（视图）：**负责进行模型的展示，一般就是我们见到的用户界面，客户想看到的东西。





**Controller（控制器）：**接收用户请求，委托给模型进行处理（状态改变），处理完毕后把返回的模型数据返回给视图，由视图负责展示。 也就是说控制器做了个调度员的工作，。





从图1-1我们还看到，在标准的MVC中模型能主动推数据给视图进行更新（观察者设计模式，在模型上注册视图，当模型更新时自动更新视图），但在Web开发中模型是无法主动推给视图（无法主动更新用户界面），因为在Web开发是请求-响应模型。





那接下来我们看一下在Web里MVC是什么样子，我们称其为 Web MVC 来区别标准的MVC。





### Web MVC概述





模型-视图-控制器概念和标准MVC概念一样，请参考1.2，我们再看一下Web MVC标准架构，如图1-3：





![](http://sishuok.com/forum/upload/2012/7/1/baa1df353ed98b79231b535bc1f73dea__3.JPG)





如图1-3





在Web MVC模式下，模型无法主动推数据给视图，如果用户想要视图更新，需要再发送一次请求（即请求-响应模型）。





概念差不多了，我们接下来了解下Web端开发的发展历程，和使用代码来演示一下Web MVC是如何实现的，还有为什么要使用MVC这个模式呢？





### Web端开发发展历程





此处我们只是简单的叙述比较核心的历程，如图1-4





![](http://sishuok.com/forum/upload/2012/7/1/41f193e4f961be27d511789df2ee7680__4.JPG)





图1-4





**1.4.1、CGI**：（Common Gateway Interface）公共网关接口，一种在web服务端使用的脚本技术，使用C或Perl语言编写，用于接收web用户请求并处理，最后动态产生响应给用户，但每次请求将产生一个进程，重量级。





**1.4.2、Servlet**：一种JavaEE web组件技术，是一种在服务器端执行的web组件，用于接收web用户请求并处理，最后动态产生响应给用户。但每次请求只产生一个线程（而且有线程池），轻量级。而且能利用许多JavaEE技术（如JDBC等）。本质就是在java代码里面 输出 html流。但表现逻辑、控制逻辑、业务逻辑调用混杂。如图1-5





![](http://sishuok.com/forum/upload/2012/7/1/799db5c79c85cb59b68b915916f8dddc__5.JPG "点击查看原始大小图片")





图1-5





如图1-5，这种做法是绝对不可取的，控制逻辑、表现代码、业务逻辑对象调用混杂在一起，最大的问题是直接在Java代码里面输出Html，这样前端开发人员无法进行页面风格等的设计与修改，即使修改也是很麻烦，因此实际项目这种做法不可取。





**1.4.3、JSP：**（Java Server Page）：一种在服务器端执行的web组件，是一种运行在标准的HTML页面中嵌入脚本语言（现在只支持Java）的模板页面技术。本质就是在html代码中嵌入java代码。JSP最终还是会被编译为Servlet，只不过比纯Servlet开发页面更简单、方便。但表现逻辑、控制逻辑、业务逻辑调用还是混杂。如图1-6





![](http://sishuok.com/forum/upload/2012/7/1/7486df5510b6068a360d8f0c6bbb706c__6.JPG "点击查看原始大小图片")





图1-6





如图1-6，这种做法也是绝对不可取的，控制逻辑、表现代码、业务逻辑对象调用混杂在一起，但比直接在servlet里输出html要好一点，前端开发人员可以进行简单的页面风格等的设计与修改（但如果嵌入的java脚本太多也是很难修改的），因此实际项目这种做法不可取。





![](http://sishuok.com/forum/upload/2012/7/1/c59e61daa2ed98f75aab1ae3397fb235__7.JPG)





JSP本质还是Servlet，最终在运行时会生成一个Servlet（如tomcat，将在tomcat\work\Catalina\web应用名\org\apache\jsp下生成），但这种使得写html简单点，但仍是控制逻辑、表现代码、业务逻辑对象调用混杂在一起。





**1.4.4、Model1：**可以认为是JSP的增强版，可以认为是jsp+javabean如图1-7





特点：使用jsp:useBean标准动作，自动将请求参数封装为JavaBean组件；还必须使用java脚本执行控制逻辑。





![](http://sishuok.com/forum/upload/2012/7/1/c25b9ce9064a37ab93ffe02711b3ecc7__8.JPG "点击查看原始大小图片")





图1-7





此处我们可以看出，使用jsp:useBean标准动作可以简化javabean的获取/创建，及将请求参数封装到javabean，再看一下Model1架构，如图1-8。





![](http://sishuok.com/forum/upload/2012/7/1/d3b5b2ec88706fff7ef4b98d110837d1__9.JPG "点击查看原始大小图片")





图1-8 Model1架构





Model1架构中，JSP负责控制逻辑、表现逻辑、业务对象（javabean）的调用，只是比纯JSP简化了获取请求参数和封装请求参数。同样是不好的，在项目中应该严禁使用（或最多再demo里使用）。





**1.4.5、Model2：**在JavaEE世界里，它可以认为就是**Web MVC**模型





Model2架构其实可以认为就是我们所说的Web MVC模型，只是控制器采用Servlet、模型采用JavaBean、视图采用JSP，如图1-9





![](http://sishuok.com/forum/upload/2012/7/1/a6b7b2ca293e610a7a2b32e47a16d718__10.JPG "点击查看原始大小图片")





图1-9 Model2架构





具体代码事例如下：





![](http://sishuok.com/forum/upload/2012/7/1/254a61cd5c20f1e8f0e1012f02cdaa31__11.JPG "点击查看原始大小图片")





![](http://sishuok.com/forum/upload/2012/7/1/0ec4c098750790295eaad2f1b3a4dc82__12.JPG "点击查看原始大小图片")





![](http://sishuok.com/forum/upload/2012/7/1/1f330763be94e5abe7ecdd7928930dbb__13.JPG "点击查看原始大小图片")





从Model2架构可以看出，视图和模型分离了，控制逻辑和展示逻辑分离了。





但我们也看到严重的缺点：





1．  1、控制器：





1．1．1、控制逻辑可能比较复杂，其实我们可以按照规约，如请求参数submitFlag=toAdd，我们其实可以直接调用toAdd方法，来简化控制逻辑；而且每个模块基本需要一个控制器，造成控制逻辑可能很复杂；





1．1．2、请求参数到模型的封装比较麻烦，如果能交给框架来做这件事情，我们可以从中得到解放；





1．1．3、选择下一个视图，严重依赖Servlet API，这样很难或基本不可能更换视图；





1．1．4、给视图传输要展示的模型数据，使用Servlet API，更换视图技术也要一起更换，很麻烦。





1.2、模型：





1．2．1、此处模型使用JavaBean，可能造成JavaBean组件类很庞大，一般现在项目都是采用三层架构，而不采用JavaBean。





![](http://sishuok.com/forum/upload/2012/7/1/1570f54f6a52301d8ea58fe8fa9efb29__14.JPG)





1.3、视图





1．3．1、现在被绑定在JSP，很难更换视图，比如Velocity、FreeMarker；比如我要支持Excel、PDF视图等等。





**1.4.5、服务到工作者：Front Controller + Application Controller + Page Controller + Context**





即，前端控制器+应用控制器+页面控制器（也有称其为动作）+上下文，也是Web MVC，只是责任更加明确，详情请参考《核心J2EE设计模式》和《企业应用架构模式》如图1-10：





![](http://sishuok.com/forum/upload/2012/7/1/e7fae17e52bb3664d0d3f4ea8db7ae55__15.JPG "点击查看原始大小图片")





图1-10





运行流程如下：





![](http://sishuok.com/forum/upload/2012/7/1/925c5ff2cc8613c898b05a4817db1f56__16.JPG "点击查看原始大小图片")





职责：





**Front Controller：**前端控制器，负责为表现层提供统一访问点，从而避免Model2中出现的重复的控制逻辑（由前端控制器统一回调相应的功能方法，如前边的根据submitFlag=login转调login方法）；并且可以为多个请求提供共用的逻辑（如准备上下文等等），将选择具体视图和具体的功能处理（如login里边封装请求参数到模型，并调用业务逻辑对象）分离。





**Application Controller：**应用控制器，前端控制器分离选择具体视图和具体的功能处理之后，需要有人来管理，应用控制器就是用来选择具体视图技术（视图的管理）和具体的功能处理（页面控制器/命令对象/动作管理），一种策略设计模式的应用，可以很容易的切换视图/页面控制器，相互不产生影响。





**Page Controller(Command)：**页面控制器/动作/处理器：功能处理代码，收集参数、封装参数到模型，转调业务对象处理模型，返回逻辑视图名交给前端控制器（和具体的视图技术解耦），由前端控制器委托给应用控制器选择具体的视图来展示，可以是命令设计模式的实现。页面控制器也被称为处理器或动作。





**Context：**上下文，还记得Model2中为视图准备要展示的模型数据吗，我们直接放在request中（Servlet API相关），有了上下文之后，我们就可以将相关数据放置在上下文，从而与协议无关（如Servlet API）的访问/设置模型数据，一般通过ThreadLocal模式实现。





到此，我们回顾了整个web开发架构的发展历程，可能不同的web层框架在细节处理方面不同，但的目的是一样的：





干净的web表现层：





模型和视图的分离；





控制器中的控制逻辑与功能处理分离（收集并封装参数到模型对象、业务对象调用）；





控制器中的视图选择与具体视图技术分离。





轻薄的web表现层：





做的事情越少越好，薄薄的，不应该包含无关代码；





只负责收集并组织参数到模型对象，启动业务对象的调用；





控制器只返回逻辑视图名并由相应的应用控制器来选择具体使用的视图策略；





尽量少使用框架特定API，保证容易测试。





到此我们了解Web MVC的发展历程，接下来让我们了解下Spring MVC到底是什么、架构及来个HelloWorld了解下具体怎么使用吧。





本章具体代码请参考 springmvc-chapter1工程。





## Spring Web MVC是什么





Spring Web MVC是一种基于Java的实现了Web MVC设计模式的请求驱动类型的轻量级Web框架，即使用了MVC架构模式的思想，将web层进行职责解耦，基于请求驱动指的就是使用请求-响应模型，框架的目的就是帮助我们简化开发，Spring Web MVC也是要简化我们日常Web开发的。





另外还有一种基于组件的、事件驱动的Web框架在此就不介绍了，如Tapestry、JSF等。





Spring Web MVC也是服务到工作者模式的实现，但进行可优化。前端控制器是`DispatcherServlet；`应用控制器其实拆为处理器映射器(Handler Mapping)进行处理器管理和视图解析器(View Resolver)进行视图管理；页面控制器/动作/处理器为Controller接口（仅包含`ModelAndView handleRequest(request, response)` 方法）的实现（也可以是任何的POJO类）；支持本地化（Locale）解析、主题（Theme）解析及文件上传等；提供了非常灵活的数据验证、格式化和数据绑定机制；提供了强大的约定大于配置（惯例优先原则）的契约式编程支持。





## Spring Web MVC能帮我们做什么





√让我们能非常简单的设计出干净的Web层和薄薄的Web层；





√进行更简洁的Web层的开发；





√天生与Spring框架集成（如IoC容器、AOP等）；





√提供强大的约定大于配置的契约式编程支持；





√能简单的进行Web层的单元测试；





√支持灵活的URL到页面控制器的映射；





√非常容易与其他视图技术集成，如Velocity、FreeMarker等等，因为模型数据不放在特定的API里，而是放在一个Model里（`Map`数据结构实现，因此很容易被其他框架使用）；





√非常灵活的数据验证、格式化和数据绑定机制，能使用任何对象进行数据绑定，不必实现特定框架的API；





√提供一套强大的JSP标签库，简化JSP开发；





√支持灵活的本地化、主题等解析；





√更加简单的异常处理；





√对静态资源的支持；





√支持Restful风格。





## Spring Web MVC架构





Spring Web MVC框架也是一个基于请求驱动的Web框架，并且也使用了前端控制器模式来进行设计，再根据请求映射规则分发给相应的页面控制器（动作/处理器）进行处理。首先让我们整体看一下Spring Web MVC处理请求的流程：





### Spring Web MVC处理请求的流程





如图2-1





![](http://sishuok.com/forum/upload/2012/7/14/529024df9d2b0d1e62d8054a86d866c9__1.JPG "点击查看原始大小图片")





图2-1





具体执行步骤如下：





1、  首先用户发送请求————>前端控制器，前端控制器根据请求信息（如URL）来决定选择哪一个页面控制器进行处理并把请求委托给它，即以前的控制器的控制逻辑部分；图2-1中的1、2步骤；





2、  页面控制器接收到请求后，进行功能处理，首先需要收集和绑定请求参数到一个对象，这个对象在Spring Web MVC中叫命令对象，并进行验证，然后将命令对象委托给业务对象进行处理；处理完毕后返回一个ModelAndView（模型数据和逻辑视图名）；图2-1中的3、4、5步骤；





3、  前端控制器收回控制权，然后根据返回的逻辑视图名，选择相应的视图进行渲染，并把模型数据传入以便视图渲染；图2-1中的步骤6、7；





4、  前端控制器再次收回控制权，将响应返回给用户，图2-1中的步骤8；至此整个结束。





问题：





1、  请求如何给前端控制器？





2、  前端控制器如何根据请求信息选择页面控制器进行功能处理？





3、  如何支持多种页面控制器呢？





4、  如何页面控制器如何使用业务对象？





5、  页面控制器如何返回模型数据？





6、  前端控制器如何根据页面控制器返回的逻辑视图名选择具体的视图进行渲染？





7、  不同的视图技术如何使用相应的模型数据？





首先我们知道有如上问题，那这些问题如何解决呢？请让我们先继续，在后边依次回答。





### Spring Web MVC架构





1、Spring Web MVC核心架构图，如图2-2





![](http://sishuok.com/forum/upload/2012/7/14/57ea9e7edeebd5ee2ec0cf27313c5fb6__2.JPG "点击查看原始大小图片")





图2-2





架构图对应的DispatcherServlet核心代码如下：





java代码：

```
//前端控制器分派方法  
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {  
        HttpServletRequest processedRequest = request;  
        HandlerExecutionChain mappedHandler = null;  
        int interceptorIndex = -1;  

        try {  
            ModelAndView mv;  
            boolean errorView = false;  

            try {  
                   //检查是否是请求是否是multipart（如文件上传），如果是将通过MultipartResolver解析  
                processedRequest = checkMultipart(request);  
                   //步骤2、请求到处理器（页面控制器）的映射，通过HandlerMapping进行映射  
                mappedHandler = getHandler(processedRequest, false);  
                if (mappedHandler == null || mappedHandler.getHandler() == null) {  
                    noHandlerFound(processedRequest, response);  
                    return;  
                }  
                   //步骤3、处理器适配，即将我们的处理器包装成相应的适配器（从而支持多种类型的处理器）  
                HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());  

                  // 304 Not Modified缓存支持  
                //此处省略具体代码  

                // 执行处理器相关的拦截器的预处理（HandlerInterceptor.preHandle）  
                //此处省略具体代码  

                // 步骤4、由适配器执行处理器（调用处理器相应功能处理方法）  
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());  

                // Do we need view name translation?  
                if (mv != null && !mv.hasView()) {  
                    mv.setViewName(getDefaultViewName(request));  
                }  

                // 执行处理器相关的拦截器的后处理（HandlerInterceptor.postHandle）  
                //此处省略具体代码  
            }  
            catch (ModelAndViewDefiningException ex) {  
                logger.debug("ModelAndViewDefiningException encountered", ex);  
                mv = ex.getModelAndView();  
            }  
            catch (Exception ex) {  
                Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);  
                mv = processHandlerException(processedRequest, response, handler, ex);  
                errorView = (mv != null);  
            }  

            //步骤5 步骤6、解析视图并进行视图的渲染  
//步骤5 由ViewResolver解析View（viewResolver.resolveViewName(viewName, locale)）  
//步骤6 视图在渲染时会把Model传入（view.render(mv.getModelInternal(), request, response);）  
            if (mv != null && !mv.wasCleared()) {  
                render(mv, processedRequest, response);  
                if (errorView) {  
                    WebUtils.clearErrorRequestAttributes(request);  
                }  
            }  
            else {  
                if (logger.isDebugEnabled()) {  
                    logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +  
                            "': assuming HandlerAdapter completed request handling");  
                }  
            }  

            // 执行处理器相关的拦截器的完成后处理（HandlerInterceptor.afterCompletion）  
            //此处省略具体代码  

        catch (Exception ex) {  
            // Trigger after-completion for thrown exception.  
            triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, ex);  
            throw ex;  
        }  
        catch (Error err) {  
            ServletException ex = new NestedServletException("Handler processing failed", err);  
            // Trigger after-completion for thrown exception.  
            triggerAfterCompletion(mappedHandler, interceptorIndex, processedRequest, response, ex);  
            throw ex;  
        }  

        finally {  
            // Clean up any resources used by a multipart request.  
            if (processedRequest != request) {  
                cleanupMultipart(processedRequest);  
            }  
        }  
    }  

```





核心架构的具体流程步骤如下：





1、  首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；





2、  DispatcherServlet——>HandlerMapping， HandlerMapping将会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略；





3、  DispatcherServlet——>HandlerAdapter，HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；





4、  HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView对象（包含模型数据、逻辑视图名）；





5、  ModelAndView的逻辑视图名——> ViewResolver， ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；





6、  View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；





7、返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束。





此处我们只是讲了核心流程，没有考虑拦截器、本地解析、文件上传解析等，后边再细述。





到此，再来看我们前边提出的问题：





1、  请求如何给前端控制器？这个应该在web.xml中进行部署描述，在HelloWorld中详细讲解。





2、  前端控制器如何根据请求信息选择页面控制器进行功能处理？ 我们需要配置HandlerMapping进行映射





3、  如何支持多种页面控制器呢？配置HandlerAdapter从而支持多种类型的页面控制器





4、  如何页面控制器如何使用业务对象？可以预料到，肯定利用Spring IoC容器的依赖注入功能





5、  页面控制器如何返回模型数据？使用ModelAndView返回





6、  前端控制器如何根据页面控制器返回的逻辑视图名选择具体的视图进行渲染？ 使用ViewResolver进行解析





7、  不同的视图技术如何使用相应的模型数据？ 因为Model是一个Map数据结构，很容易支持其他视图技术





在此我们可以看出具体的核心开发步骤：





1、  DispatcherServlet在web.xml中的部署描述，从而拦截请求到Spring Web MVC





2、  HandlerMapping的配置，从而将请求映射到处理器





3、  HandlerAdapter的配置，从而支持多种类型的处理器





4、  ViewResolver的配置，从而将逻辑视图名解析为具体视图技术





5、处理器（页面控制器）的配置，从而进行功能处理





上边的开发步骤我们会在Hello World中详细验证。





## Spring Web MVC优势





1、清晰的角色划分：前端控制器（`DispatcherServlet`）、请求到处理器映射（HandlerMapping）、处理器适配器（HandlerAdapter）、视图解析器（ViewResolver）、处理器或页面控制器（Controller）、验证器（   Validator）、命令对象（Command  请求参数绑定到的对象就叫命令对象）、表单对象（Form Object 提供给表单展示和提交到的对象就叫表单对象）。





2、分工明确，而且扩展点相当灵活，可以很容易扩展，虽然几乎不需要；





3、由于命令对象就是一个POJO，无需继承框架特定API，可以使用命令对象直接作为业务对象；





4、和Spring 其他框架无缝集成，是其它Web框架所不具备的；





5、可适配，通过HandlerAdapter可以支持任意的类作为处理器；





6、可定制性，HandlerMapping、ViewResolver等能够非常简单的定制；





7、功能强大的数据验证、格式化、绑定机制；





8、利用Spring提供的Mock对象能够非常简单的进行Web层单元测试；





9、本地化、主题的解析的支持，使我们更容易进行国际化和主题的切换。





10、强大的JSP标签库，使JSP编写更容易。





………………还有比如RESTful风格的支持、简单的文件上传、约定大于配置的契约式编程支持、基于注解的零配置支持等等。





## DispatcherServlet作用





DispatcherServlet是前端控制器设计模式的实现，提供Spring Web MVC的集中访问点，而且负责职责的分派，而且与Spring IoC容器无缝集成，从而可以获得Spring的所有好处。 具体请参考第二章的图2-1。





DispatcherServlet主要用作职责调度工作，本身主要用于控制流程，主要职责如下：





1、文件上传解析，如果请求类型是multipart将通过MultipartResolver进行文件上传解析；





2、通过HandlerMapping，将请求映射到处理器（返回一个HandlerExecutionChain，它包括一个处理器、多个HandlerInterceptor拦截器）；





3、通过HandlerAdapter支持多种类型的处理器(HandlerExecutionChain中的处理器)；





4、通过ViewResolver解析逻辑视图名到具体视图实现；





5、本地化解析；





6、渲染具体的视图等；





7、如果执行过程中遇到异常将交给HandlerExceptionResolver来解析。





从以上我们可以看出DispatcherServlet主要负责流程的控制（而且在流程中的每个关键点都是很容易扩展的）。



作者：黄小斜
链接：http://www.imooc.com/article/291594?block_id=tuijian_wz
来源：慕课网
