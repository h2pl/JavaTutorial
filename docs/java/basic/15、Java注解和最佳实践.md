# 目录
  * [Java注解简介](#java注解简介)
    * [注解如同标签](#注解如同标签)
  * [Java 注解概述](#java-注解概述)
    * [什么是注解？](#什么是注解？)
    * [注解的用处](#注解的用处)
    * [注解的原理](#注解的原理)
    * [元注解](#元注解)
    * [JDK里的注解](#jdk里的注解)
  * [注解处理器实战](#注解处理器实战)
  * [不同类型的注解](#不同类型的注解)
    * [类注解](#类注解)
    * [方法注解](#方法注解)
    * [参数注解](#参数注解)
    * [变量注解](#变量注解)
  * [Java注解相关面试题](#java注解相关面试题)
    * [什么是注解？他们的典型用例是什么？](#什么是注解？他们的典型用例是什么？)
    * [描述标准库中一些有用的注解。](#描述标准库中一些有用的注解。)
    * [可以从注解方法声明返回哪些对象类型？](#可以从注解方法声明返回哪些对象类型？)
    * [哪些程序元素可以注解？](#哪些程序元素可以注解？)
    * [有没有办法限制可以应用注解的元素？](#有没有办法限制可以应用注解的元素？)
    * [什么是元注解？](#什么是元注解？)
    * [下面的代码会编译吗？](#下面的代码会编译吗？)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


---
title: 夯实Java基础系列15：Java注解简介和最佳实践
date: 2019-9-15 15:56:26 # 文章生成时间，一般不改
categories:
    - Java技术江湖
    - Java基础
tags:
    - annotation
    - Java注解
---

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《夯实Java基础系列博文》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。
该系列博文会告诉你如何从入门到进阶，一步步地学习Java基础知识，并上手进行实战，接着了解每个Java知识点背后的实现原理，更完整地了解整个Java技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。 

<!-- more -->

## Java注解简介

Annotation 中文译过来就是注解、标释的意思，在 Java 中注解是一个很重要的知识点，但经常还是有点让新手不容易理解。

**我个人认为，比较糟糕的技术文档主要特征之一就是：用专业名词来介绍专业名词。**
比如：

> Java 注解用于为 Java 代码提供元数据。作为元数据，注解不直接影响你的代码执行，但也有一些类型的注解实际上可以用于这一目的。Java 注解是从 Java5 开始添加到 Java 的。
> 这是大多数网站上对于 Java 注解，解释确实正确，但是说实在话，我第一次学习的时候，头脑一片空白。这什么跟什么啊？听了像没有听一样。因为概念太过于抽象，所以初学者实在是比较吃力才能够理解，然后随着自己开发过程中不断地强化练习，才会慢慢对它形成正确的认识。

我在写这篇文章的时候，我就在思考。如何让自己或者让读者能够比较直观地认识注解这个概念？是要去官方文档上翻译说明吗？我马上否定了这个答案。

后来，我想到了一样东西————墨水，墨水可以挥发、可以有不同的颜色，用来解释注解正好。

不过，我继续发散思维后，想到了一样东西能够更好地代替墨水，那就是印章。印章可以沾上不同的墨水或者印泥，可以定制印章的文字或者图案，如果愿意它也可以被戳到你任何想戳的物体表面。

但是，我再继续发散思维后，又想到一样东西能够更好地代替印章，那就是标签。标签是一张便利纸，标签上的内容可以自由定义。常见的如货架上的商品价格标签、图书馆中的书本编码标签、实验室中化学材料的名称类别标签等等。

并且，往抽象地说，标签并不一定是一张纸，它可以是对人和事物的属性评价。也就是说，标签具备对于抽象事物的解释。

![](https://img-blog.csdn.net/20170627213419176?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJpYmx1ZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



所以，基于如此，我完成了自我的知识认知升级，我决定用标签来解释注解。

### 注解如同标签

之前某新闻客户端的评论有盖楼的习惯，于是 “乔布斯重新定义了手机、罗永浩重新定义了傻X” 就经常极为工整地出现在了评论楼层中，并且广大网友在相当长的一段时间内对于这种行为乐此不疲。这其实就是等同于贴标签的行为。
在某些网友眼中，罗永浩就成了傻X的代名词。

广大网友给罗永浩贴了一个名为“傻x”的标签，他们并不真正了解罗永浩，不知道他当教师、砸冰箱、办博客的壮举，但是因为“傻x”这样的标签存在，这有助于他们直接快速地对罗永浩这个人做出评价，然后基于此，罗永浩就可以成为茶余饭后的谈资，这就是标签的力量。

而在网络的另一边，老罗靠他的人格魅力自然收获一大批忠实的拥泵，他们对于老罗贴的又是另一种标签。 

![](https://img-blog.csdn.net/20170627213530055?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYnJpYmx1ZQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


老罗还是老罗，但是由于人们对于它贴上的标签不同，所以造成对于他的看法大相径庭，不喜欢他的人整天在网络上评论抨击嘲讽，而崇拜欣赏他的人则会愿意挣钱购买锤子手机的发布会门票。

我无意于评价这两种行为，我再引个例子。

《奇葩说》是近年网络上非常火热的辩论节目，其中辩手陈铭被另外一个辩手马薇薇攻击说是————“站在宇宙中心呼唤爱”，然后贴上了一个大大的标签————“鸡汤男”，自此以后，观众再看到陈铭的时候，首先映入脑海中便是“鸡汤男”三个大字，其实本身而言陈铭非常优秀，为人师表、作风正派、谈吐举止得体，但是在网络中，因为娱乐至上的环境所致，人们更愿意以娱乐的心态来认知一切，于是“鸡汤男”就如陈铭自己所说成了一个撕不了的标签。

**我们可以抽象概括一下，标签是对事物行为的某些角度的评价与解释。**

到这里，终于可以引出本文的主角注解了。

**初学者可以这样理解注解：想像代码具有生命，注解就是对于代码中某些鲜活个体的贴上去的一张标签。简化来讲，注解如同一张标签。**

在未开始学习任何注解具体语法而言，你可以把注解看成一张标签。这有助于你快速地理解它的大致作用。如果初学者在学习过程有大脑放空的时候，请不要慌张，对自己说：

注解，标签。注解，标签。

## Java 注解概述
### 什么是注解？


>   对于很多初次接触的开发者来说应该都有这个疑问？Annontation是Java5开始引入的新特征，中文名称叫注解。它提供了一种安全的类似注释的机制，用来将任何的信息或元数据（metadata）与程序元素（类、方法、成员变量等）进行关联。为程序的元素（类、方法、成员变量）加上更直观更明了的说明，这些说明信息是与程序的业务逻辑无关，并且供指定的工具或框架使用。Annontation像一种修饰符一样，应用于包、类型、构造方法、方法、成员变量、参数及本地变量的声明语句中。


　　Java注解是附加在代码中的一些元信息，用于一些工具在编译、运行时进行解析和使用，起到说明、配置的功能。注解不会也不能影响代码的实际逻辑，仅仅起到辅助性的作用。包含在 java.lang.annotation 包中。

 

### 注解的用处

  1、生成文档。这是最常见的，也是java 最早提供的注解。常用的有@param @return 等
  2、跟踪代码依赖性，实现替代配置文件功能。比如Dagger 2依赖注入，未来java开发，将大量注解配置，具有很大用处;
  3、在编译时进行格式检查。如@override 放在方法前，如果你这个方法并不是覆盖了超类方法，则编译时就能检查出。

 

### 注解的原理
　　注解本质是一个继承了Annotation的特殊接口，其具体实现类是Java运行时生成的动态代理类。而我们通过反射获取注解时，返回的是Java运行时生成的动态代理对象$Proxy1。通过代理对象调用自定义注解（接口）的方法，会最终调用AnnotationInvocationHandler的invoke方法。该方法会从memberValues这个Map中索引出对应的值。而memberValues的来源是Java常量池。

 

### 元注解
java.lang.annotation提供了四种元注解，专门注解其他的注解（在自定义注解的时候，需要使用到元注解）：
   @Documented –注解是否将包含在JavaDoc中
   @Retention –什么时候使用该注解
   @Target –注解用于什么地方
   @Inherited – 是否允许子类继承该注解

  1.）@Retention– 定义该注解的生命周期

      ●   RetentionPolicy.SOURCE : 在编译阶段丢弃。这些注解在编译结束之后就不再有任何意义，所以它们不会写入字节码。@Override, @SuppressWarnings都属于这类注解。
      
      ●   RetentionPolicy.CLASS : 在类加载的时候丢弃。在字节码文件的处理中有用。注解默认使用这种方式
      
      ●   RetentionPolicy.RUNTIME : 始终不会丢弃，运行期也保留该注解，因此可以使用反射机制读取该注解的信息。我们自定义的注解通常使用这种方式。

  2.）Target – 表示该注解用于什么地方。默认值为任何元素，表示该注解用于什么地方。可用的ElementType参数包括

      ● ElementType.CONSTRUCTOR:用于描述构造器
      ● ElementType.FIELD:成员变量、对象、属性（包括enum实例）
      ● ElementType.LOCAL_VARIABLE:用于描述局部变量
      ● ElementType.METHOD:用于描述方法
      ● ElementType.PACKAGE:用于描述包
      ● ElementType.PARAMETER:用于描述参数
      ● ElementType.TYPE:用于描述类、接口(包括注解类型) 或enum声明

 3.)@Documented–一个简单的Annotations标记注解，表示是否将注解信息添加在java文档中。

 4.)@Inherited – 定义该注释和子类的关系
     @Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。


### JDK里的注解
JDK 内置注解
先来看几个 Java 内置的注解，让大家热热身。

   @Override 演示

    class Parent {
        public void run() {
        }
    }
    
    class Son extends Parent {
        /**
         * 这个注解是为了检查此方法是否真的是重写父类的方法
         * 这时候就不用我们用肉眼去观察到底是不是重写了
         */
        @Override
        public void run() {
        }
    }
@Deprecated 演示
class Parent {

    /**
     * 此注解代表过时了，但是如果可以调用到，当然也可以正常使用
     * 但是，此方法有可能在以后的版本升级中会被慢慢的淘汰
     * 可以放在类，变量，方法上面都起作用
     */
    @Deprecated
    public void run() {
    }
    }
    
    public class JDKAnnotationDemo {
        public static void main(String[] args) {
            Parent parent = new Parent();
            parent.run(); // 在编译器中此方法会显示过时标志
        }
    }
@SuppressWarnings 演示
class Parent {

    // 因为定义的 name 没有使用，那么编译器就会有警告，这时候使用此注解可以屏蔽掉警告
    // 即任意不想看到的编译时期的警告都可以用此注解屏蔽掉，但是不推荐，有警告的代码最好还是处理一下
    @SuppressWarnings("all")
    private String name;
    }

@FunctionalInterface 演示
/**
 * 此注解是 Java8 提出的函数式接口，接口中只允许有一个抽象方法
 * 加上这个注解之后，类中多一个抽象方法或者少一个抽象方法都会报错
 */
@FunctionalInterface
interface Func {
    void run();
}


## 注解处理器实战

注解处理器
注解处理器才是使用注解整个流程中最重要的一步了。所有在代码中出现的注解，它到底起了什么作用，都是在注解处理器中定义好的。
概念：注解本身并不会对程序的编译方式产生影响，而是注解处理器起的作用；注解处理器能够通过在运行时使用反射获取在程序代码中的使用的注解信息，从而实现一些额外功能。前提是我们自定义的注解使用的是 RetentionPolicy.RUNTIME 修饰的。这也是我们在开发中使用频率很高的一种方式。

我们先来了解下如何通过在运行时使用反射获取在程序中的使用的注解信息。如下类注解和方法注解。

类注解
    Class aClass = ApiController.class;
    Annotation[] annotations = aClass.getAnnotations();
    
    for(Annotation annotation : annotations) {
        if(annotation instanceof ApiAuthAnnotation) {
            ApiAuthAnnotation apiAuthAnnotation = (ApiAuthAnnotation) annotation;
            System.out.println("name: " + apiAuthAnnotation.name());
            System.out.println("age: " + apiAuthAnnotation.age());
        }
    }
    方法注解
    Method method = ... //通过反射获取方法对象
    Annotation[] annotations = method.getDeclaredAnnotations();
    
    for(Annotation annotation : annotations) {
        if(annotation instanceof ApiAuthAnnotation) {
            ApiAuthAnnotation apiAuthAnnotation = (ApiAuthAnnotation) annotation;
            System.out.println("name: " + apiAuthAnnotation.name());
            System.out.println("age: " + apiAuthAnnotation.age());
        }
    }   
此部分内容可参考: 通过反射获取注解信息

注解处理器实战
接下来我通过在公司中的一个实战改编来演示一下注解处理器的真实使用场景。
需求: 网站后台接口只能是年龄大于 18 岁的才能访问，否则不能访问
前置准备: 定义注解（这里使用上文的完整注解），使用注解（这里使用上文中使用注解的例子）
接下来要做的事情: 写一个切面，拦截浏览器访问带注解的接口，取出注解信息，判断年龄来确定是否可以继续访问。

在 dispatcher-servlet.xml 文件中定义 aop 切面

    <aop:config>
        <!--定义切点，切的是我们自定义的注解-->
        <aop:pointcut id="apiAuthAnnotation" expression="@annotation(cn.caijiajia.devops.aspect.ApiAuthAnnotation)"/>
        <!--定义切面，切点是 apiAuthAnnotation，切面类即注解处理器是 apiAuthAspect，主处理逻辑在方法名为 auth 的方法中-->
        <aop:aspect ref="apiAuthAspect">
            <aop:around method="auth" pointcut-ref="apiAuthAnnotation"/>
        </aop:aspect>
    </aop:config>
切面类处理逻辑即注解处理器代码如

    @Component("apiAuthAspect")
    public class ApiAuthAspect {
    
        public Object auth(ProceedingJoinPoint pjp) throws Throwable {
            Method method = ((MethodSignature) pjp.getSignature()).getMethod();
            ApiAuthAnnotation apiAuthAnnotation = method.getAnnotation(ApiAuthAnnotation.class);
            Integer age = apiAuthAnnotation.age();
            if (age > 18) {
                return pjp.proceed();
            } else {
                throw new RuntimeException("你未满18岁，禁止访问");
            }
        }
    }



## 不同类型的注解

### 类注解

你可以在运行期访问类，方法或者变量的注解信息，下是一个访问类注解的例子：

```
 Class aClass = TheClass.class;
Annotation[] annotations = aClass.getAnnotations();

for(Annotation annotation : annotations){
    if(annotation instanceof MyAnnotation){
        MyAnnotation myAnnotation = (MyAnnotation) annotation;
        System.out.println("name: " + myAnnotation.name());
        System.out.println("value: " + myAnnotation.value());
    }
}
```

你还可以像下面这样指定访问一个类的注解：

```
Class aClass = TheClass.class;
Annotation annotation = aClass.getAnnotation(MyAnnotation.class);

if(annotation instanceof MyAnnotation){
    MyAnnotation myAnnotation = (MyAnnotation) annotation;
    System.out.println("name: " + myAnnotation.name());
    System.out.println("value: " + myAnnotation.value());
}
```

### 方法注解

下面是一个方法注解的例子：

```
public class TheClass {
  @MyAnnotation(name="someName",  value = "Hello World")
  public void doSomething(){}
}
```

你可以像这样访问方法注解：

```
Method method = ... //获取方法对象
Annotation[] annotations = method.getDeclaredAnnotations();

for(Annotation annotation : annotations){
    if(annotation instanceof MyAnnotation){
        MyAnnotation myAnnotation = (MyAnnotation) annotation;
        System.out.println("name: " + myAnnotation.name());
        System.out.println("value: " + myAnnotation.value());
    }
}
```

你可以像这样访问指定的方法注解：

```
Method method = ... // 获取方法对象
Annotation annotation = method.getAnnotation(MyAnnotation.class);

if(annotation instanceof MyAnnotation){
    MyAnnotation myAnnotation = (MyAnnotation) annotation;
    System.out.println("name: " + myAnnotation.name());
    System.out.println("value: " + myAnnotation.value());
}
```

### 参数注解

方法参数也可以添加注解，就像下面这样：

```
public class TheClass {
  public static void doSomethingElse(
        @MyAnnotation(name="aName", value="aValue") String parameter){
  }
}
```

你可以通过 Method对象来访问方法参数注解：

```
Method method = ... //获取方法对象
Annotation[][] parameterAnnotations = method.getParameterAnnotations();
Class[] parameterTypes = method.getParameterTypes();

int i=0;
for(Annotation[] annotations : parameterAnnotations){
  Class parameterType = parameterTypes[i++];

  for(Annotation annotation : annotations){
    if(annotation instanceof MyAnnotation){
        MyAnnotation myAnnotation = (MyAnnotation) annotation;
        System.out.println("param: " + parameterType.getName());
        System.out.println("name : " + myAnnotation.name());
        System.out.println("value: " + myAnnotation.value());
    }
  }
}
```

需要注意的是 Method.getParameterAnnotations()方法返回一个注解类型的二维数组，每一个方法的参数包含一个注解数组。

### 变量注解

下面是一个变量注解的例子：

```
public class TheClass {

  @MyAnnotation(name="someName",  value = "Hello World")
  public String myField = null;
}
```

你可以像这样来访问变量的注解：

```
Field field = ... //获取方法对象</pre>
<pre>Annotation[] annotations = field.getDeclaredAnnotations();

for(Annotation annotation : annotations){
 if(annotation instanceof MyAnnotation){
 MyAnnotation myAnnotation = (MyAnnotation) annotation;
 System.out.println("name: " + myAnnotation.name());
 System.out.println("value: " + myAnnotation.value());
 }
}
```

你可以像这样访问指定的变量注解：

```
Field field = ...//获取方法对象</pre>
<pre>
Annotation annotation = field.getAnnotation(MyAnnotation.class);

if(annotation instanceof MyAnnotation){
 MyAnnotation myAnnotation = (MyAnnotation) annotation;
 System.out.println("name: " + myAnnotation.name());
 System.out.println("value: " + myAnnotation.value());
}
```

## Java注解相关面试题

### 什么是注解？他们的典型用例是什么？

注解是绑定到程序源代码元素的元数据，对运行代码的操作没有影响。

他们的典型用例是：

*   编译器的信息 - 使用注解，编译器可以检测错误或抑制警告
*   编译时和部署时处理 - 软件工具可以处理注解并生成代码，配置文件等。
*   运行时处理 - 可以在运行时检查注解以自定义程序的行为

### 描述标准库中一些有用的注解。

java.lang和java.lang.annotation包中有几个注解，更常见的包括但不限于此：

*   @Override -标记方法是否覆盖超类中声明的元素。如果它无法正确覆盖该方法，编译器将发出错误
*   @Deprecated - 表示该元素已弃用且不应使用。如果程序使用标有此批注的方法，类或字段，编译器将发出警告
*   @SuppressWarnings - 告诉编译器禁止特定警告。在与泛型出现之前编写的遗留代码接口时最常用的
*   @FunctionalInterface - 在Java 8中引入，表明类型声明是一个功能接口，可以使用Lambda Expression提供其实现


### 可以从注解方法声明返回哪些对象类型？

返回类型必须是基本类型，String，Class，Enum或数组类型之一。否则，编译器将抛出错误。

这是一个成功遵循此原则的示例代码：

```
enum Complexity {
    LOW, HIGH
}

public @interface ComplexAnnotation {
    Class<? extends Object> value();

    int[] types();

    Complexity complexity();
}

```

下一个示例将无法编译，因为Object不是有效的返回类型：

```
public @interface FailingAnnotation {
    Object complexity();
}

```

### 哪些程序元素可以注解？

注解可以应用于整个源代码的多个位置。它们可以应用于类，构造函数和字段的声明：

```
@SimpleAnnotation
public class Apply {
    @SimpleAnnotation
    private String aField;

    @SimpleAnnotation
    public Apply() {
        // ...
    }
}

```

方法及其参数：

```
@SimpleAnnotation
public void aMethod(@SimpleAnnotation String param) {
    // ...
}

```

局部变量，包括循环和资源变量：

```
@SimpleAnnotation
int i = 10;

for (@SimpleAnnotation int j = 0; j < i; j++) {
    // ...
}

try (@SimpleAnnotation FileWriter writer = getWriter()) {
    // ...
} catch (Exception ex) {
    // ...
}

```

其他注解类型：

```
@SimpleAnnotation
public @interface ComplexAnnotation {
    // ...
}

```

甚至包，通过package-info.java文件：

```
@PackageAnnotation
package com.baeldung.interview.annotations;

```

从Java 8开始，它们也可以应用于类型的使用。为此，注解必须指定值为ElementType.USE的@Target注解：

```
@Target(ElementType.TYPE_USE)
public @interface SimpleAnnotation {
    // ...
}

```

现在，注解可以应用于类实例创建：

```
new @SimpleAnnotation Apply();

```

类型转换：

```
aString = (@SimpleAnnotation String) something;

```

接口中：

```
public class SimpleList<T>
  implements @SimpleAnnotation List<@SimpleAnnotation T> {
    // ...
}

```

抛出异常上：

```
void aMethod() throws @SimpleAnnotation Exception {
    // ...
}

```

### 有没有办法限制可以应用注解的元素？

有，@ Target注解可用于此目的。如果我们尝试在不适用的上下文中使用注解，编译器将发出错误。

以下是仅将@SimpleAnnotation批注的用法限制为字段声明的示例：

```
@Target(ElementType.FIELD)
public @interface SimpleAnnotation {
    // ...
}

```

如果我们想让它适用于更多的上下文，我们可以传递多个常量：

```
@Target({ ElementType.FIELD, ElementType.METHOD, ElementType.PACKAGE })

```

我们甚至可以制作一个注解，因此它不能用于注解任何东西。当声明的类型仅用作复杂注解中的成员类型时，这可能会派上用场：

```
@Target({})
public @interface NoTargetAnnotation {
    // ...
}

```

### 什么是元注解？

元注解适用于其他注解的注解。

所有未使用@Target标记或使用它标记但包含ANNOTATION_TYPE常量的注解也是元注解：

```
@Target(ElementType.ANNOTATION_TYPE)
public @interface SimpleAnnotation {
    // ...
}

```


### 下面的代码会编译吗？

```
@Target({ ElementType.FIELD, ElementType.TYPE, ElementType.FIELD })
public @interface TestAnnotation {
    int[] value() default {};
}

```

不能。如果在@Target注解中多次出现相同的枚举常量，那么这是一个编译时错误。

删除重复常量将使代码成功编译：

```
@Target({ ElementType.FIELD, ElementType.TYPE})

```

## 参考文章

https://blog.fundodoo.com/2018/04/19/130.html
https://blog.csdn.net/qq_37939251/article/details/83215703
https://blog.51cto.com/4247649/2109129
https://www.jianshu.com/p/2f2460e6f8e7
https://blog.csdn.net/yuzongtao/article/details/83306182

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。 

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
