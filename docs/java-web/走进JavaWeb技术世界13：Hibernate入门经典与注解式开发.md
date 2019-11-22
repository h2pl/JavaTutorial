# Table of Contents

  * [前言](#前言)
  * [ORM概述](#orm概述)
  * [测试](#测试)
  * [相关类](#相关类)
  * [扩展](#扩展)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [个人公众号：程序员黄小斜](#个人公众号：程序员黄小斜)
    * [技术公众号：Java技术江湖](#技术公众号：java技术江湖)


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

## 前言

本博文主要讲解介绍Hibernate框架，ORM的概念和Hibernate入门，相信你们看了就会使用Hibernate了!

什么是Hibernate框架？
Hibernate是一种ORM框架，全称为 Object_Relative DateBase-Mapping，在Java对象与关系数据库之间建立某种映射，以实现直接存取Java对象！

为什么要使用Hibernate？
既然Hibernate是关于Java对象和关系数据库之间的联系的话，也就是我们MVC中的数据持久层->在编写程序中的DAO层...

首先，我们来回顾一下我们在DAO层写程序的历程吧：

在DAO层操作XML，将数据封装到XML文件上，读写XML文件数据实现CRUD
在DAO层使用原生JDBC连接数据库，实现CRUD
嫌弃JDBC的ConnectionStatementResultSet等对象太繁琐，使用对原生JDBC的封装组件-->DbUtils组件
我们来看看使用DbUtils之后，程序的代码是怎么样的：


    public class CategoryDAOImpl implements zhongfucheng.dao.CategoryDao {
    
        @Override
        public void addCategory(Category category) {
    
            QueryRunner queryRunner = new QueryRunner(Utils2DB.getDataSource());
    
            String sql = "INSERT INTO category (id, name, description) VALUES(?,?,?)";
            try {
                queryRunner.update(sql, new Object[]{category.getId(), category.getName(), category.getDescription()});
    
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
        }
    
        @Override
        public Category findCategory(String id) {
            QueryRunner queryRunner = new QueryRunner(Utils2DB.getDataSource());
            String sql = "SELECT * FROM category WHERE id=?";
    
            try {
                Category category = (Category) queryRunner.query(sql, id, new BeanHandler(Category.class));
    
                return category;
    
            } catch (SQLException e) {
                throw new RuntimeException(e);
            }
    
        }
    
        @Override
        public List<Category> getAllCategory() {
            QueryRunner queryRunner = new QueryRunner(Utils2DB.getDataSource());
            String sql = "SELECT * FROM category";
    
            try {
                List<Category> categories = (List<Category>) queryRunner.query(sql, new BeanListHandler(Category.class));
    
                return categories;
            } catch (SQLException e) {
                throw  new RuntimeException(e);
            }
    
        }
    }
    
其实使用DbUtils时，DAO层中的代码编写是很有规律的。

当插入数据的时候，就将JavaBean对象拆分，拼装成SQL语句
当查询数据的时候，用SQL把数据库表中的列组合，拼装成JavaBean对象
也就是说：javaBean对象和数据表中的列存在映射关系!如果程序能够自动生成SQL语句就好了....那么Hibernate就实现了这个功能！

简单来说：我们使用Hibernate框架就不用我们写很多繁琐的SQL语句，从而简化我们的开发！


## ORM概述
在介绍Hibernate的时候，说了Hibernate是一种ORM的框架。那什么是ORM呢？ORM是一种思想

O代表的是Objcet
R代表的是Relative
M代表的是Mapping
ORM->对象关系映射....ORM关注是对象与数据库中的列的关系



Hibernate快速入门
学习一个框架无非就是三个步骤：

引入jar开发包
配置相关的XML文件
熟悉API
引入相关jar包
我们使用的是Hibernate3.6的版本

hibernate3.jar核心 + required 必须引入的(6个) + jpa 目录 + 数据库驱动包


编写对象和对象映射
编写一个User对象->User.java


    public class User {
    
        private int id;
        private String username;
        private String password;
        private String cellphone;
    
        //各种setter和getter
    }
编写对象映射->User.hbm.xml。一般它和JavaBean对象放在同一目录下

我们是不知道该XML是怎么写的，可以搜索一下Hibernate文件夹中后缀为.hbm.xml。看看它们是怎么写的。然后复制一份过来

    
    <?xml version="1.0"?>
    <!DOCTYPE hibernate-mapping PUBLIC 
        "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
    
    <!-- 
    
      This mapping demonstrates content-based discrimination for the
      table-per-hierarchy mapping strategy, using a formula
      discriminator.
    
    -->
    
    <hibernate-mapping 
        package="org.hibernate.test.array">
    
        <class name="A" lazy="true" table="aaa">
    
            <id name="id">
                <generator class="native"/>
            </id>
    
            
                <key column="a_id"/>
                <list-index column="idx"/>
                <one-to-many class="B"/>
            
    
        </class>
    
        <class name="B" lazy="true" table="bbb">
            <id name="id">
                <generator class="native"/>
            </id>
        </class>
    
    </hibernate-mapping>
    
在上面的模板上修改～下面会具体讲解这个配置文件!

    <!--在domain包下-->
        <hibernate-mapping package="zhongfucheng.domain">
        
            <!--类名为User，表名也为User-->
            <class name="User"  table="user">
        
                <!--主键映射，属性名为id，列名也为id-->
                <id name="id" column="id">
                    <!--根据底层数据库主键自动增长-->
                    <generator class="native"/>
        
                </id>
        
                <!--非主键映射，属性和列名一一对应-->
                <property name="username" column="username"/>
                <property name="cellphone" column="cellphone"/>
                <property name="password" column="password"/>
            </class>
        </hibernate-mapping>

如果使用Intellij Idea生成的Hibernate可以指定生成出主配置文件hibernate.cfg.xml，它是要放在src目录下的

如果不是自动生成的，我们可以在Hibernate的hibernate-distribution-3.6.0.Final\project\etc这个目录下可以找到

它长得这个样子：

    
    <?xml version='1.0' encoding='utf-8'?>
    <!DOCTYPE hibernate-configuration PUBLIC
            "-//Hibernate/Hibernate Configuration DTD//EN"
            "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
    <hibernate-configuration>
        <session-factory>
            <property name="connection.url."/>
            <property name="connection.driver_class"/>
            <property name="connection.username"/>
            <property name="connection.password"/>
            <!-- DB schema will be updated if needed -->
            <!-- <property name="hbm2ddl.auto">update</property> -->
        </session-factory>
    </hibernate-configuration>
    
通过上面的模板进行修改，后面会有对该配置文件进行讲解！


    <hibernate-configuration>
        <!-- 通常，一个session-factory节点代表一个数据库 -->
        <session-factory>
    
            <!-- 1\. 数据库连接配置 -->
            <property name="hibernate.connection.driver_class">com.mysql.jdbc.Driver</property>
            <property name="hibernate.connection.url">jdbc:mysql:///zhongfucheng</property>
            <property name="hibernate.connection.username">root</property>
            <property name="hibernate.connection.password">root</property>
            <!--
                数据库方法配置， hibernate在运行的时候，会根据不同的方言生成符合当前数据库语法的sql
             -->
            <property name="hibernate.dialect">org.hibernate.dialect.MySQL5Dialect</property>
    
            <!-- 2\. 其他相关配置 -->
            <!-- 2.1 显示hibernate在运行时候执行的sql语句 -->
            <property name="hibernate.show_sql">true</property>
            <!-- 2.2 格式化sql -->
            <property name="hibernate.format_sql">true</property>
            <!-- 2.3 自动建表  -->
            <property name="hibernate.hbm2ddl.auto">create</property>
    
            <!--3\. 加载所有映射-->
            <mapping resource="zhongfucheng/domain/User.hbm.xml"/>
    
        </session-factory>
    </hibernate-configuration>

## 测试

    package zhongfucheng.domain;
    
    import org.hibernate.SessionFactory;
    import org.hibernate.Transaction;
    import org.hibernate.cfg.Configuration;
    import org.hibernate.classic.Session;

    /**
     * Created by ozc on 2017/5/6.
     */
    public class App {
        public static void main(String[] args) {
    
            //创建对象
            User user = new User();
            user.setPassword("123");
            user.setCellphone("122222");
            user.setUsername("nihao");
    
            //获取加载配置管理类
            Configuration configuration = new Configuration();
    
            //不给参数就默认加载hibernate.cfg.xml文件，
            configuration.configure();
    
            //创建Session工厂对象
            SessionFactory factory = configuration.buildSessionFactory();
    
            //得到Session对象
            Session session = factory.openSession();
    
            //使用Hibernate操作数据库，都要开启事务,得到事务对象
            Transaction transaction = session.getTransaction();
    
            //开启事务
            transaction.begin();
    
            //把对象添加到数据库中
            session.save(user);
    
            //提交事务
            transaction.commit();
    
            //关闭Session
            session.close();
        }
    }

值得注意的是：JavaBean的主键类型只能是int类型，因为在映射关系中配置是自动增长的，String类型是不能自动增长的。如果是你设置了String类型，又使用了自动增长，那么就会报出下面的错误！


    Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Table 'zhongfucheng.user' does
    
执行完程序后，Hibernate就为我们创建对应的表，并把数据存进了数据库了

我们看看快速入门案例的代码用到了什么对象吧，然后一个一个讲解


    public static void main(String[] args) {

        //创建对象
        User user = new User();
        user.setPassword("123");
        user.setCellphone("122222");
        user.setUsername("nihao");

        //获取加载配置管理类
        Configuration configuration = new Configuration();

        //不给参数就默认加载hibernate.cfg.xml文件，
        configuration.configure();

        //创建Session工厂对象
        SessionFactory factory = configuration.buildSessionFactory();

        //得到Session对象
        Session session = factory.openSession();

        //使用Hibernate操作数据库，都要开启事务,得到事务对象
        Transaction transaction = session.getTransaction();

        //开启事务
        transaction.begin();

        //把对象添加到数据库中
        session.save(user);

        //提交事务
        transaction.commit();

        //关闭Session
        session.close();
    }
## 相关类    
Configuration
配置管理类：主要管理配置文件的一个类

它拥有一个子类AnnotationConfiguration，也就是说：我们可以使用注解来代替XML配置文件来配置相对应的信息


configure方法
configure()方法用于加载配置文件

加载主配置文件的方法

如果指定参数，那么加载参数的路径配置文件
如果不指定参数，默认加载src/目录下的hibernate.cfg.xml

buildSessionFactory方法
buildSessionFactory()用于创建Session工厂


SessionFactory
SessionFactory-->Session的工厂，也可以说代表了hibernate.cfg.xml这个文件...hibernate.cfg.xml的就有<session-factory>这么一个节点

openSession方法
创建一个Session对象

getCurrentSession方法
创建Session对象或取出Session对象

Session
Session是Hibernate最重要的对象，Session维护了一个连接（Connection），只要使用Hibernate操作数据库，都需要用到Session对象

通常我们在DAO层中都会有以下的方法，Session也为我们提供了对应的方法来实现！


    public interface IEmployeeDao {
    
        void save(Employee emp);
        void update(Employee emp);
        Employee findById(Serializable id);
        List<Employee> getAll();
        List<Employee> getAll(String employeeName);
        List<Employee> getAll(int index, int count);
        void delete(Serializable id);
    
    }

更新操作

我们在快速入门中使用到了save(Objcet o)方法，调用了这个方法就把对象保存在数据库之中了。Session对象还提供着其他的方法来进行对数据库的更新

session.save(obj); 【保存一个对象】
session.update(obj); 【更新一个对象】
session.saveOrUpdate(obj); 【保存或者更新的方法】

没有设置主键，执行保存；
有设置主键，执行更新操作;
如果设置主键不存在报错！
我们来使用一下update()方法吧....既然是更新操作了，那么肯定需要设置主键的，不设置主键，数据库怎么知道你要更新什么。将id为1的记录修改成如下：

        user.setId(1);
        user.setPassword("qwer");
        user.setCellphone("1111");
        user.setUsername("zhongfucheng");

主键查询

通过主键来查询数据库的记录，从而返回一个JavaBean对象

session.get(javaBean.class, int id); 【传入对应的class和id就可以查询】
session.load(javaBean.class, int id); 【支持懒加载】
User重写toString()来看一下效果：


       User user1 = (User) session.get(User.class, 1);
        System.out.println(user1);

HQL查询

HQL:hibernate query language 即hibernate提供的面向对象的查询语言

查询的是对象以及对象的属性【它查询的是对象以及属性，因此是区分大小写的！】。

    SQL：Struct query language 结构化查询语言

查询的是表以及列【不区分大小写】
HQL是面向对象的查询语言，可以用来查询全部的数据！


        Query query = session.createQuery("FROM User");

        List list = query.list();
        System.out.println(list);

当然啦，它也可以传递参数进去查询


        Query query = session.createQuery("FROM User WHERE id=?");

        //这里的？号是从0开始的，并不像JDBC从1开始的！
        query.setParameter(0, user.getId());

        List list = query.list();
        System.out.println(list);



QBC查询

QBC查询: query by criteria 完全面向对象的查询

从上面的HQL查询，我们就可以发现：HQL查询是需要SQL的基础的，因为还是要写少部分的SQL代码....QBC查询就是完全的面向对象查询...但是呢，我们用得比较少

我们来看一下怎么使用吧：


        //创建关于user对象的criteria对象
        Criteria criteria = session.createCriteria(User.class);

        //添加条件
        criteria.add(Restrictions.eq("id", 1));

        //查询全部数据
        List list = criteria.list();
        System.out.println(list);


本地SQL查询

有的时候，如果SQL是非常复杂的，我们不能靠HQL查询来实现功能的话，我们就需要使用原生的SQL来进行复杂查询了！

但是呢，它有一个缺陷：它是不能跨平台的...因此我们在主配置文件中已经配置了数据库的“方言“了。

我们来简单使用一下把：


        //将所有的记录封装成User对象存进List集合中
        SQLQuery sqlQuery = session.createSQLQuery("SELECT * FROM user").addEntity(User.class);

        List list = sqlQuery.list();

        System.out.println(list);

beginTransaction方法

开启事务，返回的是一个事务对象....Hibernate规定所有的数据库操作都必须在事务环境下进行，否则报错！



Hibernate注解开发

在Hibernate中我们一般都会使用注解，这样可以帮助我们大大简化hbm映射文件的配置。下面我就来为大家详细介绍。

PO类注解配置

首先肯定是搭建好Hibernate的开发环境啦，我在此也不过多赘述，读者自行实践。接着在src目录下创建一个cn.itheima.domain包，并在该包下创建一个Book实体类，由于Book实体类中写有注解配置，所以就不用编写那个映射配置文件啦！

    @Entity // 定义了一个实体
    @Table(name="t_book",catalog="hibernateTest")
    public class Book {

    @Id // 这表示一个主键
    // @GeneratedValue 相当于native主键生成策略
    @GeneratedValue(strategy=GenerationType.IDENTITY) // 相当于identity主键生成策略
    private Integer id; // 主键

    @Column(name="c_name", length=30, nullable=true)
    private String name;

    @Temporal(TemporalType.TIMESTAMP) // 是用来定义日期类型
    private Date publicationDate; // 出版日期

    @Type(type="double") // 允许你去指定Hibernate里面的一些类型
    private Double price; // 价格，如果没有添加注解，也会自动的生成在表中

    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public Date getPublicationDate() {
        return publicationDate;
    }
    public void setPublicationDate(Date publicationDate) {
        this.publicationDate = publicationDate;
    }
    public Double getPrice() {
        return price;
    }
    public void setPrice(Double price) {
        this.price = price;
    }

}

下面我就来详细说一下Book实体类中的注解。

    @Entity：声明一个实体。
    @Table：来描述类与表之间的对应关系。

    @Entity // 定义了一个实体
    @Table(name="t_book",catalog="hibernateTest")
    public class Book {
        ......
    }

    @id：声明一个主键。
    @GeneratedValue：用它来声明一个主键生成策略。默认情况是native主键生成策略。可以选择的主键生成策略有：AUTO、IDENTITY、SEQUENCE

    @Id // 这表示一个主键
    // @GeneratedValue 相当于native主键生成策略
    @GeneratedValue(strategy=GenerationType.IDENTITY) // 相当于identity主键生成策略
    private Integer id; // 主键
    
    @Column：定义列。
    
    @Column(name="c_name", length=30, nullable=true)
    private String name;

**注意：对于PO类中所有属性，如果你不写注解，默认情况下也会在表中生成对应的列，列的名称就是属性的名称，列的类型也即属性的类型**。
    @Temporal：声明日期类型。
    
    @Temporal(TemporalType.TIMESTAMP) // 是用来定义日期类型
    private Date publicationDate; // 出版日期

日期类型可以选择的有：

    *   TemporalType.DATA：只有年月日。
    *   TemporalType.TIME：只有小时分钟秒。
    *   TemporalType.TIMESTAMP：有年月日小时分钟秒。
    @Type：可允许你去指定Hibernate里面的一些类型。
    
    @Type(type="double") // 允许你去指定Hibernate里面的一些类型
    private Double price; // 价格，如果没有添加注解，也会自动的生成在表中

最后我们在src目录下创建一个cn.itheima.test包，在该包下编写一个HibernateAnnotationTest单元测试类，并在该类中编写一个用于测试PO类的注解开发的方法：

    public class HibernateAnnotationTest {
    
        // 测试PO的注解开发
        @Test
        public void test1() {
            Session session = HibernateUtils.openSession();
            session.beginTransaction();
    
            Book b = new Book();
            b.setName("情书");
            b.setPrice(56.78);
            b.setPublicationDate(new Date());
    
            session.save(b);
    
            session.getTransaction().commit();
            session.close();
        }
    
    }

现在来思考两个问题：

如果主键生成策略我们想使用UUID类型呢？
如何设定类的属性不在表中映射？
这两个问题我们一起解决。废话不多说，直接上例子。在cn.itheima.domain包下再编写一个Person实体类，同样使用注解配置。

    @Entity
    @Table(name="t_person", catalog="hibernateTest")
    public class Person {
    
        // 生成UUID的主键生成策略
        @Id
        @GenericGenerator(name="myuuid", strategy="uuid") // 声明一种主键生成策略(uuid)
        @GeneratedValue(generator="myuuid") // 引用uuid主键生成策略
        private String id;
    
        @Type(type="string") // 允许你去指定Hibernate里面的一些类型
        private String name;
    
        @Transient
        private String msg; // 现在这个属性不想生成在表中
    
        public String getId() {
            return id;
        }
    
        public void setId(String id) {
            this.id = id;
        }
    
        public String getName() {
            return name;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    
        public String getMsg() {
            return msg;
        }
    
        public void setMsg(String msg) {
            this.msg = msg;
        }
    
    }



最后在HibernateAnnotationTest单元测试类中编写如下一个方法：

    public class HibernateAnnotationTest {
    
        // 测试uuid的主键生成策略及不生成表中映射
        @Test
        public void test2() {
            Session session = HibernateUtils.openSession();
            session.beginTransaction();
    
            Person p = new Person();
            p.setName("李四");
            p.setMsg("这是一个好人");
    
            session.save(p);
    
            session.getTransaction().commit();
            session.close();
        }
    
    }

至此，两个问题就解决了。 
注意：对于我们以上讲解的关于属性配置的注解，我们也可以在其对应的getXxx方法去使用。

Hibernate关联映射——一对多（多对一）
仍以客户(Customer)和订单(Order)为例来开始我的表演。 
在src目录下创建一个cn.itheima.oneToMany包，并在该包编写这两个实体类：

客户(Customer)类
    
        // 客户 ---- 一的一方
        @Entity
        @Table(name="t_customer")
        public class Customer {
    
        @Id
        @GeneratedValue(strategy=GenerationType.IDENTITY)
        private Integer id; // 主键
        private String name; // 姓名
    
        // 描述客户可以有多个订单
        /*
         * targetEntity="..."：相当于<one-to-many >
         */
        @OneToMany(targetEntity=Order.class,mappedBy="c")
        private Set<Order> orders = new HashSet<Order>();
    
        public Set<Order> getOrders() {
            return orders;
        }
        public void setOrders(Set<Order> orders) {
            this.orders = orders;
        }
        public Integer getId() {
            return id;
        }
        public void setId(Integer id) {
            this.id = id;
        }
        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }
    
        }
    
    
        订单(Order)类
        
        // 订单 ---- 多的一方
        @Entity
        @Table(name="t_order")
        public class Order {
    
        @Id
        @GeneratedValue(strategy=GenerationType.IDENTITY)
        private Integer id;
        private Double money;
        private String receiverInfo; // 收货地址
    
        // 订单与客户关联
        @ManyToOne(targetEntity=Customer.class)
        @JoinColumn(name="c_customer_id") // 指定外键列
        private Customer c; // 描述订单属于某一个客户
    
        public Customer getC() {
            return c;
        }
        public void setC(Customer c) {
            this.c = c;
        }
        public Integer getId() {
            return id;
        }
        public void setId(Integer id) {
            this.id = id;
        }
        public Double getMoney() {
            return money;
        }
        public void setMoney(Double money) {
            this.money = money;
        }
        public String getReceiverInfo() {
            return receiverInfo;
        }
        public void setReceiverInfo(String receiverInfo) {
            this.receiverInfo = receiverInfo;
        }
    
    }


这儿用到了`@OneToMany`和`@ManyToOne`这两个注解。
以上两个实体类编写好之后，可以很明显的看出我们不需要写它们对应的映射配置文件了，是不是很爽呢！接下来，我就要编写测试程序测试一下了。现在我的需求是保存客户时，顺便保存订单，对于这种情况我们需要在Customer类中配置cascade操作，即配置cascade="save-update"，配置的方式有两种，下面我细细说来：

第一种方式，可以使用JPA提供的注解。 
那么@OneToMany注解就应修改为：

    @OneToMany(targetEntity=Order.class,mappedBy="c",cascade=CascadeType.ALL)
    private Set<Order> orders = new HashSet<Order>();

第二种方式，可以使用Hibernate提供的注解。 
那么@OneToMany注解就应修改为：

    @OneToMany(targetEntity=Order.class,mappedBy="c")
    @Cascade(CascadeType.SAVE_UPDATE)
    private Set<Order> orders = new HashSet<Order>();

两种方式都可以，口味任君选择，不过我倾向于第二种方式。 
接下来在HibernateAnnotationTest单元测试类中编写如下方法进行测试：

    public class HibernateAnnotationTest {
    
        // 测试one-to-many注解操作(保存客户时级联保存订单)
        @Test
        public void test3() {
            Session session = HibernateUtils.openSession();
            session.beginTransaction();
    
            // 1.创建一个客户
            Customer c = new Customer();
            c.setName("叶子");
    
            // 2.创建两个订单
            Order o1 = new Order();
            o1.setMoney(1000d);
            o1.setReceiverInfo("武汉");
            Order o2 = new Order();
            o2.setMoney(2000d);
            o2.setReceiverInfo("天门");
    
            // 3.建立关系
            c.getOrders().add(o1);
            c.getOrders().add(o2);
    
            // 4.保存客户，并级联保存订单
            session.save(c);
    
            session.getTransaction().commit();
            session.close();
        }
    
    }

这时运行以上方法，会发现虽然客户表的那条记录插进去了，但是订单表就变成这个鬼样了： 

订单表中没有关联客户的id，这是为什么呢？原因是我们在Customer类中配置了mappedBy=”c”，它代表的是外键的维护由Order方来维护，而Customer不维护，这时你在保存客户时，级联保存订单，是可以的，但是不能维护外键，所以，我们必须在代码中添加订单与客户之间的关系。所以须将test3方法修改为：

    public class HibernateAnnotationTest {
    
        // 测试one-to-many注解操作(保存客户时级联保存订单)
        @Test
        public void test3() {
            Session session = HibernateUtils.openSession();
            session.beginTransaction();
    
            // 1.创建一个客户
            Customer c = new Customer();
            c.setName("叶子");
    
            // 2.创建两个订单
            Order o1 = new Order();
            o1.setMoney(1000d);
            o1.setReceiverInfo("武汉");
            Order o2 = new Order();
            o2.setMoney(2000d);
            o2.setReceiverInfo("天门");
    
            // 3.建立关系
            // 原因：是为了维护外键，不然的话，外键就不能正确的生成！！！
            o1.setC(c);
            o2.setC(c);
    
            // 原因：是为了进行级联操作
            c.getOrders().add(o1);
            c.getOrders().add(o2);
    
            // 4.保存客户，并级联保存订单
            session.save(c);
    
            session.getTransaction().commit();
            session.close();
        }
    
    }

这时再测试，就没有任何问题啦！

## 扩展
Hibernate注解@Cascade中的DELETE_ORPHAN已经过时了，如下： 

可使用下面方案来替换过时方案： 

Hibernate关联映射——多对多
以学生与老师为例开始我的表演，我是使用注解完成这种多对多的配置。使用@ManyToMany注解来配置多对多，只需要在一端配置中间表，另一端使用mappedBy表示放置外键的维护权。 
在src目录下创建一个cn.itheima.manyToMany包，并在该包编写这两个实体类：

学生类

    @Entity
    @Table(name="t_student")
    public class Student {
    
        @Id
        @GeneratedValue(strategy=GenerationType.IDENTITY)
        private Integer id;
    
        private String name;
    
        @ManyToMany(targetEntity=Teacher.class)
        // @JoinTable：使用@JoinTable来描述中间表，并描述中间表中外键与Student、Teacher的映射关系
        // joinColumns：它是用来描述Student与中间表的映射关系
        // inverseJoinColumns：它是用来描述Teacher与中间表的映射关系
        @JoinTable(name="s_t", joinColumns={@JoinColumn(name="c_student_id",referencedColumnName="id")}, inverseJoinColumns={@JoinColumn(name="c_teacher_id")}) 
        private Set<Teacher> teachers = new HashSet<Teacher>();
    
        public Integer getId() {
            return id;
        }
        public void setId(Integer id) {
            this.id = id;
        }
        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }
        public Set<Teacher> getTeachers() {
            return teachers;
        }
        public void setTeachers(Set<Teacher> teachers) {
            this.teachers = teachers;
        }
    
    }

老师类

    @Entity
    @Table(name="t_teacher")
    public class Teacher {
    
        @Id
        @GeneratedValue(strategy=GenerationType.IDENTITY)
        private Integer id;
    
        private String name;
    
        @ManyToMany(targetEntity=Student.class, mappedBy="teachers") // 代表由对方来维护外键
        private Set<Student> students = new HashSet<Student>();
    
        public Integer getId() {
            return id;
        }
        public void setId(Integer id) {
            this.id = id;
        }
        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }
        public Set<Student> getStudents() {
            return students;
        }
        public void setStudents(Set<Student> students) {
            this.students = students;
        }
    
    }

接下来，我就要编写测试程序测试一下了。 从上面可看出我们将外键的维护权利交由Student类来维护，现在我们演示保存学生时，将老师也级联保存，对于这种情况我们需要在Student类中配置cascade操作，即配置cascade=”save-update”，如下：

    @JoinTable(name="s_t", joinColumns={@JoinColumn(name="c_student_id",referencedColumnName="id")}, inverseJoinColumns={@JoinColumn(name="c_teacher_id")}) 
    @Cascade(CascadeType.SAVE_UPDATE)
    private Set<Teacher> teachers = new HashSet<Teacher>();

接下来在HibernateAnnotationTest单元测试类中编写如下方法进行测试：

    public class HibernateAnnotationTest {
    
        // 测试多对多级联保存(保存学生时同时保存老师)
        @Test
        public void test4() {
            Session session = HibernateUtils.openSession();
            session.beginTransaction();
    
            // 1.创建两个老师
            Teacher t1 = new Teacher();
            t1.setName("Tom");
    
            Teacher t2 = new Teacher();
            t2.setName("Fox");
    
            // 2.创建两个学生
            Student s1 = new Student();
            s1.setName("张丹");
    
            Student s2 = new Student();
            s2.setName("叶紫");
    
            // 3.学生关联老师
            s1.getTeachers().add(t1);
            s1.getTeachers().add(t2);
    
            s2.getTeachers().add(t1);
            s2.getTeachers().add(t2);
    
            // 保存学生同时保存老师
            session.save(s1);
            session.save(s2);
    
            session.getTransaction().commit();
            session.close();
        }
    
    }

运行以上方法，一切正常。 
接着我们测试级联删除操作。见下图： 
这里写图片描述

可在HibernateAnnotationTest单元测试类中编写如下方法进行测试：

    public class HibernateAnnotationTest {
    
        // 测试多对多级联删除(前提是建立了双向的级联)
        @Test
        public void test5() {
            Session session = HibernateUtils.openSession();
            session.beginTransaction();
    
            Student s = session.get(Student.class, 1);
            session.delete(s);
    
            session.getTransaction().commit();
            session.close();
        }
    
    }

## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348

## 微信公众号

### 个人公众号：程序员黄小斜

​
黄小斜是 985 硕士，阿里巴巴Java工程师，在自学编程、技术求职、Java学习等方面有丰富经验和独到见解，希望帮助到更多想要从事互联网行业的程序员们。
​
作者专注于 JAVA 后端技术栈，热衷于分享程序员干货、学习经验、求职心得，以及自学编程和Java技术栈的相关干货。
​
黄小斜是一个斜杠青年，坚持学习和写作，相信终身学习的力量，希望和更多的程序员交朋友，一起进步和成长！

**原创电子书:**
关注微信公众号【程序员黄小斜】后回复【原创电子书】即可领取我原创的电子书《菜鸟程序员修炼手册：从技术小白到阿里巴巴Java工程师》这份电子书总结了我2年的Java学习之路，包括学习方法、技术总结、求职经验和面试技巧等内容，已经帮助很多的程序员拿到了心仪的offer！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取，包括Java、python、C++、大数据、机器学习、前端、移动端等方向的技术资料。


![](https://img-blog.csdnimg.cn/20190829222750556.jpg)


### 技术公众号：Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的微信公众号【Java技术江湖】

这是一位阿里 Java 工程师的技术小站。作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！


**Java工程师必备学习资源:** 
关注公众号后回复”Java“即可领取 Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源


![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

​                     
