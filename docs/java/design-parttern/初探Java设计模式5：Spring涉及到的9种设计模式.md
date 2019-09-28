# Table of Contents

  * [结构型模式](#结构型模式)
    * [代理模式](#代理模式)
    * [适配器模式](#适配器模式)
    * [桥梁模式](#桥梁模式)
    * [装饰模式](#装饰模式)
    * [门面模式](#门面模式)
    * [组合模式](#组合模式)
    * [享元模式](#享元模式)
    * [结构型模式总结](#结构型模式总结)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)




本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

文章也将发表在我的个人博客，阅读体验更佳：

> www.how2playlife.com

**设计模式**作为工作学习中的枕边书，却时常处于勤说不用的尴尬境地，也不是我们时常忘记，只是一直没有记忆。

今天，螃蟹在IT学习者网站就设计模式的内在价值做一番探讨，并以spring为例进行讲解，只有领略了其设计的思想理念，才能在工作学习中运用到“无形”。

Spring作为业界的经典框架，无论是在架构设计方面，还是在代码编写方面，都堪称行内典范。好了，话不多说，开始今天的内容。

spring中常用的设计模式达到九种，我们举例说明：

**第一种：简单工厂**

又叫做静态工厂方法（StaticFactory Method）模式，但不属于23种GOF设计模式之一。 
简单工厂模式的实质是由一个工厂类根据传入的参数，动态决定应该创建哪一个产品类。 
spring中的BeanFactory就是简单工厂模式的体现，根据传入一个唯一的标识来获得bean对象，但是否是在传入参数后创建还是传入参数前创建这个要根据具体情况来定。如下配置，就是在 HelloItxxz 类中创建一个 itxxzBean。

    <beans>
    
        <bean id="singletonBean" >
    
            <constructor-arg>
    
                <value>Hello! 这是singletonBean!value>
    
            </constructor-arg>
    
       </ bean>
    
        <bean id="itxxzBean"
    
            singleton="false">
    
            <constructor-arg>
    
                <value>Hello! 这是itxxzBean! value>
    
            </constructor-arg>
    
        </bean>
    
    </beans>

**第二种：工厂方法（Factory Method）**

通常由应用程序直接使用new创建新的对象，为了将对象的创建和使用相分离，采用工厂模式,即应用程序将对象的创建及初始化职责交给工厂对象。

一般情况下,应用程序有自己的工厂对象来创建bean.如果将应用程序自己的工厂对象交给Spring管理,那么Spring管理的就不是普通的bean,而是工厂Bean。

螃蟹就以工厂方法中的静态方法为例讲解一下：
    
    import java.util.Random;
    
    public class StaticFactoryBean {
    
          public static Integer createRandom() {
    
               return new Integer(new Random().nextInt());
    
           }
    
    }

建一个config.xm配置文件，将其纳入Spring容器来管理,需要通过factory-method指定静态方法名称

    <bean id="random"
    
    factory-method="createRandom" //createRandom方法必须是static的,才能找到 scope="prototype"
    
    />

测试:

    public static void main(String[] args) {
          //调用getBean()时,返回随机数.如果没有指定factory-method,会返回StaticFactoryBean的实例,即返回工厂Bean的实例       XmlBeanFactory factory = new XmlBeanFactory(new ClassPathResource("config.xml"));       System.out.println("我是IT学习者创建的实例:"+factory.getBean("random").toString());
    
    }

**第三种：单例模式（Singleton）**

保证一个类仅有一个实例，并提供一个访问它的全局访问点。 
spring中的单例模式完成了后半句话，即提供了全局的访问点BeanFactory。但没有从构造器级别去控制单例，这是因为spring管理的是是任意的java对象。 
核心提示点：Spring下默认的bean均为singleton，可以通过singleton=“true|false” 或者 scope=“？”来指定

**第四种：适配器（Adapter）**

在Spring的Aop中，使用的Advice（通知）来增强被代理类的功能。Spring实现这一AOP功能的原理就使用代理模式（1、JDK动态代理。2、CGLib字节码生成技术代理。）对类进行方法级别的切面增强，即，生成被代理类的代理类， 并在代理类的方法前，设置拦截器，通过执行拦截器重的内容增强了代理方法的功能，实现的面向切面编程。

**Adapter类接口**：

    public interface AdvisorAdapter {
    
    boolean supportsAdvice(Advice advice);
    
          MethodInterceptor getInterceptor(Advisor advisor);
    
    } **MethodBeforeAdviceAdapter类**，Adapter
    
    class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {
    
          public boolean supportsAdvice(Advice advice) {
    
                return (advice instanceof MethodBeforeAdvice);
    
          }
    
          public MethodInterceptor getInterceptor(Advisor advisor) {
    
                MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
    
          return new MethodBeforeAdviceInterceptor(advice);
    
          }
    
    }

**第五种：包装器（Decorator）**

在我们的项目中遇到这样一个问题：我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。我们以往在spring和hibernate框架中总是配置一个数据源，因而sessionFactory的dataSource属性总是指向这个数据源并且恒定不变，所有DAO在使用sessionFactory的时候都是通过这个数据源访问数据库。

但是现在，由于项目的需要，我们的DAO在访问sessionFactory的时候都不得不在多个数据源中不断切换，问题就出现了：如何让sessionFactory在执行数据持久化的时候，根据客户的需求能够动态切换不同的数据源？我们能不能在spring的框架下通过少量修改得到解决？是否有什么设计模式可以利用呢？ 


首先想到在spring的applicationContext中配置所有的dataSource。这些dataSource可能是各种不同类型的，比如不同的数据库：Oracle、SQL Server、MySQL等，也可能是不同的数据源：比如apache 提供的org.apache.commons.dbcp.BasicDataSource、spring提供的org.springframework.jndi.JndiObjectFactoryBean等。然后sessionFactory根据客户的每次请求，将dataSource属性设置成不同的数据源，以到达切换数据源的目的。

spring中用到的包装器模式在类名上有两种表现：一种是类名中含有Wrapper，另一种是类名中含有Decorator。基本上都是动态地给一个对象添加一些额外的职责。 

**第六种：代理（Proxy）**

为其他对象提供一种代理以控制对这个对象的访问。  从结构上来看和Decorator模式类似，但Proxy是控制，更像是一种对功能的限制，而Decorator是增加职责。 
spring的Proxy模式在aop中有体现，比如JdkDynamicAopProxy和Cglib2AopProxy。 

**第七种：观察者（Observer）**

定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。
spring中Observer模式常用的地方是listener的实现。如ApplicationListener。 

**第八种：策略（Strategy）**

定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。 
spring中在实例化对象的时候用到Strategy模式
在SimpleInstantiationStrategy中有如下代码说明了策略模式的使用情况： 

**第九种：模板方法（Template Method）**

定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。Template Method使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。
Template Method模式一般是需要继承的。这里想要探讨另一种对Template Method的理解。

spring中的JdbcTemplate，在用这个类时并不想去继承这个类，因为这个类的方法太多，但是我们还是想用到JdbcTemplate已有的稳定的、公用的数据库连接，那么我们怎么办呢？我们可以把变化的东西抽出来作为一个参数传入JdbcTemplate的方法中。但是变化的东西是一段代码，而且这段代码会用到JdbcTemplate中的变量。

怎么办？那我们就用回调对象吧。在这个回调对象中定义一个操纵JdbcTemplate中变量的方法，我们去实现这个方法，就把变化的东西集中到这里了。然后我们再传入这个回调对象到JdbcTemplate，从而完成了调用。这可能是Template Method不需要继承的另一种实现方式吧。 


本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

文章也将发表在我的个人博客，阅读体验更佳：

> www.how2playlife.com

## 结构型模式

前面创建型模式介绍了创建对象的一些设计模式，这节介绍的结构型模式旨在通过改变代码结构来达到解耦的目的，使得我们的代码容易维护和扩展。

### 代理模式

第一个要介绍的代理模式是最常使用的模式之一了，用一个代理来隐藏具体实现类的实现细节，通常还用于在真实的实现的前后添加一部分逻辑。

既然说是**代理**，那就要对客户端隐藏真实实现，由代理来负责客户端的所有请求。当然，代理只是个代理，它不会完成实际的业务逻辑，而是一层皮而已，但是对于客户端来说，它必须表现得就是客户端需要的真实实现。

> 理解**代理**这个词，这个模式其实就简单了。

```
public interface FoodService {
    Food makeChicken();
    Food makeNoodle();
}

public class FoodServiceImpl implements FoodService {
    public Food makeChicken() {
          Food f = new Chicken()
        f.setChicken("1kg");
          f.setSpicy("1g");
          f.setSalt("3g");
        return f;
    }
    public Food makeNoodle() {
        Food f = new Noodle();
        f.setNoodle("500g");
        f.setSalt("5g");
        return f;
    }
}

// 代理要表现得“就像是”真实实现类，所以需要实现 FoodService
public class FoodServiceProxy implements FoodService {

    // 内部一定要有一个真实的实现类，当然也可以通过构造方法注入
    private FoodService foodService = new FoodServiceImpl();

    public Food makeChicken() {
        System.out.println("我们马上要开始制作鸡肉了");

        // 如果我们定义这句为核心代码的话，那么，核心代码是真实实现类做的，
        // 代理只是在核心代码前后做些“无足轻重”的事情
        Food food = foodService.makeChicken();

        System.out.println("鸡肉制作完成啦，加点胡椒粉"); // 增强
          food.addCondiment("pepper");

        return food;
    }
    public Food makeNoodle() {
        System.out.println("准备制作拉面~");
        Food food = foodService.makeNoodle();
        System.out.println("制作完成啦")
        return food;
    }
}

```

客户端调用，注意，我们要用代理来实例化接口：

```
// 这里用代理类来实例化
FoodService foodService = new FoodServiceProxy();
foodService.makeChicken();

```

![](https://javadoop.com/blogimages/design-pattern/proxy-1.png)
我们发现没有，代理模式说白了就是做 **“方法包装”** 或做 **“方法增强”**。在面向切面编程中，算了还是不要吹捧这个名词了，在 AOP 中，其实就是动态代理的过程。比如 Spring 中，我们自己不定义代理类，但是 Spring 会帮我们动态来定义代理，然后把我们定义在 @Before、@After、@Around 中的代码逻辑动态添加到代理中。

说到动态代理，又可以展开说 …… Spring 中实现动态代理有两种，一种是如果我们的类定义了接口，如 UserService 接口和 UserServiceImpl 实现，那么采用 JDK 的动态代理，感兴趣的读者可以去看看 java.lang.reflect.Proxy 类的源码；另一种是我们自己没有定义接口的，Spring 会采用 CGLIB 进行动态代理，它是一个 jar 包，性能还不错。

### 适配器模式

说完代理模式，说适配器模式，是因为它们很相似，这里可以做个比较。

适配器模式做的就是，有一个接口需要实现，但是我们现成的对象都不满足，需要加一层适配器来进行适配。

适配器模式总体来说分三种：默认适配器模式、对象适配器模式、类适配器模式。先不急着分清楚这几个，先看看例子再说。

默认适配器模式

首先，我们先看看最简单的适配器模式**默认适配器模式(Default Adapter)**是怎么样的。

我们用 Appache commons-io 包中的 FileAlterationListener 做例子，此接口定义了很多的方法，用于对文件或文件夹进行监控，一旦发生了对应的操作，就会触发相应的方法。

```
public interface FileAlterationListener {
    void onStart(final FileAlterationObserver observer);
    void onDirectoryCreate(final File directory);
    void onDirectoryChange(final File directory);
    void onDirectoryDelete(final File directory);
    void onFileCreate(final File file);
    void onFileChange(final File file);
    void onFileDelete(final File file);
    void onStop(final FileAlterationObserver observer);
}

```

此接口的一大问题是抽象方法太多了，如果我们要用这个接口，意味着我们要实现每一个抽象方法，如果我们只是想要监控文件夹中的**文件创建**和**文件删除**事件，可是我们还是不得不实现所有的方法，很明显，这不是我们想要的。

所以，我们需要下面的一个**适配器**，它用于实现上面的接口，但是**所有的方法都是空方法**，这样，我们就可以转而定义自己的类来继承下面这个类即可。

```
public class FileAlterationListenerAdaptor implements FileAlterationListener {

    public void onStart(final FileAlterationObserver observer) {
    }

    public void onDirectoryCreate(final File directory) {
    }

    public void onDirectoryChange(final File directory) {
    }

    public void onDirectoryDelete(final File directory) {
    }

    public void onFileCreate(final File file) {
    }

    public void onFileChange(final File file) {
    }

    public void onFileDelete(final File file) {
    }

    public void onStop(final FileAlterationObserver observer) {
    }
}

```

比如我们可以定义以下类，我们仅仅需要实现我们想实现的方法就可以了：

```
public class FileMonitor extends FileAlterationListenerAdaptor {
    public void onFileCreate(final File file) {
        // 文件创建
        doSomething();
    }

    public void onFileDelete(final File file) {
        // 文件删除
        doSomething();
    }
}

```

当然，上面说的只是适配器模式的其中一种，也是最简单的一种，无需多言。下面，再介绍**“正统的”**适配器模式。

对象适配器模式

来看一个《Head First 设计模式》中的一个例子，我稍微修改了一下，看看怎么将鸡适配成鸭，这样鸡也能当鸭来用。因为，现在鸭这个接口，我们没有合适的实现类可以用，所以需要适配器。

```
public interface Duck {
    public void quack(); // 鸭的呱呱叫
      public void fly(); // 飞
}

public interface Cock {
    public void gobble(); // 鸡的咕咕叫
      public void fly(); // 飞
}

public class WildCock implements Cock {
    public void gobble() {
        System.out.println("咕咕叫");
    }
      public void fly() {
        System.out.println("鸡也会飞哦");
    }
}

```

鸭接口有 fly() 和 quare() 两个方法，鸡 Cock 如果要冒充鸭，fly() 方法是现成的，但是鸡不会鸭的呱呱叫，没有 quack() 方法。这个时候就需要适配了：

```
// 毫无疑问，首先，这个适配器肯定需要 implements Duck，这样才能当做鸭来用
public class CockAdapter implements Duck {

    Cock cock;
    // 构造方法中需要一个鸡的实例，此类就是将这只鸡适配成鸭来用
      public CockAdapter(Cock cock) {
        this.cock = cock;
    }

    // 实现鸭的呱呱叫方法
      @Override
      public void quack() {
        // 内部其实是一只鸡的咕咕叫
        cock.gobble();
    }

      @Override
      public void fly() {
        cock.fly();
    }
}

```

客户端调用很简单了：

```
public static void main(String[] args) {
    // 有一只野鸡
      Cock wildCock = new WildCock();
      // 成功将野鸡适配成鸭
      Duck duck = new CockAdapter(wildCock);
      ...
}

```

到这里，大家也就知道了适配器模式是怎么回事了。无非是我们需要一只鸭，但是我们只有一只鸡，这个时候就需要定义一个适配器，由这个适配器来充当鸭，但是适配器里面的方法还是由鸡来实现的。

我们用一个图来简单说明下：

![](https://javadoop.com/blogimages/design-pattern/adapter-1.png)

上图应该还是很容易理解的，我就不做更多的解释了。下面，我们看看类适配模式怎么样的。

类适配器模式

废话少说，直接上图：

![](https://javadoop.com/blogimages/design-pattern/adapter-2.png)

看到这个图，大家应该很容易理解的吧，通过继承的方法，适配器自动获得了所需要的大部分方法。这个时候，客户端使用更加简单，直接 `Target t = new SomeAdapter();` 就可以了。

适配器模式总结

1.  类适配和对象适配的异同

    > 一个采用继承，一个采用组合；
    > 
    > 类适配属于静态实现，对象适配属于组合的动态实现，对象适配需要多实例化一个对象。
    > 
    > 总体来说，对象适配用得比较多。

2.  适配器模式和代理模式的异同

    比较这两种模式，其实是比较对象适配器模式和代理模式，在代码结构上，它们很相似，都需要一个具体的实现类的实例。但是它们的目的不一样，代理模式做的是增强原方法的活；适配器做的是适配的活，为的是提供“把鸡包装成鸭，然后当做鸭来使用”，而鸡和鸭它们之间原本没有继承关系。

![](https://javadoop.com/blogimages/design-pattern/adapter-5.png)

### 桥梁模式

理解桥梁模式，其实就是理解代码抽象和解耦。

我们首先需要一个桥梁，它是一个接口，定义提供的接口方法。

```
public interface DrawAPI {
   public void draw(int radius, int x, int y);
}

```

然后是一系列实现类：

```
public class RedPen implements DrawAPI {
   @Override
   public void draw(int radius, int x, int y) {
      System.out.println("用红色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
   }
}
public class GreenPen implements DrawAPI {
   @Override
   public void draw(int radius, int x, int y) {
      System.out.println("用绿色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
   }
}
public class BluePen implements DrawAPI {
   @Override
   public void draw(int radius, int x, int y) {
      System.out.println("用蓝色笔画图，radius:" + radius + ", x:" + x + ", y:" + y);
   }
}

```

定义一个抽象类，此类的实现类都需要使用 DrawAPI：

```
public abstract class Shape {
   protected DrawAPI drawAPI;

   protected Shape(DrawAPI drawAPI){
      this.drawAPI = drawAPI;
   }
   public abstract void draw();    
}

```

定义抽象类的子类：

```
// 圆形
public class Circle extends Shape {
   private int radius;

   public Circle(int radius, DrawAPI drawAPI) {
      super(drawAPI);
      this.radius = radius;
   }

   public void draw() {
      drawAPI.draw(radius, 0, 0);
   }
}
// 长方形
public class Rectangle extends Shape {
    private int x;
      private int y;

      public Rectangle(int x, int y, DrawAPI drawAPI) {
        super(drawAPI);
          this.x = x;
          this.y = y;
    }
      public void draw() {
      drawAPI.draw(0, x, y);
   }
}

```

最后，我们来看客户端演示：

```
public static void main(String[] args) {
    Shape greenCircle = new Circle(10, new GreenPen());
      Shape redRectangle = new Rectangle(4, 8, new RedPen());

      greenCircle.draw();
      redRectangle.draw();
}

```

可能大家看上面一步步还不是特别清晰，我把所有的东西整合到一张图上：

![](https://javadoop.com/blogimages/design-pattern/bridge-1.png)

这回大家应该就知道抽象在哪里，怎么解耦了吧。桥梁模式的优点也是显而易见的，就是非常容易进行扩展。

> 本节引用了[这里](https://www.tutorialspoint.com/design_pattern/bridge_pattern.htm)的例子，并对其进行了修改。

### 装饰模式

要把装饰模式说清楚明白，不是件容易的事情。也许读者知道 Java IO 中的几个类是典型的装饰模式的应用，但是读者不一定清楚其中的关系，也许看完就忘了，希望看完这节后，读者可以对其有更深的感悟。

首先，我们先看一个简单的图，看这个图的时候，了解下层次结构就可以了：

![](https://javadoop.com/blogimages/design-pattern/decorator-1.png)

我们来说说装饰模式的出发点，从图中可以看到，接口 `Component` 其实已经有了 `ConcreteComponentA` 和 `ConcreteComponentB` 两个实现类了，但是，如果我们要**增强**这两个实现类的话，我们就可以采用装饰模式，用具体的装饰器来**装饰**实现类，以达到增强的目的。

> 从名字来简单解释下装饰器。既然说是装饰，那么往往就是**添加小功能**这种，而且，我们要满足可以添加多个小功能。最简单的，代理模式就可以实现功能的增强，但是代理不容易实现多个功能的增强，当然你可以说用代理包装代理的方式，但是那样的话代码就复杂了。

首先明白一些简单的概念，从图中我们看到，所有的具体装饰者们 ConcreteDecorator_ 都可以作为 Component 来使用，因为它们都实现了 Component 中的所有接口。它们和 Component 实现类 ConcreteComponent_ 的区别是，它们只是装饰者，起**装饰**作用，也就是即使它们看上去牛逼轰轰，但是它们都只是在具体的实现中**加了层皮来装饰**而已。

> 注意这段话中混杂在各个名词中的 Component 和 Decorator，别搞混了。

下面来看看一个例子，先把装饰模式弄清楚，然后再介绍下 java io 中的装饰模式的应用。

最近大街上流行起来了“快乐柠檬”，我们把快乐柠檬的饮料分为三类：红茶、绿茶、咖啡，在这三大类的基础上，又增加了许多的口味，什么金桔柠檬红茶、金桔柠檬珍珠绿茶、芒果红茶、芒果绿茶、芒果珍珠红茶、烤珍珠红茶、烤珍珠芒果绿茶、椰香胚芽咖啡、焦糖可可咖啡等等，每家店都有很长的菜单，但是仔细看下，其实原料也没几样，但是可以搭配出很多组合，如果顾客需要，很多没出现在菜单中的饮料他们也是可以做的。

在这个例子中，红茶、绿茶、咖啡是最基础的饮料，其他的像金桔柠檬、芒果、珍珠、椰果、焦糖等都属于装饰用的。当然，在开发中，我们确实可以像门店一样，开发这些类：LemonBlackTea、LemonGreenTea、MangoBlackTea、MangoLemonGreenTea......但是，很快我们就发现，这样子干肯定是不行的，这会导致我们需要组合出所有的可能，而且如果客人需要在红茶中加双份柠檬怎么办？三份柠檬怎么办？万一有个变态要四份柠檬，所以这种做法是给自己找加班的。

不说废话了，上代码。

首先，定义饮料抽象基类：

```
public abstract class Beverage {
      // 返回描述
      public abstract String getDescription();
      // 返回价格
      public abstract double cost();
}

```

然后是三个基础饮料实现类，红茶、绿茶和咖啡：

```
public class BlackTea extends Beverage {
      public String getDescription() {
        return "红茶";
    }
      public double cost() {
        return 10;
    }
}
public class GreenTea extends Beverage {
    public String getDescription() {
        return "绿茶";
    }
      public double cost() {
        return 11;
    }
}
...// 咖啡省略

```

定义调料，也就是装饰者的基类，此类必须继承自 Beverage：

```
// 调料
public abstract class Condiment extends Beverage {

}

```

然后我们来定义柠檬、芒果等具体的调料，它们属于装饰者，毫无疑问，这些调料肯定都需要继承 Condiment 类：

```
public class Lemon extends Condiment {
    private Beverage bevarage;
      // 这里很关键，需要传入具体的饮料，如需要传入没有被装饰的红茶或绿茶，
      // 当然也可以传入已经装饰好的芒果绿茶，这样可以做芒果柠檬绿茶
      public Lemon(Beverage bevarage) {
        this.bevarage = bevarage;
    }
      public String getDescription() {
        // 装饰
        return bevarage.getDescription() + ", 加柠檬";
    }
      public double cost() {
          // 装饰
        return beverage.cost() + 2; // 加柠檬需要 2 元
    }
}
public class Mango extends Condiment {
    private Beverage bevarage;
      public Mango(Beverage bevarage) {
        this.bevarage = bevarage;
    }
      public String getDescription() {
        return bevarage.getDescription() + ", 加芒果";
    }
      public double cost() {
        return beverage.cost() + 3; // 加芒果需要 3 元
    }
}
...// 给每一种调料都加一个类

```

看客户端调用：

```
public static void main(String[] args) {
      // 首先，我们需要一个基础饮料，红茶、绿茶或咖啡
    Beverage beverage = new GreenTea();
      // 开始装饰
      beverage = new Lemon(beverage); // 先加一份柠檬
      beverage = new Mongo(beverage); // 再加一份芒果

      System.out.println(beverage.getDescription() + " 价格：￥" + beverage.cost());
      //"绿茶, 加柠檬, 加芒果 价格：￥16"
}

```

如果我们需要芒果珍珠双份柠檬红茶：

```
Beverage beverage = new Mongo(new Pearl(new Lemon(new Lemon(new BlackTea()))));

```

是不是很变态？

看看下图可能会清晰一些：

![](https://javadoop.com/blogimages/design-pattern/decorator-2.png)

到这里，大家应该已经清楚装饰模式了吧。

下面，我们再来说说 java IO 中的装饰模式。看下图 InputStream 派生出来的部分类：

![](https://javadoop.com/blogimages/design-pattern/decorator-3.png)

我们知道 InputStream 代表了输入流，具体的输入来源可以是文件（FileInputStream）、管道（PipedInputStream）、数组（ByteArrayInputStream）等，这些就像前面奶茶的例子中的红茶、绿茶，属于基础输入流。

FilterInputStream 承接了装饰模式的关键节点，其实现类是一系列装饰器，比如 BufferedInputStream 代表用缓冲来装饰，也就使得输入流具有了缓冲的功能，LineNumberInputStream 代表用行号来装饰，在操作的时候就可以取得行号了，DataInputStream 的装饰，使得我们可以从输入流转换为 java 中的基本类型值。

当然，在 java IO 中，如果我们使用装饰器的话，就不太适合面向接口编程了，如：

```
InputStream inputStream = new LineNumberInputStream(new BufferedInputStream(new FileInputStream("")));

```

这样的结果是，InputStream 还是不具有读取行号的功能，因为读取行号的方法定义在 LineNumberInputStream 类中。

我们应该像下面这样使用：

```
DataInputStream is = new DataInputStream(
                              new BufferedInputStream(
                                  new FileInputStream("")));

```

> 所以说嘛，要找到纯的严格符合设计模式的代码还是比较难的。

### 门面模式

门面模式（也叫外观模式，Facade Pattern）在许多源码中有使用，比如 slf4j 就可以理解为是门面模式的应用。这是一个简单的设计模式，我们直接上代码再说吧。

首先，我们定义一个接口：

```
public interface Shape {
   void draw();
}

```

定义几个实现类：

```
public class Circle implements Shape {

   @Override
   public void draw() {
      System.out.println("Circle::draw()");
   }
}

public class Rectangle implements Shape {

   @Override
   public void draw() {
      System.out.println("Rectangle::draw()");
   }
}

```

客户端调用：

```
public static void main(String[] args) {
    // 画一个圆形
      Shape circle = new Circle();
      circle.draw();

      // 画一个长方形
      Shape rectangle = new Rectangle();
      rectangle.draw();
}

```

以上是我们常写的代码，我们需要画圆就要先实例化圆，画长方形就需要先实例化一个长方形，然后再调用相应的 draw() 方法。

下面，我们看看怎么用门面模式来让客户端调用更加友好一些。

我们先定义一个门面：

```
public class ShapeMaker {
   private Shape circle;
   private Shape rectangle;
   private Shape square;

   public ShapeMaker() {
      circle = new Circle();
      rectangle = new Rectangle();
      square = new Square();
   }

  /**
   * 下面定义一堆方法，具体应该调用什么方法，由这个门面来决定
   */

   public void drawCircle(){
      circle.draw();
   }
   public void drawRectangle(){
      rectangle.draw();
   }
   public void drawSquare(){
      square.draw();
   }
}

```

看看现在客户端怎么调用：

```
public static void main(String[] args) {
  ShapeMaker shapeMaker = new ShapeMaker();

  // 客户端调用现在更加清晰了
  shapeMaker.drawCircle();
  shapeMaker.drawRectangle();
  shapeMaker.drawSquare();        
}

```

门面模式的优点显而易见，客户端不再需要关注实例化时应该使用哪个实现类，直接调用门面提供的方法就可以了，因为门面类提供的方法的方法名对于客户端来说已经很友好了。

### 组合模式

组合模式用于表示具有层次结构的数据，使得我们对单个对象和组合对象的访问具有一致性。

直接看一个例子吧，每个员工都有姓名、部门、薪水这些属性，同时还有下属员工集合（虽然可能集合为空），而下属员工和自己的结构是一样的，也有姓名、部门这些属性，同时也有他们的下属员工集合。

```
public class Employee {
   private String name;
   private String dept;
   private int salary;
   private List<Employee> subordinates; // 下属

   public Employee(String name,String dept, int sal) {
      this.name = name;
      this.dept = dept;
      this.salary = sal;
      subordinates = new ArrayList<Employee>();
   }

   public void add(Employee e) {
      subordinates.add(e);
   }

   public void remove(Employee e) {
      subordinates.remove(e);
   }

   public List<Employee> getSubordinates(){
     return subordinates;
   }

   public String toString(){
      return ("Employee :[ Name : " + name + ", dept : " + dept + ", salary :" + salary+" ]");
   }   
}

```

通常，这种类需要定义 add(node)、remove(node)、getChildren() 这些方法。

这说的其实就是组合模式，这种简单的模式我就不做过多介绍了，相信各位读者也不喜欢看我写废话。

### 享元模式

英文是 Flyweight Pattern，不知道是谁最先翻译的这个词，感觉这翻译真的不好理解，我们试着强行关联起来吧。Flyweight 是轻量级的意思，享元分开来说就是 共享 元器件，也就是复用已经生成的对象，这种做法当然也就是轻量级的了。

复用对象最简单的方式是，用一个 HashMap 来存放每次新生成的对象。每次需要一个对象的时候，先到 HashMap 中看看有没有，如果没有，再生成新的对象，然后将这个对象放入 HashMap 中。

这种简单的代码我就不演示了。

### 结构型模式总结

前面，我们说了代理模式、适配器模式、桥梁模式、装饰模式、门面模式、组合模式和享元模式。读者是否可以分别把这几个模式说清楚了呢？在说到这些模式的时候，心中是否有一个清晰的图或处理流程在脑海里呢？

代理模式是做方法增强的，适配器模式是把鸡包装成鸭这种用来适配接口的，桥梁模式做到了很好的解耦，装饰模式从名字上就看得出来，适合于装饰类或者说是增强类的场景，门面模式的优点是客户端不需要关心实例化过程，只要调用需要的方法即可，组合模式用于描述具有层次结构的数据，享元模式是为了在特定的场景中缓存已经创建的对象，用于提高性能。

## 参考文章

转自https://javadoop.com/post/design-pattern


## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
