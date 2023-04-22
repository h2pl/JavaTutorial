# 目录
* [简介](#简介)
* [概述](#概述)
* [好处](#好处)
* [Junit单元测试](#junit单元测试)
    * [1 简介](#1-简介)
    * [2 特点](#2-特点)
    * [3 内容](#3-内容)
        * [3.1 注解](#31-注解)
        * [3.2 断言](#32-断言)
    * [4 JUnit 3.X 和 JUnit 4.X 的区别](#4-junit-3x-和-junit-4x-的区别)
        * [4.1 JUnit 3.X](#41-junit-3x)
        * [4.2 JUnit 4.X](#42-junit-4x)
        * [4.3 特别提醒](#43-特别提醒)
    * [5 测试示例](#5-测试示例)
        * [5.1 示例一：简单的 JUnit 3.X 测试](#51-示例一：简单的-junit-3x-测试)
    * [6 个人建议](#6-个人建议)
* [8 大单元测试框架](#8-大单元测试框架)



本文转载自互联网，侵删
本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本系列文章将同步到我的个人博客：
> www.how2playlife.com

更多Java技术文章将陆续在微信公众号【Java技术江湖】更新，敬请关注。

本文是《走进JavaWeb技术世界》系列博文的其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个JavaWeb技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

**文末赠送8000G的Java架构师学习资料，需要的朋友可以到文末了解领取方式，资料包括Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源）**
<!-- more -->


## 简介

测试在软件开发中是一个很重要的方面，良好的测试可以在很大程度决定一个应用的命运。
软件测试中，主要有3大种类：

*   [单元测试](https://en.wikipedia.org/wiki/Unit_testing)
    单元测试主要是用于测试程序模块，确保代码运行正确。单元测试是由开发者编写并进行运行测试。一般使用的测试框架是[JUnit](http://junit.org/junit4/)或者[TestNG](https://github.com/cbeust/testng)。测试用例一般是针对_方法_级别的测试。
*   [集成测试](https://en.wikipedia.org/wiki/Integration_testing)
    集成测试用于检测系统是否能正常工作。集成测试也是由开发者共同进行测试，与单元测试专注测试个人代码组件不同的是，集成测试是系统进行跨组件测试。
*   [功能性测试](https://en.wikipedia.org/wiki/Functional_testing)
    功能性测试是一种质量保证过程以及基于测试软件组件的规范下的由输入得到输出的一种黑盒测试。功能性测试通常由不同的测试团队进行测试，测试用例的编写要遵循组件规范，然后根据测试输入得到的实际输出与期望值进行对比，判断功能是否正确运行。

## 概述

本文只对[单元测试](https://en.wikipedia.org/wiki/Unit_testing)进行介绍，主要介绍如何在[Android Studio](https://developer.android.com/studio/index.html?gclid=Cj0KCQjwgIPOBRDnARIsAHA1X3SC5vOHyIHQnIIfJ8hqJSuTiCG6p3u2ff_ti3EIVeCIGJLnP82YCKoaArSPEALw_wcB)下进行单元测试，单元测试使用的测试框架为[JUnit](http://junit.org/junit4/)

## 好处

可能目前仍有很大一部分开发者未使用[单元测试](https://en.wikipedia.org/wiki/Unit_testing)对他们的代码进行测试，一方面可能是觉得没有必要，因为即使没有进行单元测试，程序照样运行得很好；另一方面，也许有些人也认同单元测试的好处，但是由于需要额外的学习成本，所以很多人也是没有时间或者说是没有耐心进行学习······
这里我想说的是，如果大家去看下[github](https://www.jianshu.com/p/www.github.com)上目前主流的开源框架，star 数比较多的项目，一般都有很详尽的测试用例。所以说，单元测试对于我们的项目开发，还是挺有好处的。
至于单元测试的好处，我这里提及几点：

*   保证代码运行与我们预想的一样，代码正确性可以得到保证
*   程序运行出错时，有利于我们对错误进行查找（因为我们忽略我们测试通过的代码）
*   有利于提升代码架构设计（用于测试的用例应力求简单低耦合，因此编写代码的时候，开发者往往会为了对代码进行测试，将其他耦合的部分进行解耦处理）
    

## Junit单元测试

本文实例讲述了java单元测试JUnit框架原理与用法。分享给大家供大家参考，具体如下：

### 1 简介

JUnit是一个Java语言的单元测试框架，它由 Kent Beck 和 Erich Gamma 建立，逐渐成为 xUnit 家族中最为成功的一个。

JUnit有它自己的JUnit扩展生态圈，多数Java的开发环境都已经集成了JUnit作为单元测试的工具。在这里，一个单元可以是一个方法、类、包或者子系统。

因此，单元测试是指对代码中的最小可测试单元进行检查和验证，以便确保它们正常工作。例如，我们可以给予一定的输入测试输出是否是所希望得到的结果。在本篇博客中，作者将着重介绍 JUnit 4.X 版本的特性，这也是我们在日常开发中使用最多的版本。

### 2 特点

    JUnit提供了注释以及确定的测试方法；
    JUnit提供了断言用于测试预期的结果；
    JUnit测试优雅简洁不需要花费太多的时间；
    JUnit测试让大家可以更快地编写代码并且提高质量；
    JUnit测试可以组织成测试套件包含测试案例，甚至其他测试套件；
    Junit显示测试进度，如果测试是没有问题条形是绿色的，测试失败则会变成红色；
    JUnit测试可以自动运行，检查自己的结果，并提供即时反馈，没有必要通过测试结果报告来手动梳理。

### 3 内容

#### 3.1 注解

@Test ：该注释表示，用其附着的公共无效方法（即用public修饰的void类型的方法 ）可以作为一个测试用例；

@Before ：该注释表示，用其附着的方法必须在类中的每个测试之前执行，以便执行测试某些必要的先决条件；

@BeforeClass ：该注释表示，用其附着的静态方法必须执行一次并在类的所有测试之前，发生这种情况时一般是测试计算共享配置方法，如连接到数据库；

@After ：该注释表示，用其附着的方法在执行每项测试后执行，如执行每一个测试后重置某些变量，删除临时变量等；

@AfterClass ：该注释表示，当需要执行所有的测试在JUnit测试用例类后执行，AfterClass注解可以使用以清理建立方法，如断开数据库连接，注意：附有此批注（类似于BeforeClass）的方法必须定义为静态；

@Ignore ：该注释表示，当想暂时禁用特定的测试执行可以使用忽略注释，每个被注解为@Ignore的方法将不被执行。


````
/
* JUnit 注解示例
*/
@Test
public void testYeepay(){
  Syetem.out.println("用@Test标示测试方法！");
}
@AfterClass
public static void paylus(){
  Syetem.out.println("用@AfterClass标示的方法在测试用例类执行完之后！");
}

````





#### 3.2 断言

在这里，作者将介绍一些断言方法，所有这些方法都来自 org.junit.Assert 类，其扩展了 java.lang.Object 类并为它们提供编写测试，以便检测故障。简而言之，我们就是通过断言方法来判断实际结果与我们预期的结果是否相同，如果相同，则测试成功，反之，则测试失败。
````
void assertEquals([String message], expected value, actual value)：断言两个值相等，值的类型可以为int、short、long、byte、char 或者
java.lang.Object，其中第一个参数是一个可选的字符串消息；
void assertTrue([String message], boolean condition)：断言一个条件为真；
void assertFalse([String message],boolean condition)：断言一个条件为假；
void assertNotNull([String message], java.lang.Object object)：断言一个对象不为空(null)；
void assertNull([String message], java.lang.Object object)：断言一个对象为空(null)；
void assertSame([String message], java.lang.Object expected, java.lang.Object actual)：断言两个对象引用相同的对象；
void assertNotSame([String message], java.lang.Object unexpected, java.lang.Object actual)：断言两个对象不是引用同一个对象；
void assertArrayEquals([String message], expectedArray, resultArray)：断言预期数组和结果数组相等，数组的类型可以为int、long、short、char、byte 或者 java.lang.Object
````
### 4 JUnit 3.X 和 JUnit 4.X 的区别

#### 4.1 JUnit 3.X

（1）使用 JUnit 3.X 版本进行单元测试时，测试类必须要继承于 TestCase 父类；
（2）测试方法需要遵循的原则：

① public的；
② void的；
③ 无方法参数；
④方法名称必须以 test 开头；

（3）不同的测试用例之间一定要保持完全的独立性，不能有任何的关联；

（4）要掌握好测试方法的顺序，不能依赖于测试方法自己的执行顺序。

````
/
* 用 JUnit 3.X 进行测试
*/
import junit.framework.Assert;
import junit.framework.TestCase;
public class TestOperation extends TestCase {
  private Operation operation;
  public TestOperation(String name) { // 构造函数
    super(name);
  }
  @Override
  public void setUp() throws Exception { // 在每个测试方法执行 [之前] 都会被调用，多用于初始化
    System.out.println("欢迎使用Junit进行单元测试...");
    operation = new Operation();
  }
  @Override
  public void tearDown() throws Exception { // 在每个测试方法执行 [之后] 都会被调用，多用于释放资源
    System.out.println("Junit单元测试结束...");
  }
  public void testDivideByZero() {
    Throwable te = null;
    try {
      operation.divide(6, 0);
      Assert.fail("测试失败"); //断言失败
    } catch (Exception e) {
      e.printStackTrace();
      te = e;
    }
    Assert.assertEquals(Exception.class, te.getClass());
    Assert.assertEquals("除数不能为 0 ", te.getMessage());
  }
}
````

#### 4.2 JUnit 4.X

（1）使用 JUnit 4.X 版本进行单元测试时，不用测试类继承TestCase父类；
（2）JUnit 4.X 版本，引用了注解的方式进行单元测试；
（3）JUnit 4.X 版本我们常用的注解包括：

@Before注解：与JUnit 3.X 中的 setUp() 方法功能一样，在每个测试方法之前执行，多用于初始化；

@After注解：与 JUnit 3.X 中的 tearDown() 方法功能一样，在每个测试方法之后执行，多用于释放资源；

@Test(timeout = xxx)注解：设置当前测试方法在一定时间内运行完，否则返回错误；

@Test(expected = Exception.class)注解：设置被测试的方法是否有异常抛出。抛出异常类型为：Exception.class；

此外，我们可以通过阅读上面的第二部分“2 注解”了解更多的注解。

````
/
* 用 JUnit 4.X 进行测试
*/
import static org.junit.Assert.*;
import org.junit.After;
import org.junit.AfterClass;
import org.junit.Before;
import org.junit.BeforeClass;
import org.junit.Test;
public class TestOperation {
  private Operation operation;
  @BeforeClass
  public static void globalInit() { // 在所有方法执行之前执行
    System.out.println("@BeforeClass标注的方法，在所有方法执行之前执行...");
  }
  @AfterClass
  public static void globalDestory() { // 在所有方法执行之后执行
    System.out.println("@AfterClass标注的方法，在所有方法执行之后执行...");
  }
  @Before
  public void setUp() { // 在每个测试方法之前执行
    System.out.println("@Before标注的方法，在每个测试方法之前执行...");
    operation = new Operation();
  }
  @After
  public void tearDown() { // 在每个测试方法之后执行
    System.out.println("@After标注的方法，在每个测试方法之后执行...");
  }
  @Test(timeout=600)
  public void testAdd() { // 设置限定测试方法的运行时间 如果超出则返回错误
    System.out.println("测试 add 方法...");
    int result = operation.add(2, 3);
    assertEquals(5, result);
  }
  @Test
  public void testSubtract() {
    System.out.println("测试 subtract 方法...");
    int result = operation.subtract(1, 2);
    assertEquals(-1, result);
  }
  @Test
  public void testMultiply() {
    System.out.println("测试 multiply 方法...");
    int result = operation.multiply(2, 3);
    assertEquals(6, result);
  }
  @Test
  public void testDivide() {
    System.out.println("测试 divide 方法...");
    int result = 0;
    try {
      result = operation.divide(6, 2);
    } catch (Exception e) {
      fail();
    }
    assertEquals(3, result);
  }
  @Test(expected = Exception.class)
  public void testDivideAgain() throws Exception {
    System.out.println("测试 divide 方法，除数为 0 的情况...");
    operation.divide(6, 0);
    fail("test Error");
  }
  public static void main(String[] args) {
  }
}
````
#### 4.3 特别提醒

通过以上两个例子，我们已经可以大致知道 JUnit 3.X 和 JUnit 4.X 两个版本的区别啦！

首先，如果我们使用 JUnit 3.X，那么在我们写的测试类的时候，一定要继承 TestCase 类，但是如果我们使用 JUnit 4.X，则不需继承 TestCase 类，直接使用注解就可以啦！

在 JUnit 3.X 中，还强制要求测试方法的命名为“ testXxxx ”这种格式；

在 JUnit 4.X 中，则不要求测试方法的命名格式，但作者还是建议测试方法统一命名为“ testXxxx ”这种格式，简洁明了。

此外，在上面的两个示例中，我们只给出了测试类，但是在这之前，还应该有一个被测试类，也就是我们真正要实现功能的类。现在，作者将给出上面示例中被测试的类，即 Operation 类：

````
/
* 定义了加减乘除的法则
*/
public class Operation {
  public static void main(String[] args) {
    System.out.println("a + b = " + add(1,2));
    System.out.println("a - b = " + subtract(1,2));
    System.out.println("a * b = " + multiply(1,2));
    System.out.println("a / b = " + divide(4,2));
    System.out.println("a / b = " + divide(1,0));
  }
  public static int add(int a, int b) {
    return a + b;
  }
  public static int subtract(int a, int b) {
    return a - b;
  }
  public static int multiply(int a, int b) {
    return a * b;
  }
  public static int divide(int a, int b) {
    return a / b;
  }
}
````


### 5 测试示例

#### 5.1 示例一：简单的 JUnit 3.X 测试

````
import junit.framework.Test;
import junit.framework.TestCase;
import junit.framework.TestSuite;
import java.util.ArrayList;
import java.util.Collection;
/
 * 1、创建一个测试类，继承TestCase类
 */
public class SimpleTestDemo extends TestCase {
  public SimpleTestDemo(String name) {
    super(name);
  }
  /
   * 2、写一个测试方法，断言期望的结果
   */
  public void testEmptyCollection(){
    Collection collection = new ArrayList();
    assertTrue(collection.isEmpty());
  }
  /
   * 3、写一个suite()方法，它会使用反射动态的创建一个包含所有的testXxxx方法的测试套件
   */
  public static Test suit(){
    return new TestSuite(SimpleTestDemo.class);
  }
  /
   * 4、写一个main()方法，以文本运行器的方式方便的运行测试
   */
  public static void main(String[] args) {
    junit.textui.TestRunner.run(suit());
  }
}
````


### 6 个人建议

有些童鞋可能会有一些误解，认为写测试代码没有用，而且还会增大自己的压力，浪费时间。但事实上，写测试代码与否，还是有很大区别的，如果是在小的项目中，或许这种区别还不太明显，但如果在大型项目中，一旦出现错误或异常，用人力去排查的话，那将会浪费很多时间，而且还不一定排查的出来，但是如果用测试代码的话，JUnit 就是自动帮我们判断一些代码的结果正确与否，从而节省的时间将会远远超过你写测试代码的时间。

因此，个人建议：要养成编写测试代码的习惯，码一点、测一点；再码一点，再测一点，如此循环。在我们不断编写与测试代码的过程中，我们将会对类的行为有一个更为深入的了解，从而可以有效的提高我们的工作效率。下面，作者就给出一些具体的编写测试代码的技巧和较好的实践方法：

1\. 不要用 TestCase 的构造函数初始化 Fixture，而要用 setUp() 和 tearDown() 方法；
2\. 不要依赖或假定测试运行的顺序，因为 JUnit 会利用 Vector 保存测试方法，所以不同的平台会按不同的顺序从 Vector 中取出测试方法；
3\. 避免编写有副作用的 TestCase，例如：如果随后的测试依赖于某些特定的交易数据，就不要提交交易数据，只需要简单的回滚就可以了；
4\. 当继承一个测试类时，记得调用父类的 setUp() 和 tearDown() 方法；
5\. 将测试代码和工作代码放在一起，同步编译和更新；
6\. 测试类和测试方法应该有一致的命名方案，如在工作类名前加上 test 从而形成测试类名；
7\. 确保测试与时间无关，不要使用过期的数据进行测试，以至于导致在随后的维护过程中很难重现测试；
8\. 如果编写的软件面向国际市场，那么编写测试时一定要考虑国际化的因素；
9\. 尽可能地利用 JUnit 提供地 assert 和 fail 方法以及异常处理的方法，其可以使代码更为简洁；
10\. 测试要尽可能地小，执行速度快；
11\. 不要硬性规定数据文件的路径；
12\. 使用文档生成器做测试文档。

## 8 大单元测试框架
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4310fd169d4d464db44fa73f25a241c3.jpg)

1.Arquillian

Arquillian是一个基于JVM的高度可扩展的测试平台，允许开发人员创建Java的自动化集成，功能和验收测试。Arquillian允许你在运行态时执行测试。Arquillian可用于管理容器（或容器）的生命周期，绑定测试用例，依赖类和资源。它还能够将压缩包部署到容器中，并在容器中执行测试并捕获结果并创建报告。

Arquillian集成了熟悉的测试框架，如JUnit 4、TestNG 5，并允许使用现有的IDE启动测试。并且由于其模块化设计，它能够运行Ant和Maven测试插件。Arquillian目的是简化项目集成测试和功能测试的编写，让它们能像单元测试一样简单。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/8c78fe84d7fa46b9b6dd9fc8ab001e00.jpg)

2.JTEST

JTest也被称为“Parasoft JTest”，是Parasoft公司生产的自动化Java软件测试和静态分析软件。 JTest包括用于单元测试用例生成和执行，静态代码分析，数据流静态分析和度量分析，回归测试，运行时错误检测的功能。

还可以进行结对的代码审查流程自动化和运行时错误检测，例如：条件，异常，资源和内存泄漏，安全攻击漏洞等。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/582197f6e97549bd894380f2b7320cc9.jpg)

3.The Grinder

“The Grinder”是一个Java负载测试框架。并且通过使用大量负载注射器来为分布式测试提供便利。Grinder可以对具有Java API的任何内容加载测试。这包括HTTP Web服务器，SOAP、REST Web服务、应用程序服务器，包括自定义协议。测试脚本用强大的Jython和Clojure语言编写。Grinder的GUI控制台允许对多个负载注射器进行监控和控制，并自动管理客户端连接和Cookie，SSL，代理感知和连接限制。您可以在这里找到关于磨床功能的更多深入信息。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/2d5c47277d254ad783c08b5f03a42372_th.jpg)

4.TestNG

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0e1f6408b31d4665b4cc75b47750d58d.jpg)

TestNG受JUnit和NUnit的启发，是为Java编程语言而设计的测试框架。TestNG主要设计用于覆盖更广泛的测试类别，如单元，功能，端到端，集成等。它还引入了一些新功能，使其更强大，更易于使用，如：注解，运行在大线程池中进行各种策略测试，多线程安全验证代码测试，灵活的测试配置，数据驱动的参数测试支持等等。

TestNG有各种工具和插件（如Eclipse，IDEA，Maven等）支持。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/21bc4cc5fd924b16b29be4880f9cfeab_th.jpg)

5.JUnit

JUnit是为Java编程语言设计的单元测试框架。JUnit在测试驱动开发框架的开发中发挥了重要作用。它是单元测试框架之一，统称为由SUnit起源的xUnit。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/9a6d602756f94c1ea8682fc811d679ef.jpg)

6.JWalk

JWalk被设计为用于Java编程语言的单元测试工具包。它被设计为支持称为“Lazy系统单元测试”的测试范例。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/5d6b9c4de39c43df9eabc2fc5eecca5e_th.jpg)

JWalkTester工具对任何由程序员提供的编译的Java类执行任何测试。它能够通过静态和动态分析以及来自程序员的提示来测试懒惰Lazy规范的一致性。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/34c2bae3fea44fd9a2ec54c7447207d5.jpg)

7.Mockito

Mockito被设计为用于Java的开源测试框架，MIT许可证。Mockito允许程序员为了测试驱动开发（TDD）或行为驱动开发（BDD）而在自动化单元测试中创建和测试双对象（Mock对象）。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/cdef7a64440c499a865249fcbc91e38e.jpg)

8 Powermock

PowerMock是用于对源代码进行单元测试的Java框架，它可以作为其他模拟框架的扩展，比如原型Mockito或EasyMock，但具有更强大的功能。PowerMock利用自定义的类加载器和字节码操纵器来实现静态方法，构造函数，最终类和方法以及私有方法等的模拟。它主要是为了扩展现有的API，使用少量的方法和注解来实现额外的功能。


