# 目录
  * [IO概述](#io概述)
    * [什么是Java IO流](#什么是java-io流)
    * [IO文件](#io文件)
    * [字符流和字节流](#字符流和字节流)
    * [IO管道](#io管道)
    * [Java IO：网络](#java-io：网络)
    * [字节和字符数组](#字节和字符数组)
    * [System.in, System.out, System.err](#systemin-systemout-systemerr)
    * [字符流的Buffered和Filter](#字符流的buffered和filter)
  * [JavaIO流面试题](#javaio流面试题)
    * [什么是IO流？](#什么是io流？)
    * [字节流和字符流的区别。](#字节流和字符流的区别。)
    * [Java中流类的超类主要由那些？](#java中流类的超类主要由那些？)
    * [FileInputStream和FileOutputStream是什么？](#fileinputstream和fileoutputstream是什么？)
    * [System.out.println()是什么？](#systemoutprintln是什么？)
    * [什么是Filter流？](#什么是filter流？)
    * [有哪些可用的Filter流？](#有哪些可用的filter流？)
    * [在文件拷贝的时候，那一种流可用提升更多的性能？](#在文件拷贝的时候，那一种流可用提升更多的性能？)
    * [说说管道流(Piped Stream)](#说说管道流piped-stream)
    * [说说File类](#说说file类)
    * [说说RandomAccessFile?](#说说randomaccessfile)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)
---
title: 夯实Java基础系列16：一文读懂Java IO流和常见面试题
date: 2019-9-16 15:56:26 # 文章生成时间，一般不改
categories:
    - Java技术江湖
    - Java基础
tags:
    - Java IO流
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


本文参考

并发编程网 – ifeve.com

## IO概述

> 在这一小节，我会试着给出Java IO(java.io)包下所有类的概述。更具体地说，我会根据类的用途对类进行分组。这个分组将会使你在未来的工作中，进行类的用途判定时，或者是为某个特定用途选择类时变得更加容易。


​    
**输入和输出**

    术语“输入”和“输出”有时候会有一点让人疑惑。一个应用程序的输入往往是另外一个应用程序的输出
    
    那么OutputStream流到底是一个输出到目的地的流呢，还是一个产生输出的流？InputStream流到底会不会输出它的数据给读取数据的程序呢？就我个人而言，在第一天学习Java IO的时候我就感觉到了一丝疑惑。
    
    为了消除这个疑惑，我试着给输入和输出起一些不一样的别名，让它们从概念上与数据的来源和数据的流向相联系。

Java的IO包主要关注的是从原始数据源的读取以及输出原始数据到目标媒介。以下是最典型的数据源和目标媒介：
    
    文件
    管道
    网络连接
    内存缓存
    System.in, System.out, System.error(注：Java标准输入、输出、错误输出)
下面这张图描绘了一个程序从数据源读取数据，然后将数据输出到其他媒介的原理：

![](http://ifeve.com/wp-content/uploads/2014/10/%E6%97%A0%E6%A0%87%E9%A2%981.png)

**流**

    在Java IO中，流是一个核心的概念。流从概念上来说是一个连续的数据流。你既可以从流中读取数据，也可以往流中写数据。流与数据源或者数据流向的媒介相关联。在Java IO中流既可以是字节流(以字节为单位进行读写)，也可以是字符流(以字符为单位进行读写)。

类InputStream, OutputStream, Reader 和Writer
一个程序需要InputStream或者Reader从数据源读取数据，需要OutputStream或者Writer将数据写入到目标媒介中。以下的图说明了这一点：

![](http://ifeve.com/wp-content/uploads/2014/10/%E6%97%A0%E6%A0%87%E9%A2%982.png)

InputStream和Reader与数据源相关联，OutputStream和writer与目标媒介相关联。

**Java IO的用途和特征**

Java IO中包含了许多InputStream、OutputStream、Reader、Writer的子类。这样设计的原因是让每一个类都负责不同的功能。这也就是为什么IO包中有这么多不同的类的缘故。各类用途汇总如下：
    
    文件访问
    网络访问
    内存缓存访问
    线程内部通信(管道)
    缓冲
    过滤
    解析
    读写文本 (Readers / Writers)
    读写基本类型数据 (long, int etc.)
    读写对象

当通读过Java IO类的源代码之后，我们很容易就能了解这些用途。这些用途或多或少让我们更加容易地理解，不同的类用于针对不同业务场景。

Java IO类概述表
已经讨论了数据源、目标媒介、输入、输出和各类不同用途的Java IO类，接下来是一张通过输入、输出、基于字节或者字符、以及其他比如缓冲、解析之类的特定用途划分的大部分Java IO类的表格。

![](http://ifeve.com/wp-content/uploads/2014/10/QQ%E6%88%AA%E5%9B%BE20141020174145.png)

Java IO类图

![](https://images.cnblogs.com/cnblogs_com/davidgu/java_io_hierarchy.jpg)

### 什么是Java IO流

Java IO流是既可以从中读取，也可以写入到其中的数据流。正如这个系列教程之前提到过的，流通常会与数据源、数据流向目的地相关联，比如文件、网络等等。

流和数组不一样，不能通过索引读写数据。在流中，你也不能像数组那样前后移动读取数据，除非使用RandomAccessFile 处理文件。流仅仅只是一个连续的数据流。

某些类似PushbackInputStream 流的实现允许你将数据重新推回到流中，以便重新读取。然而你只能把有限的数据推回流中，并且你不能像操作数组那样随意读取数据。流中的数据只能够顺序访问。
>
> Java IO流通常是基于字节或者基于字符的。字节流通常以“stream”命名，比如InputStream和OutputStream。除了DataInputStream 和DataOutputStream 还能够读写int, long, float和double类型的值以外，其他流在一个操作时间内只能读取或者写入一个原始字节。
>
> 字符流通常以“Reader”或者“Writer”命名。字符流能够读写字符(比如Latin1或者Unicode字符)。可以浏览Java Readers and Writers获取更多关于字符流输入输出的信息。

**InputStream**

java.io.InputStream类是所有Java IO输入流的基类。如果你正在开发一个从流中读取数据的组件，请尝试用InputStream替代任何它的子类(比如FileInputStream)进行开发。这么做能够让你的代码兼容任何类型而非某种确定类型的输入流。

**组合流**

你可以将流整合起来以便实现更高级的输入和输出操作。比如，一次读取一个字节是很慢的，所以可以从磁盘中一次读取一大块数据，然后从读到的数据块中获取字节。为了实现缓冲，可以把InputStream包装到BufferedInputStream中。

代码示例
    InputStream input = new BufferedInputStream(new FileInputStream("c:\\data\\input-file.txt"));
    
> 缓冲同样可以应用到OutputStream中。你可以实现将大块数据批量地写入到磁盘(或者相应的流)中，这个功能由BufferedOutputStream实现。
>
> 缓冲只是通过流整合实现的其中一个效果。你可以把InputStream包装到PushbackInputStream中，之后可以将读取过的数据推回到流中重新读取，在解析过程中有时候这样做很方便。或者，你可以将两个InputStream整合成一个SequenceInputStream。
>
> 将不同的流整合到一个链中，可以实现更多种高级操作。通过编写包装了标准流的类，可以实现你想要的效果和过滤器。

### IO文件

在Java应用程序中，文件是一种常用的数据源或者存储数据的媒介。所以这一小节将会对Java中文件的使用做一个简短的概述。这篇文章不会对每一个技术细节都做出解释，而是会针对文件存取的方法提供给你一些必要的知识点。在之后的文章中，将会更加详细地描述这些方法或者类，包括方法示例等等。


**通过Java IO读文件**

    如果你需要在不同端之间读取文件，你可以根据该文件是二进制文件还是文本文件来选择使用FileInputStream或者FileReader。
    
    这两个类允许你从文件开始到文件末尾一次读取一个字节或者字符，或者将读取到的字节写入到字节数组或者字符数组。你不必一次性读取整个文件，相反你可以按顺序地读取文件中的字节和字符。

如果你需要跳跃式地读取文件其中的某些部分，可以使用RandomAccessFile。

**通过Java IO写文件**

    如果你需要在不同端之间进行文件的写入，你可以根据你要写入的数据是二进制型数据还是字符型数据选用FileOutputStream或者FileWriter。
    
    你可以一次写入一个字节或者字符到文件中，也可以直接写入一个字节数组或者字符数据。数据按照写入的顺序存储在文件当中。

**通过Java IO随机存取文件**

正如我所提到的，你可以通过RandomAccessFile对文件进行随机存取。
    
    随机存取并不意味着你可以在真正随机的位置进行读写操作，它只是意味着你可以跳过文件中某些部分进行操作，并且支持同时读写，不要求特定的存取顺序。
    
    这使得RandomAccessFile可以覆盖一个文件的某些部分、或者追加内容到它的末尾、或者删除它的某些内容，当然它也可以从文件的任何位置开始读取文件。

下面是具体例子：

    @Test
        //文件流范例，打开一个文件的输入流，读取到字节数组，再写入另一个文件的输出流
        public void test1() {
            try {
                FileInputStream fileInputStream = new FileInputStream(new File("a.txt"));
                FileOutputStream fileOutputStream = new FileOutputStream(new File("b.txt"));
                byte []buffer = new byte[128];
                while (fileInputStream.read(buffer) != -1) {
                    fileOutputStream.write(buffer);
                }
                //随机读写，通过mode参数来决定读或者写
                RandomAccessFile randomAccessFile = new RandomAccessFile(new File("c.txt"), "rw");
            } catch (FileNotFoundException e) {
                e.printStackTrace();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
### 字符流和字节流

Java IO的Reader和Writer除了基于字符之外，其他方面都与InputStream和OutputStream非常类似。他们被用于读写文本。InputStream和OutputStream是基于字节的，还记得吗？

Reader
Reader类是Java IO中所有Reader的基类。子类包括BufferedReader，PushbackReader，InputStreamReader，StringReader和其他Reader。

Writer
Writer类是Java IO中所有Writer的基类。子类包括BufferedWriter和PrintWriter等等。

这是一个简单的Java IO Reader的例子：

    Reader reader = new FileReader("c:\\data\\myfile.txt");
    
    int data = reader.read();
    
    while(data != -1){
    
        char dataChar = (char) data;
    
        data = reader.read();
    
    }

你通常会使用Reader的子类，而不会直接使用Reader。Reader的子类包括InputStreamReader，CharArrayReader，FileReader等等。可以查看Java IO概述浏览完整的Reader表格。

**整合Reader与InputStream**

一个Reader可以和一个InputStream相结合。如果你有一个InputStream输入流，并且想从其中读取字符，可以把这个InputStream包装到InputStreamReader中。把InputStream传递到InputStreamReader的构造函数中：

    Reader reader = new InputStreamReader(inputStream);
在构造函数中可以指定解码方式。

**Writer**

Writer类是Java IO中所有Writer的基类。子类包括BufferedWriter和PrintWriter等等。这是一个Java IO Writer的例子：

    Writer writer = new FileWriter("c:\\data\\file-output.txt"); 
    
    writer.write("Hello World Writer"); 
    
    writer.close();

同样，你最好使用Writer的子类，不需要直接使用Writer，因为子类的实现更加明确，更能表现你的意图。常用子类包括OutputStreamWriter，CharArrayWriter，FileWriter等。Writer的write(int c)方法，会将传入参数的低16位写入到Writer中，忽略高16位的数据。

**整合Writer和OutputStream**

与Reader和InputStream类似，一个Writer可以和一个OutputStream相结合。把OutputStream包装到OutputStreamWriter中，所有写入到OutputStreamWriter的字符都将会传递给OutputStream。这是一个OutputStreamWriter的例子：

    Writer writer = new OutputStreamWriter(outputStream);

### IO管道

Java IO中的管道为运行在同一个JVM中的两个线程提供了通信的能力。所以管道也可以作为数据源以及目标媒介。

你不能利用管道与不同的JVM中的线程通信(不同的进程)。在概念上，Java的管道不同于Unix/Linux系统中的管道。在Unix/Linux中，运行在不同地址空间的两个进程可以通过管道通信。在Java中，通信的双方应该是运行在同一进程中的不同线程。

通过Java IO创建管道

    可以通过Java IO中的PipedOutputStream和PipedInputStream创建管道。一个PipedInputStream流应该和一个PipedOutputStream流相关联。
    
    一个线程通过PipedOutputStream写入的数据可以被另一个线程通过相关联的PipedInputStream读取出来。

Java IO管道示例
这是一个如何将PipedInputStream和PipedOutputStream关联起来的简单例子：

    //使用管道来完成两个线程间的数据点对点传递
        @Test
        public void test2() throws IOException {
            PipedInputStream pipedInputStream = new PipedInputStream();
            PipedOutputStream pipedOutputStream = new PipedOutputStream(pipedInputStream);
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        pipedOutputStream.write("hello input".getBytes());
                        pipedOutputStream.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        byte []arr = new byte[128];
                        while (pipedInputStream.read(arr) != -1) {
                            System.out.println(Arrays.toString(arr));
                        }
                        pipedInputStream.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }).start();

管道和线程
请记得，当使用两个相关联的管道流时，务必将它们分配给不同的线程。read()方法和write()方法调用时会导致流阻塞，这意味着如果你尝试在一个线程中同时进行读和写，可能会导致线程死锁。

管道的替代
除了管道之外，一个JVM中不同线程之间还有许多通信的方式。实际上，线程在大多数情况下会传递完整的对象信息而非原始的字节数据。但是，如果你需要在线程之间传递字节数据，Java IO的管道是一个不错的选择。

### Java IO：网络

Java中网络的内容或多或少的超出了Java IO的范畴。关于Java网络更多的是在我的Java网络教程中探讨。但是既然网络是一个常见的数据来源以及数据流目的地，并且因为你使用Java IO的API通过网络连接进行通信，所以本文将简要的涉及网络应用。


当两个进程之间建立了网络连接之后，他们通信的方式如同操作文件一样：利用InputStream读取数据，利用OutputStream写入数据。换句话来说，Java网络API用来在不同进程之间建立网络连接，而Java IO则用来在建立了连接之后的进程之间交换数据。

基本上意味着如果你有一份能够对文件进行写入某些数据的代码，那么这些数据也可以很容易地写入到网络连接中去。你所需要做的仅仅只是在代码中利用OutputStream替代FileOutputStream进行数据的写入。因为FileOutputStream是OuputStream的子类，所以这么做并没有什么问题。

    //从网络中读取字节流也可以直接使用OutputStream
    public void test3() {
        //读取网络进程的输出流
        OutputStream outputStream = new OutputStream() {
            @Override
            public void write(int b) throws IOException {
            }
        };
    }
    public void process(OutputStream ouput) throws IOException {
        //处理网络信息
        //do something with the OutputStream
    }

### 字节和字符数组


从InputStream或者Reader中读入数组

从OutputStream或者Writer中写数组

在java中常用字节和字符数组在应用中临时存储数据。而这些数组又是通常的数据读取来源或者写入目的地。如果你需要在程序运行时需要大量读取文件里的内容，那么你也可以把一个文件加载到数组中。

前面的例子中，字符数组或字节数组是用来缓存数据的临时存储空间，不过它们同时也可以作为数据来源或者写入目的地。
举个例子：

    //字符数组和字节数组在io过程中的作用
        public void test4() {
            //arr和brr分别作为数据源
            char []arr = {'a','c','d'};
            CharArrayReader charArrayReader = new CharArrayReader(arr);
            byte []brr = {1,2,3,4,5};
            ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(brr);
        }

### System.in, System.out, System.err    

System.in, System.out, System.err这3个流同样是常见的数据来源和数据流目的地。使用最多的可能是在控制台程序里利用System.out将输出打印到控制台上。

JVM启动的时候通过Java运行时初始化这3个流，所以你不需要初始化它们(尽管你可以在运行时替换掉它们)。


    System.in
    System.in是一个典型的连接控制台程序和键盘输入的InputStream流。通常当数据通过命令行参数或者配置文件传递给命令行Java程序的时候，System.in并不是很常用。图形界面程序通过界面传递参数给程序，这是一块单独的Java IO输入机制。
    
    System.out
    System.out是一个PrintStream流。System.out一般会把你写到其中的数据输出到控制台上。System.out通常仅用在类似命令行工具的控制台程序上。System.out也经常用于打印程序的调试信息(尽管它可能并不是获取程序调试信息的最佳方式)。
    
    System.err
    System.err是一个PrintStream流。System.err与System.out的运行方式类似，但它更多的是用于打印错误文本。一些类似Eclipse的程序，为了让错误信息更加显眼，会将错误信息以红色文本的形式通过System.err输出到控制台上。

System.out和System.err的简单例子：
这是一个System.out和System.err结合使用的简单示例：

     //测试System.in, System.out, System.err    
        public static void main(String[] args) {
            int in = new Scanner(System.in).nextInt();
            System.out.println(in);
            System.out.println("out");
            System.err.println("err");
            //输入10，结果是
    //        err（红色）
    //        10
    //        out
        }


### 字符流的Buffered和Filter

BufferedReader能为字符输入流提供缓冲区，可以提高许多IO处理的速度。你可以一次读取一大块的数据，而不需要每次从网络或者磁盘中一次读取一个字节。特别是在访问大量磁盘数据时，缓冲通常会让IO快上许多。

BufferedReader和BufferedInputStream的主要区别在于，BufferedReader操作字符，而BufferedInputStream操作原始字节。只需要把Reader包装到BufferedReader中，就可以为Reader添加缓冲区(译者注：默认缓冲区大小为8192字节，即8KB)。代码如下：

    Reader input = new BufferedReader(new FileReader("c:\\data\\input-file.txt"));

你也可以通过传递构造函数的第二个参数，指定缓冲区大小，代码如下：

    Reader input = new BufferedReader(new FileReader("c:\\data\\input-file.txt"), 8 * 1024);

这个例子设置了8KB的缓冲区。最好把缓冲区大小设置成1024字节的整数倍，这样能更高效地利用内置缓冲区的磁盘。

除了能够为输入流提供缓冲区以外，其余方面BufferedReader基本与Reader类似。BufferedReader还有一个额外readLine()方法，可以方便地一次性读取一整行字符。

**BufferedWriter**

与BufferedReader类似，BufferedWriter可以为输出流提供缓冲区。可以构造一个使用默认大小缓冲区的BufferedWriter(译者注：默认缓冲区大小8 * 1024B)，代码如下：

    Writer writer = new BufferedWriter(new FileWriter("c:\\data\\output-file.txt"));

也可以手动设置缓冲区大小，代码如下：

    Writer writer = new BufferedWriter(new FileWriter("c:\\data\\output-file.txt"), 8 * 1024);

为了更好地使用内置缓冲区的磁盘，同样建议把缓冲区大小设置成1024的整数倍。除了能够为输出流提供缓冲区以外，其余方面BufferedWriter基本与Writer类似。类似地，BufferedWriter也提供了writeLine()方法，能够把一行字符写入到底层的字符输出流中。


**值得注意是，你需要手动flush()方法确保写入到此输出流的数据真正写入到磁盘或者网络中。**

**FilterReader**

与FilterInputStream类似，FilterReader是实现自定义过滤输入字符流的基类，基本上它仅仅只是简单覆盖了Reader中的所有方法。

就我自己而言，我没发现这个类明显的用途。除了构造函数取一个Reader变量作为参数之外，我没看到FilterReader任何对Reader新增或者修改的地方。如果你选择继承FilterReader实现自定义的类，同样也可以直接继承自Reader从而避免额外的类层级结构。

## JavaIO流面试题

### 什么是IO流？
它是一种数据的流从源头流到目的地。比如文件拷贝，输入流和输出流都包括了。输入流从文件中读取数据存储到进程(process)中，输出流从进程中读取数据然后写入到目标文件。

### 字节流和字符流的区别。
字节流在JDK1.0中就被引进了，用于操作包含ASCII字符的文件。JAVA也支持其他的字符如Unicode，为了读取包含Unicode字符的文件，JAVA语言设计者在JDK1.1中引入了字符流。ASCII作为Unicode的子集，对于英语字符的文件，可以可以使用字节流也可以使用字符流。

### Java中流类的超类主要由那些？

java.io.InputStream
java.io.OutputStream
java.io.Reader
java.io.Writer

### FileInputStream和FileOutputStream是什么？
这是在拷贝文件操作的时候，经常用到的两个类。在处理小文件的时候，它们性能表现还不错，在大文件的时候，最好使用BufferedInputStream (或 BufferedReader) 和 BufferedOutputStream (或 BufferedWriter)

### System.out.println()是什么？
println是PrintStream的一个方法。out是一个静态PrintStream类型的成员变量，System是一个java.lang包中的类，用于和底层的操作系统进行交互。

### 什么是Filter流？
Filter Stream是一种IO流主要作用是用来对存在的流增加一些额外的功能，像给目标文件增加源文件中不存在的行数，或者增加拷贝的性能。

### 有哪些可用的Filter流？

在java.io包中主要由4个可用的filter Stream。两个字节filter stream，两个字符filter stream. 分别是FilterInputStream, FilterOutputStream, FilterReader and FilterWriter.这些类是抽象类，不能被实例化的。



### 在文件拷贝的时候，那一种流可用提升更多的性能？
在字节流的时候，使用BufferedInputStream和BufferedOutputStream。
在字符流的时候，使用BufferedReader 和 BufferedWriter

### 说说管道流(Piped Stream)
有四种管道流， PipedInputStream, PipedOutputStream, PipedReader 和 PipedWriter.在多个线程或进程中传递数据的时候管道流非常有用。

### 说说File类
它不属于 IO流，也不是用于文件操作的，它主要用于知道一个文件的属性，读写权限，大小等信息。

### 说说RandomAccessFile?
它在java.io包中是一个特殊的类，既不是输入流也不是输出流，它两者都可以做到。他是Object的直接子类。通常来说，一个流只有一个功能，要么读，要么写。但是RandomAccessFile既可以读文件，也可以写文件。 DataInputStream 和 DataOutStream有的方法，在RandomAccessFile中都存在。

## 参考文章

https://www.imooc.com/article/24305
https://www.cnblogs.com/UncleWang001/articles/10454685.html
https://www.cnblogs.com/Jixiangwei/p/Java.html
https://blog.csdn.net/baidu_37107022/article/details/76890019

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。 

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
