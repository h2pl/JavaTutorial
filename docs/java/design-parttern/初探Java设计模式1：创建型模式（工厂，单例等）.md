# Table of Contents

  * [创建型模式](#创建型模式)
    * [简单工厂模式](#简单工厂模式)
    * [工厂模式](#工厂模式)
    * [抽象工厂模式](#抽象工厂模式)
    * [单例模式](#单例模式)
    * [建造者模式](#建造者模式)
    * [原型模式](#原型模式)
    * [创建型模式总结](#创建型模式总结)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


Java 设计模式

转自https://javadoop.com/post/design-pattern

系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

文章也将发表在我的个人博客，阅读体验更佳：

> www.how2playlife.com

一直想写一篇介绍设计模式的文章，让读者可以很快看完，而且一看就懂，看懂就会用，同时不会将各个模式搞混。自认为本文还是写得不错的，花了不少心思来写这文章和做图，力求让读者真的能看着简单同时有所收获。

设计模式是对大家实际工作中写的各种代码进行高层次抽象的总结，其中最出名的当属 _Gang of Four_ (_GoF_) 的分类了，他们将设计模式分类为 23 种经典的模式，根据用途我们又可以分为三大类，分别为创建型模式、结构型模式和行为型模式。是的，我不善于扯这些有的没的，还是少点废话吧~~~

有一些重要的设计原则在开篇和大家分享下，这些原则将贯通全文：

1.  面向接口编程，而不是面向实现。这个很重要，也是优雅的、可扩展的代码的第一步，这就不需要多说了吧。
2.  职责单一原则。每个类都应该只有一个单一的功能，并且该功能应该由这个类完全封装起来。
3.  对修改关闭，对扩展开放。对修改关闭是说，我们辛辛苦苦加班写出来的代码，该实现的功能和该修复的 bug 都完成了，别人可不能说改就改；对扩展开放就比较好理解了，也就是说在我们写好的代码基础上，很容易实现扩展。

**目录**

*   [创建型模式](https://javadoop.com/post/design-pattern#%E5%88%9B%E5%BB%BA%E5%9E%8B%E6%A8%A1%E5%BC%8F)
    *   [简单工厂模式](https://javadoop.com/post/design-pattern#%E7%AE%80%E5%8D%95%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F)
    *   [工厂模式](https://javadoop.com/post/design-pattern#%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F)
    *   [抽象工厂模式](https://javadoop.com/post/design-pattern#%E6%8A%BD%E8%B1%A1%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F)
    *   [单例模式](https://javadoop.com/post/design-pattern#%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F)
    *   [建造者模式](https://javadoop.com/post/design-pattern#%E5%BB%BA%E9%80%A0%E8%80%85%E6%A8%A1%E5%BC%8F)
    *   [原型模式](https://javadoop.com/post/design-pattern#%E5%8E%9F%E5%9E%8B%E6%A8%A1%E5%BC%8F)
    *   [创建型模式总结](https://javadoop.com/post/design-pattern#%E5%88%9B%E5%BB%BA%E5%9E%8B%E6%A8%A1%E5%BC%8F%E6%80%BB%E7%BB%93)

## 创建型模式

创建型模式的作用就是创建对象，说到创建一个对象，最熟悉的就是 new 一个对象，然后 set 相关属性。但是，在很多场景下，我们需要给客户端提供更加友好的创建对象的方式，尤其是那种我们定义了类，但是需要提供给其他开发者用的时候。

### 简单工厂模式

和名字一样简单，非常简单，直接上代码吧：

```
public class FoodFactory {

    public static Food makeFood(String name) {
        if (name.equals("noodle")) {
            Food noodle = new LanZhouNoodle();
            noodle.addSpicy("more");
            return noodle;
        } else if (name.equals("chicken")) {
            Food chicken = new HuangMenChicken();
            chicken.addCondiment("potato");
            return chicken;
        } else {
            return null;
        }
    }
}

```

_其中，LanZhouNoodle 和 HuangMenChicken 都继承自 Food。_

简单地说，简单工厂模式通常就是这样，一个工厂类 XxxFactory，里面有一个静态方法，根据我们不同的参数，返回不同的派生自同一个父类（或实现同一接口）的实例对象。

> 我们强调**职责单一**原则，一个类只提供一种功能，FoodFactory 的功能就是只要负责生产各种 Food。

### 工厂模式

简单工厂模式很简单，如果它能满足我们的需要，我觉得就不要折腾了。之所以需要引入工厂模式，是因为我们往往需要使用两个或两个以上的工厂。

```
public interface FoodFactory {
    Food makeFood(String name);
}
public class ChineseFoodFactory implements FoodFactory {

    @Override
    public Food makeFood(String name) {
        if (name.equals("A")) {
            return new ChineseFoodA();
        } else if (name.equals("B")) {
            return new ChineseFoodB();
        } else {
            return null;
        }
    }
}
public class AmericanFoodFactory implements FoodFactory {

    @Override
    public Food makeFood(String name) {
        if (name.equals("A")) {
            return new AmericanFoodA();
        } else if (name.equals("B")) {
            return new AmericanFoodB();
        } else {
            return null;
        }
    }
}

```

其中，ChineseFoodA、ChineseFoodB、AmericanFoodA、AmericanFoodB 都派生自 Food。

客户端调用：

```
public class APP {
    public static void main(String[] args) {
        // 先选择一个具体的工厂
        FoodFactory factory = new ChineseFoodFactory();
        // 由第一步的工厂产生具体的对象，不同的工厂造出不一样的对象
        Food food = factory.makeFood("A");
    }
}

```

虽然都是调用 makeFood("A") 制作 A 类食物，但是，不同的工厂生产出来的完全不一样。

第一步，我们需要选取合适的工厂，然后第二步基本上和简单工厂一样。

**核心在于，我们需要在第一步选好我们需要的工厂**。比如，我们有 LogFactory 接口，实现类有 FileLogFactory 和 KafkaLogFactory，分别对应将日志写入文件和写入 Kafka 中，显然，我们客户端第一步就需要决定到底要实例化 FileLogFactory 还是 KafkaLogFactory，这将决定之后的所有的操作。

虽然简单，不过我也把所有的构件都画到一张图上，这样读者看着比较清晰：

![](https://javadoop.com/blogimages/design-pattern/factory-1.png)![](https://csdnimg.cn/release/phoenix/write/assets/uploading.gif)转存失败重新上传取消![factory-1](https://javadoop.com/blogimages/design-pattern/factory-1.png)![](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw== "点击并拖拽以移动")​

### 抽象工厂模式

当涉及到**产品族**的时候，就需要引入抽象工厂模式了。

一个经典的例子是造一台电脑。我们先不引入抽象工厂模式，看看怎么实现。

因为电脑是由许多的构件组成的，我们将 CPU 和主板进行抽象，然后 CPU 由 CPUFactory 生产，主板由 MainBoardFactory 生产，然后，我们再将 CPU 和主板搭配起来组合在一起，如下图：

![](https://javadoop.com/blogimages/design-pattern/abstract-factory-1.png)![](https://csdnimg.cn/release/phoenix/write/assets/uploading.gif)转存失败重新上传取消![factory-1](https://javadoop.com/blogimages/design-pattern/abstract-factory-1.png)![](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw== "点击并拖拽以移动")​

这个时候的客户端调用是这样的：

```
// 得到 Intel 的 CPU
CPUFactory cpuFactory = new IntelCPUFactory();
CPU cpu = intelCPUFactory.makeCPU();

// 得到 AMD 的主板
MainBoardFactory mainBoardFactory = new AmdMainBoardFactory();
MainBoard mainBoard = mainBoardFactory.make();

// 组装 CPU 和主板
Computer computer = new Computer(cpu, mainBoard);

```

单独看 CPU 工厂和主板工厂，它们分别是前面我们说的**工厂模式**。这种方式也容易扩展，因为要给电脑加硬盘的话，只需要加一个 HardDiskFactory 和相应的实现即可，不需要修改现有的工厂。

但是，这种方式有一个问题，那就是如果 **Intel 家产的 CPU 和 AMD 产的主板不能兼容使用**，那么这代码就容易出错，因为客户端并不知道它们不兼容，也就会错误地出现随意组合。

下面就是我们要说的**产品族**的概念，它代表了组成某个产品的一系列附件的集合：

![](https://javadoop.com/blogimages/design-pattern/abstract-factory-2.png)

![abstract-factory-2](https://javadoop.com/blogimages/design-pattern/abstract-factory-2.png)


当涉及到这种产品族的问题的时候，就需要抽象工厂模式来支持了。我们不再定义 CPU 工厂、主板工厂、硬盘工厂、显示屏工厂等等，我们直接定义电脑工厂，每个电脑工厂负责生产所有的设备，这样能保证肯定不存在兼容问题。

![](https://javadoop.com/blogimages/design-pattern/abstract-factory-3.png)

![abstract-factory-3](https://javadoop.com/blogimages/design-pattern/abstract-factory-3.png)

这个时候，对于客户端来说，不再需要单独挑选 CPU厂商、主板厂商、硬盘厂商等，直接选择一家品牌工厂，品牌工厂会负责生产所有的东西，而且能保证肯定是兼容可用的。

```
public static void main(String[] args) {
    // 第一步就要选定一个“大厂”
    ComputerFactory cf = new AmdFactory();
    // 从这个大厂造 CPU
    CPU cpu = cf.makeCPU();
    // 从这个大厂造主板
    MainBoard board = cf.makeMainBoard();
      // 从这个大厂造硬盘
      HardDisk hardDisk = cf.makeHardDisk();

    // 将同一个厂子出来的 CPU、主板、硬盘组装在一起
    Computer result = new Computer(cpu, board, hardDisk);
}

```

当然，抽象工厂的问题也是显而易见的，比如我们要加个显示器，就需要修改所有的工厂，给所有的工厂都加上制造显示器的方法。这有点违反了**对修改关闭，对扩展开放**这个设计原则。

### 单例模式

单例模式用得最多，错得最多。

饿汉模式最简单：

```
public class Singleton {
    // 首先，将 new Singleton() 堵死
    private Singleton() {};
    // 创建私有静态实例，意味着这个类第一次使用的时候就会进行创建
    private static Singleton instance = new Singleton();

    public static Singleton getInstance() {
        return instance;
    }
    // 瞎写一个静态方法。这里想说的是，如果我们只是要调用 Singleton.getDate(...)，
    // 本来是不想要生成 Singleton 实例的，不过没办法，已经生成了
    public static Date getDate(String mode) {return new Date();}
}

```

> 很多人都能说出饿汉模式的缺点，可是我觉得生产过程中，很少碰到这种情况：你定义了一个单例的类，不需要其实例，可是你却把一个或几个你会用到的静态方法塞到这个类中。

饱汉模式最容易出错：

```
public class Singleton {
    // 首先，也是先堵死 new Singleton() 这条路
    private Singleton() {}
    // 和饿汉模式相比，这边不需要先实例化出来，注意这里的 volatile，它是必须的
    private static volatile Singleton instance = null;

    public static Singleton getInstance() {
        if (instance == null) {
            // 加锁
            synchronized (Singleton.class) {
                // 这一次判断也是必须的，不然会有并发问题
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

```

> 双重检查，指的是两次检查 instance 是否为 null。
> 
> volatile 在这里是需要的，希望能引起读者的关注。
> 
> 很多人不知道怎么写，直接就在 getInstance() 方法签名上加上 synchronized，这就不多说了，性能太差。

嵌套类最经典，以后大家就用它吧：

```
public class Singleton3 {

    private Singleton3() {}
    // 主要是使用了 嵌套类可以访问外部类的静态属性和静态方法 的特性
    private static class Holder {
        private static Singleton3 instance = new Singleton3();
    }
    public static Singleton3 getInstance() {
        return Holder.instance;
    }
}

```

> 注意，很多人都会把这个**嵌套类**说成是**静态内部类**，严格地说，内部类和嵌套类是不一样的，它们能访问的外部类权限也是不一样的。

最后，一定有人跳出来说用枚举实现单例，是的没错，枚举类很特殊，它在类加载的时候会初始化里面的所有的实例，而且 JVM 保证了它们不会再被实例化，所以它天生就是单例的。不说了，读者自己看着办吧，不建议使用。

### 建造者模式

经常碰见的 XxxBuilder 的类，通常都是建造者模式的产物。建造者模式其实有很多的变种，但是对于客户端来说，我们的使用通常都是一个模式的：

```
Food food = new FoodBuilder().a().b().c().build();
Food food = Food.builder().a().b().c().build();

```

套路就是先 new 一个 Builder，然后可以链式地调用一堆方法，最后再调用一次 build() 方法，我们需要的对象就有了。

来一个中规中矩的建造者模式：

```
class User {
    // 下面是“一堆”的属性
    private String name;
    private String password;
    private String nickName;
    private int age;

    // 构造方法私有化，不然客户端就会直接调用构造方法了
    private User(String name, String password, String nickName, int age) {
        this.name = name;
        this.password = password;
        this.nickName = nickName;
        this.age = age;
    }
    // 静态方法，用于生成一个 Builder，这个不一定要有，不过写这个方法是一个很好的习惯，
    // 有些代码要求别人写 new User.UserBuilder().a()...build() 看上去就没那么好
    public static UserBuilder builder() {
        return new UserBuilder();
    }

    public static class UserBuilder {
        // 下面是和 User 一模一样的一堆属性
        private String  name;
        private String password;
        private String nickName;
        private int age;

        private UserBuilder() {
        }

        // 链式调用设置各个属性值，返回 this，即 UserBuilder
        public UserBuilder name(String name) {
            this.name = name;
            return this;
        }

        public UserBuilder password(String password) {
            this.password = password;
            return this;
        }

        public UserBuilder nickName(String nickName) {
            this.nickName = nickName;
            return this;
        }

        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }

        // build() 方法负责将 UserBuilder 中设置好的属性“复制”到 User 中。
        // 当然，可以在 “复制” 之前做点检验
        public User build() {
            if (name == null || password == null) {
                throw new RuntimeException("用户名和密码必填");
            }
            if (age <= 0 || age >= 150) {
                throw new RuntimeException("年龄不合法");
            }
            // 还可以做赋予”默认值“的功能
              if (nickName == null) {
                nickName = name;
            }
            return new User(name, password, nickName, age);
        }
    }
}

```

核心是：先把所有的属性都设置给 Builder，然后 build() 方法的时候，将这些属性**复制**给实际产生的对象。

看看客户端的调用：

```
public class APP {
    public static void main(String[] args) {
        User d = User.builder()
                .name("foo")
                .password("pAss12345")
                .age(25)
                .build();
    }
}

```

说实话，建造者模式的**链式**写法很吸引人，但是，多写了很多“无用”的 builder 的代码，感觉这个模式没什么用。不过，当属性很多，而且有些必填，有些选填的时候，这个模式会使代码清晰很多。我们可以在 **Builder 的构造方法**中强制让调用者提供必填字段，还有，在 build() 方法中校验各个参数比在 User 的构造方法中校验，代码要优雅一些。

> 题外话，强烈建议读者使用 lombok，用了 lombok 以后，上面的一大堆代码会变成如下这样:

```
@Builder
class User {
    private String  name;
    private String password;
    private String nickName;
    private int age;
}

```

> 怎么样，省下来的时间是不是又可以干点别的了。

当然，如果你只是想要链式写法，不想要建造者模式，有个很简单的办法，User 的 getter 方法不变，所有的 setter 方法都让其 **return this** 就可以了，然后就可以像下面这样调用：

```
User user = new User().setName("").setPassword("").setAge(20);

```

### 原型模式

这是我要说的创建型模式的最后一个设计模式了。

原型模式很简单：有一个原型**实例**，基于这个原型实例产生新的实例，也就是“克隆”了。

Object 类中有一个 clone() 方法，它用于生成一个新的对象，当然，如果我们要调用这个方法，java 要求我们的类必须先**实现 Cloneable 接口**，此接口没有定义任何方法，但是不这么做的话，在 clone() 的时候，会抛出 CloneNotSupportedException 异常。

```
protected native Object clone() throws CloneNotSupportedException;

```

> java 的克隆是浅克隆，碰到对象引用的时候，克隆出来的对象和原对象中的引用将指向同一个对象。通常实现深克隆的方法是将对象进行序列化，然后再进行反序列化。

原型模式了解到这里我觉得就够了，各种变着法子说这种代码或那种代码是原型模式，没什么意义。

### 创建型模式总结

创建型模式总体上比较简单，它们的作用就是为了产生实例对象，算是各种工作的第一步了，因为我们写的是**面向对象**的代码，所以我们第一步当然是需要创建一个对象了。

简单工厂模式最简单；工厂模式在简单工厂模式的基础上增加了选择工厂的维度，需要第一步选择合适的工厂；抽象工厂模式有产品族的概念，如果各个产品是存在兼容性问题的，就要用抽象工厂模式。单例模式就不说了，为了保证全局使用的是同一对象，一方面是安全性考虑，一方面是为了节省资源；建造者模式专门对付属性很多的那种类，为了让代码更优美；原型模式用得最少，了解和 Object 类中的 clone() 方法相关的知识即可。


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
