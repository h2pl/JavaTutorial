# 目录
  * [聊聊IDE的实现原理](#聊聊ide的实现原理)
    * [源代码保存](#源代码保存)
    * [编译为class文件](#编译为class文件)
    * [查找class](#查找class)
    * [生成对象，并调用对象方法](#生成对象，并调用对象方法)
  * [javac命令初窥](#javac命令初窥)
    * [classpath是什么](#classpath是什么)
      * [IDE中的classpath](#ide中的classpath)
      * [Java项目和Java web项目的本质区别](#java项目和java-web项目的本质区别)
    * [javac命令后缀](#javac命令后缀)
      * [-g、-g:none、-g:{lines,vars,source}](#-g、-gnone、-g{linesvarssource})
      * [-bootclasspath、-extdirs](#-bootclasspath、-extdirs)
      * [-sourcepath和-classpath（-cp）](#-sourcepath和-classpath（-cp）)
      * [-d](#-d)
      * [-implicit:{none,class}](#-implicit{noneclass})
      * [-source和-target](#-source和-target)
      * [-encoding](#-encoding)
      * [-verbose](#-verbose)
      * [其他命令](#其他命令)
  * [使用javac构建项目](#使用javac构建项目)
                    * [](#)
* [java文件列表目录](#java文件列表目录)
                        * [放入列表文件中](#放入列表文件中)
                * [生成bin目录](#生成bin目录)
                * [列表](#列表)
    * [通过-cp指定所有的引用jar包，将src下的所有java文件进行编译](#通过-cp指定所有的引用jar包，将src下的所有java文件进行编译)
    * [通过-cp指定所有的引用jar包，指定入口函数运行](#通过-cp指定所有的引用jar包，指定入口函数运行)
  * [javap 的使用](#javap-的使用)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)
---
title: 夯实Java基础系列20：从IDE的实现原理聊起，谈谈那些年我们用过的Java命令
date: 2019-9-20 15:56:26 # 文章生成时间，一般不改
categories:
    - Java技术江湖
    - Java基础
tags:
    - Java命令行
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

## 聊聊IDE的实现原理

> IDE是把双刃剑，它可以什么都帮你做了，你只要敲几行代码，点几下鼠标，程序就跑起来了，用起来相当方便。
>
> 你不用去关心它后面做了些什么，执行了哪些命令，基于什么原理。然而也是这种过分的依赖往往让人散失了最基本的技能，当到了一个没有IDE的地方，你便觉得无从下手，给你个代码都不知道怎么去跑。好比给你瓶水，你不知道怎么打开去喝，然后活活给渴死。
>
> 之前用惯了idea，Java文件编译运行的命令基本忘得一干二净。

那好，不如咱们先来了解一下IDE的实现原理，这样一来，即使离开IDE，我们还是知道如何运行Java程序了。

像Eclipse等java IDE是怎么编译和查找java源代码的呢？

### 源代码保存
这个无需多说，在编译器写入代码，并保存到文件。这个利用流来实现。

### 编译为class文件
java提供了JavaCompiler，我们可以通过它来编译java源文件为class文件。

### 查找class
可以通过Class.forName(fullClassPath)或自定义类加载器来实现。

### 生成对象，并调用对象方法
通过上面一个查找class，得到Class对象后，可以通过newInstance()或构造器的newInstance()得到对象。然后得到Method，最后调用方法，传入相关参数即可。

示例代码：

    public class MyIDE {
    
        public static void main(String[] args) throws IOException, ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
            // 定义java代码，并保存到文件（Test.java）
            StringBuilder sb = new StringBuilder();
            sb.append("package com.tommy.core.test.reflect;\n");
            sb.append("public class Test {\n");
            sb.append("    private String name;\n");
            sb.append("    public Test(String name){\n");
            sb.append("        this.name = name;\n");
            sb.append("        System.out.println(\"hello,my name is \" + name);\n");
            sb.append("    }\n");
            sb.append("    public String sayHello(String name) {\n");
            sb.append("        return \"hello,\" + name;\n");
            sb.append("    }\n");
            sb.append("}\n");
    
            System.out.println(sb.toString());
    
            String baseOutputDir = "F:\\output\\classes\\";
            String baseDir = baseOutputDir + "com\\tommy\\core\\test\\reflect\\";
            String targetJavaOutputPath = baseDir + "Test.java";
            // 保存为java文件
            FileWriter fileWriter = new FileWriter(targetJavaOutputPath);
            fileWriter.write(sb.toString());
            fileWriter.flush();
            fileWriter.close();
    
            // 编译为class文件
            JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
            StandardJavaFileManager manager = compiler.getStandardFileManager(null,null,null);
            List<File> files = new ArrayList<>();
            files.add(new File(targetJavaOutputPath));
            Iterable compilationUnits = manager.getJavaFileObjectsFromFiles(files);
    
            // 编译
            // 设置编译选项，配置class文件输出路径
            Iterable<String> options = Arrays.asList("-d",baseOutputDir);
            JavaCompiler.CompilationTask task = compiler.getTask(null, manager, null, options, null, compilationUnits);
            // 执行编译任务
            task.call();


​    
​            // 通过反射得到对象
​    //        Class clazz = Class.forName("com.tommy.core.test.reflect.Test");
​            // 使用自定义的类加载器加载class
​            Class clazz = new MyClassLoader(baseOutputDir).loadClass("com.tommy.core.test.reflect.Test");
​            // 得到构造器
​            Constructor constructor = clazz.getConstructor(String.class);
​            // 通过构造器new一个对象
​            Object test = constructor.newInstance("jack.tsing");
​            // 得到sayHello方法
​            Method method = clazz.getMethod("sayHello", String.class);
​            // 调用sayHello方法
​            String result = (String) method.invoke(test, "jack.ma");
​            System.out.println(result);
​        }
​    }

自定义类加载器代码：


​    
​    public class MyClassLoader extends ClassLoader {
​        private String baseDir;
​        public MyClassLoader(String baseDir) {
​            this.baseDir = baseDir;
​        }
​        @Override
​        protected Class<?> findClass(String name) throws ClassNotFoundException {
​            String fullClassFilePath = this.baseDir + name.replace("\\.","/") + ".class";
​            File classFilePath = new File(fullClassFilePath);
​            if (classFilePath.exists()) {
​                FileInputStream fileInputStream = null;
​                ByteArrayOutputStream byteArrayOutputStream = null;
​                try {
​                    fileInputStream = new FileInputStream(classFilePath);
​                    byte[] data = new byte[1024];
​                    int len = -1;
​                    byteArrayOutputStream = new ByteArrayOutputStream();
​                    while ((len = fileInputStream.read(data)) != -1) {
​                        byteArrayOutputStream.write(data,0,len);
​                    }
​    
                    return defineClass(name,byteArrayOutputStream.toByteArray(),0,byteArrayOutputStream.size());
                } catch (FileNotFoundException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    if (null != fileInputStream) {
                        try {
                            fileInputStream.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
    
                    if (null != byteArrayOutputStream) {
                        try {
                            byteArrayOutputStream.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
            return super.findClass(name);
        }
    }    

## javac命令初窥

注：以下红色标记的参数在下文中有所讲解。

本部分参考https://www.cnblogs.com/xiazdong/p/3216220.html

用法: javac <options> <source files>

其中, 可能的选项包括:

>   -g                         生成所有调试信息
>
>   -g:none                    不生成任何调试信息
>
>   -g:{lines,vars,source}     只生成某些调试信息
>
>   -nowarn                    不生成任何警告
>
>   -verbose                   输出有关编译器正在执行的操作的消息
>
>   -deprecation               输出使用已过时的 API 的源位置
>
>   -classpath <路径>            指定查找用户类文件和注释处理程序的位置
>
>   -cp <路径>                   指定查找用户类文件和注释处理程序的位置
>
>   -sourcepath <路径>           指定查找输入源文件的位置
>
>   -bootclasspath <路径>        覆盖引导类文件的位置
>
>   -extdirs <目录>              覆盖所安装扩展的位置
>
>   -endorseddirs <目录>         覆盖签名的标准路径的位置
>
>   -proc:{none,only}          控制是否执行注释处理和/或编译。
>
>   -processor <class1>[,<class2>,<class3>...] 要运行的注释处理程序的名称; 绕过默认的搜索进程
>
>   -processorpath <路径>        指定查找注释处理程序的位置
>
>   -d <目录>                    指定放置生成的类文件的位置
>
>   -s <目录>                    指定放置生成的源文件的位置
>
>   -implicit:{none,class}     指定是否为隐式引用文件生成类文件
>
>   -encoding <编码>             指定源文件使用的字符编码
>
>   -source <发行版>              提供与指定发行版的源兼容性
>
>   -target <发行版>              生成特定 VM 版本的类文件
>
>   -version                   版本信息
>
>   -help                      输出标准选项的提要
>
>   -A关键字[=值]                  传递给注释处理程序的选项
>
>   -X                         输出非标准选项的提要
>
>   -J<标记>                     直接将 <标记> 传递给运行时系统
>
>   -Werror                    出现警告时终止编译
>
>   @<文件名>                     从文件读取选项和文件名


在详细介绍javac命令之前，先看看这个classpath是什么


### classpath是什么

在dos下编译java程序，就要用到classpath这个概念，尤其是在没有设置环境变量的时候。classpath就是存放.class等编译后文件的路径。

javac：如果当前你要编译的java文件中引用了其它的类(比如说：继承)，但该引用类的.class文件不在当前目录下，这种情况下就需要在javac命令后面加上-classpath参数，通过使用以下三种类型的方法 来指导编译器在编译的时候去指定的路径下查找引用类。

> (1).绝对路径：javac -classpath c:/junit3.8.1/junit.jar   Xxx.java
>
> (2).相对路径：javac -classpath ../junit3.8.1/Junit.javr  Xxx.java
>
> (3).系统变量：javac -classpath %CLASSPATH% Xxx.java (注意：%CLASSPATH%表示使用系统变量CLASSPATH的值进行查找，这里假设Junit.jar的路径就包含在CLASSPATH系统变量中)


#### IDE中的classpath

对于一个普通的Javaweb项目，一般有这样的配置：

> 1 WEB-INF/classes,lib才是classpath，WEB-INF/ 是资源目录, 客户端不能直接访问。
>
> 2、WEB-INF/classes目录存放src目录java文件编译之后的class文件，xml、properties等资源配置文件，这是一个定位资源的入口。
>
> 3、引用classpath路径下的文件，只需在文件名前加classpath:
>
> <param-value>classpath:applicationContext-*.xml</param-value> 
> <!-- 引用其子目录下的文件,如 -->
> <param-value>classpath:context/conf/controller.xml</param-value>
>
> 4、lib和classes同属classpath，两者的访问优先级为: lib>classes。
>
> 5、classpath 和 classpath* 区别：
>
> classpath：只会到你的class路径中查找找文件;
> classpath*：不仅包含class路径，还包括jar文件中(class路径)进行查找。

总结：

(1).何时需要使用-classpath：当你要编译或执行的类引用了其它的类，但被引用类的.class文件不在当前目录下时，就需要通过-classpath来引入类

(2).何时需要指定路径：当你要编译的类所在的目录和你执行javac命令的目录不是同一个目录时，就需要指定源文件的路径(CLASSPATH是用来指定.class路径的，不是用来指定.java文件的路径的) 
#### Java项目和Java web项目的本质区别

（看清IDE及classpath本质）

> 现在只是说说Java Project和Web Project，那么二者有区别么？回答：没有！都是Java语言的应用，只是应用场合不同罢了，那么他们的本质到底是什么？

> 回答：编译后路径！虚拟机执行的是class文件而不是java文件，那么我们不管是何种项目都是写的java文件，怎么就不一样了呢？分成java和web两种了呢？

> 从.classpath文件入手来看，这个文件在每个项目目录下都是存在的，很少有人打开看吧，那么我们就来一起看吧。这是一个XML文件，使用文本编辑器打开即可。
>
> 这里展示一个web项目的.classpath

Xml代码

    <?xml version="1.0" encoding="UTF-8"?>
    <classpath>
    <classpathentry kind="src" path="src"/>
    <classpathentry kind="src" path="resources"/>
    <classpathentry kind="src" path="test"/>
    <classpathentry kind="con" path="org.eclipse.jdt.launching.JRE_CONTAINER"/>
    <classpathentry kind="lib" path="lib/servlet-api.jar"/>
    <classpathentry kind="lib" path="webapp/WEB-INF/lib/struts2-core-2.1.8.1.jar"/>
         ……
    <classpathentry kind="output" path="webapp/WEB-INF/classes"/>
    </classpath>

> XML文档包含一个根元素，就是classpath，类路径，那么这里面包含了什么信息呢？子元素是classpathentry，kind属性区别了种 类信息，src源码，con你看看后面的path就知道是JRE容器的信息。lib是项目依赖的第三方类库，output是src编译后的位置。

> 既然是web项目，那么就是WEB-INF/classes目录，可能用MyEclipse的同学会说他们那里是WebRoot或者是WebContext而不是webapp，有区别么？回答：完全没有！

> 既然看到了编译路径的本来面目后，还区分什么java项目和web项目么？回答:不区分！普通的java 项目你这样写就行了：<classpathentry kind="output" path="bin"/>，看看Eclipse是不是这样生成的？这个问题解决了吧。

> 再说说webapp目录命名的问题，这个无所谓啊，web项目是要发布到服务器上的对吧，那么服务器读取的是类文件和页面文件吧，它不管源文件，它也无法去理解源文件。那么webapp目录的命名有何关系呢？只要让服务器找到不就行了。

### javac命令后缀

#### -g、-g:none、-g:{lines,vars,source}

> •-g：在生成的class文件中包含所有调试信息（行号、变量、源文件）
> •-g:none ：在生成的class文件中不包含任何调试信息。
>
> 这个参数在javac编译中是看不到什么作用的，因为调试信息都在class文件中，而我们看不懂这个class文件。
>
> 为了看出这个参数的作用，我们在eclipse中进行实验。在eclipse中，我们经常做的事就是“debug”，而在debug的时候，我们会
> •加入“断点”，这个是靠-g:lines起作用，如果不记录行号，则不能加断点。
> •在“variables”窗口中查看当前的变量，如下图所示，这是靠-g:vars起作用，否则不能查看变量信息。
> •在多个文件之间来回调用，比如 A.java的main()方法中调用了B.java的fun()函数，而我想看看程序进入fun()后的状态，这是靠-g:source，如果没有这个参数，则不能查看B.java的源代码。

#### -bootclasspath、-extdirs

> -bootclasspath和-extdirs 几乎不需要用的，因为他是用来改变 “引导类”和“扩展类”。
> •引导类(组成Java平台的类)：Java\jdk1.7.0_25\jre\lib\rt.jar等，用-bootclasspath设置。
> •扩展类：Java\jdk1.7.0_25\jre\lib\ext目录中的文件，用-extdirs设置。
> •用户自定义类：用-classpath设置。
>
> 我们用-verbose编译后出现的“类文件的搜索路径”，就是由上面三个路径组成，如下：


    [类文件的搜索路径: C:\Java\jdk1.7.0_25\jre\lib\resources.jar,C:\Java\jdk1.7.0_25
    
    \jre\lib\rt.jar,C:\Java\jdk1.7.0_25\jre\lib\sunrsasign.jar,C:\Java\jdk1.7.0_25\j
    
    re\lib\jsse.jar,C:\Java\jdk1.7.0_25\jre\lib\jce.jar,C:\Java\jdk1.7.0_25\jre\lib\
    
    charsets.jar,C:\Java\jdk1.7.0_25\jre\lib\jfr.jar,C:\Java\jdk1.7.0_25\jre\classes
    
    ,C:\Java\jdk1.7.0_25\jre\lib\ext\access-bridge-32.jar,C:\Java\jdk1.7.0_25\jre\li
    
    b\ext\dnsns.jar,C:\Java\jdk1.7.0_25\jre\lib\ext\jaccess.jar,C:\Java\jdk1.7.0_25\
    
    jre\lib\ext\localedata.jar,C:\Java\jdk1.7.0_25\jre\lib\ext\sunec.jar,C:\Java\jdk
    
    1.7.0_25\jre\lib\ext\sunjce_provider.jar,C:\Java\jdk1.7.0_25\jre\lib\ext\sunmsca
    
    pi.jar,C:\Java\jdk1.7.0_25\jre\lib\ext\sunpkcs11.jar,C:\Java\jdk1.7.0_25\jre\lib
    \ext\zipfs.jar,..\bin]             

如果利用 -bootclasspath 重新定义： javac -bootclasspath src Xxx.java，则会出现下面错误：


致命错误: 在类路径或引导类路径中找不到程序包 java.lang

#### -sourcepath和-classpath（-cp）

•-classpath(-cp)指定你依赖的类的class文件的查找位置。在Linux中，用“:”分隔classpath，而在windows中，用“;”分隔。
•-sourcepath指定你依赖的类的java文件的查找位置。

举个例子，



    public class A
    {
        public static void main(String[] args) {
            B b = new B();
            b.print();
        }
    }


​    
​    
​    public class B
​    {
​        public void print()
​        {
​            System.out.println("old");
​        }
​    }


目录结构如下：


sourcepath          //此处为当前目录

```
|-src
　　　　|-com
　　　　　　|- B.java
　　　　|- A.java
　　|-bin
　　　　|- B.class               //是 B.java
```
 编译后的类文件

如果要编译 A.java，则必须要让编译器找到类B的位置，你可以指定B.class的位置，也可以是B.java的位置，也可以同时都存在。


    javac -classpath bin src/A.java                            //查找到B.class
    
    javac -sourcepath src/com src/A.java                   //查找到B.java
    
    javac -sourcepath src/com -classpath bin src/A.java    //同时查找到B.class和B.java

如果同时找到了B.class和B.java，则：
•如果B.class和B.java内容一致，则遵循B.class。
•如果B.class和B.java内容不一致，则遵循B.java，并编译B.java。

以上规则可以通过 -verbose选项看出。

#### -d

•d就是 destination，用于指定.class文件的生成目录，在eclipse中，源文件都在src中，编译的class文件都是在bin目录中。

这里我用来实现一下这个功能，假设项目名称为project，此目录为当前目录，且在src/com目录中有一个Main.java文件。‘


​    
​    package com;
​    public class Main
​    {
​        public static void main(String[] args) {
​            System.out.println("Hello");
​        }
​    }


​    
​    
​    javac -d bin src/com/Main.java

上面的语句将Main.class生成在bin/com目录下。

#### -implicit:{none,class}

•如果有文件为A.java（其中有类A），且在类A中使用了类B，类B在B.java中，则编译A.java时，默认会自动编译B.java，且生成B.class。
•implicit:none：不自动生成隐式引用的类文件。
•implicit:class（默认）：自动生成隐式引用的类文件。

    public class A
    {
        public static void main(String[] args) {
            B b = new B();
        }
    }
    
    public class B
    {
    }
    
    如果使用：


​    
​     javac -implicit:none A.java

则不会生成 B.class。

#### -source和-target

•-source：使用指定版本的JDK编译，比如：-source 1.4表示用JDK1.4的标准编译，如果在源文件中使用了泛型，则用JDK1.4是不能编译通过的。
•-target：指定生成的class文件要运行在哪个JVM版本，以后实际运行的JVM版本必须要高于这个指定的版本。


javac -source 1.4 Xxx.java

javac -target 1.4 Xxx.java

#### -encoding

默认会使用系统环境的编码，比如我们一般用的中文windows就是GBK编码，所以直接javac时会用GBK编码，而Java文件一般要使用utf-8，如果用GBK就会出现乱码。 

•指定源文件的编码格式，如果源文件是UTF-8编码的，而-encoding GBK，则源文件就变成了乱码（特别是有中文时）。


javac -encoding UTF-8 Xxx.java

#### -verbose

输出详细的编译信息，包括：classpath、加载的类文件信息。

比如，我写了一个最简单的HelloWorld程序，在命令行中输入：


D:\Java>javac -verbose -encoding UTF-8 HelloWorld01.java

输出：


    [语法分析开始时间 RegularFileObject[HelloWorld01.java]]
    [语法分析已完成, 用时 21 毫秒]
    [源文件的搜索路径: .,D:\大三下\编译原理\cup\java-cup-11a.jar,E:\java\jflex\lib\J           //-sourcepath
    Flex.jar]
    [类文件的搜索路径: C:\Java\jdk1.7.0_25\jre\lib\resources.jar,C:\Java\jdk1.7.0_25      //-classpath、-bootclasspath、-extdirs
    省略............................................
    [正在加载ZipFileIndexFileObject[C:\Java\jdk1.7.0_25\lib\ct.sym(META-INF/sym/rt.j
    ar/java/lang/Object.class)]]
    [正在加载ZipFileIndexFileObject[C:\Java\jdk1.7.0_25\lib\ct.sym(META-INF/sym/rt.j
    ar/java/lang/String.class)]]
    [正在检查Demo]
    省略............................................
    [已写入RegularFileObject[Demo.class]]
    [共 447 毫秒]

编写一个程序时，比如写了一句：System.out.println("hello")，实际上还需要加载：Object、PrintStream、String等类文件，而上面就显示了加载的全部类文件。

#### 其他命令

-J <标记>
•传递一些信息给 Java Launcher.


    javac -J-Xms48m   Xxx.java          //set the startup memory to 48M.

-@<文件名>

> 如果同时需要编译数量较多的源文件(比如1000个)，一个一个编译是不现实的（当然你可以直接 javac *.java ），比较好的方法是：将你想要编译的源文件名都写在一个文件中（比如sourcefiles.txt），其中每行写一个文件名，如下所示：
>
>
> HelloWorld01.java
> HelloWorld02.java
> HelloWorld03.java

则使用下面的命令：


javac @sourcefiles.txt

编译这三个源文件。



## 使用javac构建项目

这部分参考：
https://blog.csdn.net/mingover/article/details/57083176

一个简单的javac编译

新建两个文件夹,src和 build 
src/com/yp/test/HelloWorld.java 
build/


```
├─build
└─src
    └─com
        └─yp
            └─test
                    HelloWorld.java
```


java文件非常简单

    package com.yp.test;
    public class HelloWorld {
    
        public static void main(String[] args) {
            System.out.println("helloWorld");
        }
    }
编译:
javac src/com/yp/test/HelloWorld.java -d build

-d 表示编译到 build文件夹下

```
查看build文件夹
├─build
│  └─com
│      └─yp
│          └─test
│                  HelloWorld.class
│
└─src
    └─com
        └─yp
            └─test
                    HelloWorld.java
```


运行文件
> E:\codeplace\n_learn\java\javacmd> java com/yp/test/HelloWorld.class
> 错误: 找不到或无法加载主类 build.com.yp.test.HelloWorld.class
>
> 运行时要指定main
> E:\codeplace\n_learn\java\javacmd\build> java com.yp.test.HelloWorld
> helloWorld

如果引用到多个其他的类，应该怎么做呢 ？

> 编译
>
> E:\codeplace\n_learn\java\javacmd>javac src/com/yp/test/HelloWorld.java -sourcepath src -d build -g
> 1
> -sourcepath 表示 从指定的源文件目录中找到需要的.java文件并进行编译。
> 也可以用-cp指定编译好的class的路径
> 运行,注意:运行在build目录下
>
> E:\codeplace\n_learn\java\javacmd\build>java com.yp.test.HelloWorld

怎么打成jar包?

> 生成:
> E:\codeplace\n_learn\java\javacmd\build>jar cvf h.jar *
> 运行:
> E:\codeplace\n_learn\java\javacmd\build>java h.jar
> 错误: 找不到或无法加载主类 h.jar

> 这个错误是没有指定main类，所以类似这样来指定:
> E:\codeplace\n_learn\java\javacmd\build>java -cp h.jar com.yp.test.HelloWorld


生成可以运行的jar包

需要指定jar包的应用程序入口点，用-e选项：

    E:\codeplace\n_learn\java\javacmd\build> jar cvfe h.jar com.yp.test.HelloWorld *
    已添加清单
    正在添加: com/(输入 = 0) (输出 = 0)(存储了 0%)
    正在添加: com/yp/(输入 = 0) (输出 = 0)(存储了 0%)
    正在添加: com/yp/test/(输入 = 0) (输出 = 0)(存储了 0%)
    正在添加: com/yp/test/entity/(输入 = 0) (输出 = 0)(存储了 0%)
    正在添加: com/yp/test/entity/Cat.class(输入 = 545) (输出 = 319)(压缩了 41%)
    正在添加: com/yp/test/HelloWorld.class(输入 = 844) (输出 = 487)(压缩了 42%)

直接运行

    java -jar h.jar
    
    额外发现 
    指定了Main类后，jar包里面的 META-INF/MANIFEST.MF 是这样的， 比原来多了一行Main-Class….
    Manifest-Version: 1.0
    Created-By: 1.8.0 (Oracle Corporation)
    Main-Class: com.yp.test.HelloWorld

如果类里有引用jar包呢?

先下一个jar包 这里直接下 log4j 
    
    * main函数改成
    
    import com.yp.test.entity.Cat;
    import org.apache.log4j.Logger;
    
    public class HelloWorld {
    
        static Logger log = Logger.getLogger(HelloWorld.class);
    
        public static void main(String[] args) {
            Cat c = new Cat("keyboard");
            log.info("这是log4j");
            System.out.println("hello," + c.getName());
        }
    
    }

现的文件是这样的


```
├─build
├─lib
│      log4j-1.2.17.jar
│
└─src
    └─com
        └─yp
            └─test
                │  HelloWorld.java
                │
                └─entity
                        Cat.java
```


    这个时候 javac命令要接上 -cp ./lib/*.jar
    E:\codeplace\n_learn\java\javacmd>javac -encoding "utf8" src/com/yp/test/HelloWorld.java -sourcepath src -d build -g -cp ./lib/*.jar


    运行要加上-cp, -cp 选项貌似会把工作目录给换了， 所以要加上 ;../build
    E:\codeplace\n_learn\java\javacmd\build>java -cp ../lib/log4j-1.2.17.jar;../build com.yp.test.HelloWorld

结果:

    log4j:WARN No appenders could be found for logger(com.yp.test.HelloWorld).
    log4j:WARN Please initialize the log4j system properly.
    log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
    hello,keyboard

由于没有 log4j的配置文件，所以提示上面的问题,往 build 里面加上 log4j.xml

    <?xml version="1.0" encoding="UTF-8" ?>
    <!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">
    <log4j:configuration xmlns:log4j='http://jakarta.apache.org/log4j/'>
        <appender name="stdout" class="org.apache.log4j.ConsoleAppender">
            <layout class="org.apache.log4j.PatternLayout">
                <param name="ConversionPattern" value="%d{ABSOLUTE} %-5p [%c{1}] %m%n" />
            </layout>
        </appender>
    
        <root>
            <level value="info" />
            <appender-ref ref="stdout" />
        </root>
    </log4j:configuration>

再运行

    E:\codeplace\n_learn\java\javacmd>java -cp lib/log4j-1.2.17.jar;build com.yp.tes t.HelloWorld
    15:19:57,359 INFO  [HelloWorld] 这是log4j
    hello,keyboard

说明: 
这个log4j配置文件，习惯的做法是放在src目录下, 在编译过程中 copy到build中的,但根据ant的做法，不是用javac的，而是用来处理,我猜测javac是不能copy的，如果想在命令行直接 使用，应该是用cp命令主动去执行 copy操作

ok 一个简单的java 工程就运行完了
但是  貌似有些繁琐,  需要手动键入 java文件 以及相应的jar包 很是麻烦,
so 可以用 shell 来脚本来简化相关操作 
shell 文件整理如下:

    #!/bin/bash  
    echo "build start"  
      
    JAR_PATH=libs  
    BIN_PATH=bin  
    SRC_PATH=src  
      
    # java文件列表目录  
    SRC_FILE_LIST_PATH=src/sources.list  
      
    #生所有的java文件列表 放入列表文件中 
    rm -f $SRC_PATH/sources  
    find $SRC_PATH/ -name *.java > $SRC_FILE_LIST_PATH  
      
    #删除旧的编译文件 生成bin目录  
    rm -rf $BIN_PATH/  
    mkdir $BIN_PATH/  
      
    #生成依赖jar包 列表  
    for file in  ${JAR_PATH}/*.jar;  
    do  
    jarfile=${jarfile}:${file}  
    done  
    echo "jarfile = "$jarfile  
      
    #编译 通过-cp指定所有的引用jar包，将src下的所有java文件进行编译
    javac -d $BIN_PATH/ -cp $jarfile @$SRC_FILE_LIST_PATH  
      
    #运行 通过-cp指定所有的引用jar包，指定入口函数运行
    java -cp $BIN_PATH$jarfile com.zuiapps.danmaku.server.Main  

> 有一点需要注意的是,  javac -d $BIN_PATH/ -cp $jarfile @$SRC_FILE_LIST_PATH
> 在要编译的文件很多时候，一个个敲命令会显得很长，也不方便修改，

> 可以把要编译的源文件列在文件中，在文件名前加@，这样就可以对多个文件进行编译，

> 以上就是吧java文件放到 $SRC_FILE_LIST_PATH 中去了

    编译 :
         1. 需要编译所有的java文件
         2. 依赖的java 包都需要加入到 classpath 中去
         3. 最后设置 编译后的 class 文件存放目录  即 -d bin/
         4. java文件过多是可以使用  @$SRC_FILE_LIST_PATH 把他们放到一个文件中去
    运行:
       1.需要吧 编译时设置的bin目录和 所有jar包加入到 classpath 中去


​    
## javap 的使用

> javap是jdk自带的一个工具，可以对代码反编译，也可以查看java编译器生成的字节码。
>
> 情况下，很少有人使用javap对class文件进行反编译，因为有很多成熟的反编译工具可以使用，比如jad。但是，javap还可以查看java编译器为我们生成的字节码。通过它，可以对照源代码和字节码，从而了解很多编译器内部的工作。
>
> 
>
> javap命令分解一个class文件，它根据options来决定到底输出什么。如果没有使用options,那么javap将会输出包，类里的protected和public域以及类里的所有方法。javap将会把它们输出在标准输出上。来看这个例子，先编译(javac)下面这个类。

    import java.awt.*;
    import java.applet.*;
     
    public class DocFooter extends Applet {
            String date;
            String email;
     
            public void init() {
                    resize(500,100);
                    date = getParameter("LAST_UPDATED");
                    email = getParameter("EMAIL");
            }
    }

在命令行上键入javap DocFooter后，输出结果如下


Compiled from "DocFooter.java"

    public class DocFooter extends java.applet.Applet {
      java.lang.String date;
      java.lang.String email;
      public DocFooter();
      public void init();
    }

如果加入了-c，即javap -c DocFooter，那么输出结果如下

Compiled from "DocFooter.java"

    public class DocFooter extends java.applet.Applet {
      java.lang.String date;
     
      java.lang.String email;
     
      public DocFooter();
        Code:
           0: aload_0       
           1: invokespecial #1                  // Method java/applet/Applet."<init>":()V
           4: return       
     
      public void init();
        Code:
           0: aload_0       
           1: sipush        500
           4: bipush        100
           6: invokevirtual #2                  // Method resize:(II)V
           9: aload_0       
          10: aload_0       
          11: ldc           #3                  // String LAST_UPDATED
          13: invokevirtual #4                  // Method getParameter:(Ljava/lang/String;)Ljava/lang/String;
          16: putfield      #5                  // Field date:Ljava/lang/String;
          19: aload_0       
          20: aload_0       
          21: ldc           #6                  // String EMAIL
          23: invokevirtual #4                  // Method getParameter:(Ljava/lang/String;)Ljava/lang/String;
          26: putfield      #7                  // Field email:Ljava/lang/String;
          29: return       
    
    }
上面输出的内容就是字节码。

用法摘要

-help 帮助
-l 输出行和变量的表
-public 只输出public方法和域
-protected 只输出public和protected类和成员
-package 只输出包，public和protected类和成员，这是默认的
-p -private 输出所有类和成员
-s 输出内部类型签名
-c 输出分解后的代码，例如，类中每一个方法内，包含java字节码的指令，
-verbose 输出栈大小，方法参数的个数
-constants 输出静态final常量
总结

javap可以用于反编译和查看编译器编译后的字节码。平时一般用javap -c比较多，该命令用于列出每个方法所执行的JVM指令，并显示每个方法的字节码的实际作用。可以通过字节码和源代码的对比，深入分析java的编译原理，了解和解决各种Java原理级别的问题。

## 参考文章

https://blog.csdn.net/Anbernet/article/details/81449390
https://www.cnblogs.com/luobiao320/p/7975442.html
https://www.jianshu.com/p/f7330dbdc051
https://www.jianshu.com/p/6a8997560b05
https://blog.csdn.net/w372426096/article/details/81664431
https://blog.csdn.net/qincidong/article/details/82492140

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。 

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
