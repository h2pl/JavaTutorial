
## 配置数据源

### Spring 配置数据源

Spring 配置数据源有多种方式，下面一一列举：

#### [#](https://dunwu.github.io/spring-tutorial/pages/1b774c/#%E4%BD%BF%E7%94%A8-jndi-%E6%95%B0%E6%8D%AE%E6%BA%90)使用 JNDI 数据源

如果 Spring 应用部署在支持 JNDI 的 WEB 服务器上（如 WebSphere、JBoss、Tomcat 等），就可以使用 JNDI 获取数据源。



```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:jee="http://www.springframework.org/schema/jee"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
http://www.springframework.org/schema/jee
http://www.springframework.org/schema/jee/spring-jee-3.2.xsd">

  <!-- 1.使用bean配置jndi数据源 -->
  <bean id="dataSource" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName" value="java:comp/env/jdbc/orclight" />
  </bean>

  <!-- 2.使用jee标签配置jndi数据源，与1等价，但是需要引入命名空间 -->
  <jee:jndi-lookup id="dataSource" jndi-name=" java:comp/env/jdbc/orclight" />
</beans>

```



#### [#](https://dunwu.github.io/spring-tutorial/pages/1b774c/#%E4%BD%BF%E7%94%A8%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5%E6%B1%A0)使用数据库连接池

Spring 本身并没有提供数据库连接池的实现，需要自行选择合适的数据库连接池。下面是一个使用 [Druid (opens new window)](https://github.com/alibaba/druid)作为数据库连接池的示例：



```
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
        init-method="init" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driver}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>

    <!-- 配置初始化大小、最小、最大 -->
    <property name="initialSize" value="1"/>
    <property name="minIdle" value="1"/>
    <property name="maxActive" value="10"/>

    <!-- 配置获取连接等待超时的时间 -->
    <property name="maxWait" value="10000"/>

    <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
    <property name="timeBetweenEvictionRunsMillis" value="60000"/>

    <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
    <property name="minEvictableIdleTimeMillis" value="300000"/>

    <property name="testWhileIdle" value="true"/>

    <!-- 这里建议配置为TRUE，防止取到的连接不可用 -->
    <property name="testOnBorrow" value="true"/>
    <property name="testOnReturn" value="false"/>

    <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
    <property name="poolPreparedStatements" value="true"/>
    <property name="maxPoolPreparedStatementPerConnectionSize"
              value="20"/>

    <!-- 这里配置提交方式，默认就是TRUE，可以不用配置 -->

    <property name="defaultAutoCommit" value="true"/>

    <!-- 验证连接有效与否的SQL，不同的数据配置不同 -->
    <property name="validationQuery" value="select 1 "/>
    <property name="filters" value="stat"/>
  </bean>

```



#### [#](https://dunwu.github.io/spring-tutorial/pages/1b774c/#%E5%9F%BA%E4%BA%8E-jdbc-%E9%A9%B1%E5%8A%A8%E7%9A%84%E6%95%B0%E6%8D%AE%E6%BA%90)基于 JDBC 驱动的数据源



```
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
  <property name="driverClassName" value="${jdbc.driver}"/>
  <property name="url" value="${jdbc.url}"/>
  <property name="username" value="${jdbc.username}"/>
  <property name="password" value="${jdbc.password}"/>
</bean>
```


#### 使用JDBC

最后更新: 2022/11/16 20:10 / 阅读: 946668

* * *



我们在前面介绍[JDBC编程](https://www.liaoxuefeng.com/wiki/1252599548343744/1255943820274272)时已经讲过，Java程序使用JDBC接口访问关系数据库的时候，需要以下几步：

*   创建全局`DataSource`实例，表示数据库连接池；
*   在需要读写数据库的方法内部，按如下步骤访问数据库：
    *   从全局`DataSource`实例获取`Connection`实例；
    *   通过`Connection`实例创建`PreparedStatement`实例；
    *   执行SQL语句，如果是查询，则通过`ResultSet`读取结果集，如果是修改，则获得`int`结果。

正确编写JDBC代码的关键是使用`try ... finally`释放资源，涉及到事务的代码需要正确提交或回滚事务。

在Spring使用JDBC，首先我们通过IoC容器创建并管理一个`DataSource`实例，然后，Spring提供了一个`JdbcTemplate`，可以方便地让我们操作JDBC，因此，通常情况下，我们会实例化一个`JdbcTemplate`。顾名思义，这个类主要使用了[Template模式](https://www.liaoxuefeng.com/wiki/1252599548343744/1281319636041762)。

编写示例代码或者测试代码时，我们强烈推荐使用[HSQLDB](http://hsqldb.org/)这个数据库，它是一个用Java编写的关系数据库，可以以内存模式或者文件模式运行，本身只有一个jar包，非常适合演示代码或者测试代码。

我们以实际工程为例，先创建Maven工程`spring-data-jdbc`，然后引入以下依赖：

*   org.springframework:spring-context:6.0.0
*   org.springframework:spring-jdbc:6.0.0
*   jakarta.annotation:jakarta.annotation-api:2.1.1
*   com.zaxxer:HikariCP:5.0.1
*   org.hsqldb:hsqldb:2.7.1

在`AppConfig`中，我们需要创建以下几个必须的Bean：

```
@Configuration
@ComponentScan
@PropertySource("jdbc.properties")
public class AppConfig {

    @Value("${jdbc.url}")
    String jdbcUrl;

    @Value("${jdbc.username}")
    String jdbcUsername;

    @Value("${jdbc.password}")
    String jdbcPassword;

    @Bean
    DataSource createDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(jdbcUsername);
        config.setPassword(jdbcPassword);
        config.addDataSourceProperty("autoCommit", "true");
        config.addDataSourceProperty("connectionTimeout", "5");
        config.addDataSourceProperty("idleTimeout", "60");
        return new HikariDataSource(config);
    }

    @Bean
    JdbcTemplate createJdbcTemplate(@Autowired DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}

```

在上述配置中：

1.  通过`@PropertySource("jdbc.properties")`读取数据库配置文件；
2.  通过`@Value("${jdbc.url}")`注入配置文件的相关配置；
3.  创建一个DataSource实例，它的实际类型是`HikariDataSource`，创建时需要用到注入的配置；
4.  创建一个JdbcTemplate实例，它需要注入`DataSource`，这是通过方法参数完成注入的。

最后，针对HSQLDB写一个配置文件`jdbc.properties`：

```
# 数据库文件名为testdb:
jdbc.url=jdbc:hsqldb:file:testdb

# Hsqldb默认的用户名是sa，口令是空字符串:
jdbc.username=sa
jdbc.password=

```

可以通过HSQLDB自带的工具来初始化数据库表，这里我们写一个Bean，在Spring容器启动时自动创建一个`users`表：

```
@Component
public class DatabaseInitializer {
    @Autowired
    JdbcTemplate jdbcTemplate;

    @PostConstruct
    public void init() {
        jdbcTemplate.update("CREATE TABLE IF NOT EXISTS users (" //
                + "id BIGINT IDENTITY NOT NULL PRIMARY KEY, " //
                + "email VARCHAR(100) NOT NULL, " //
                + "password VARCHAR(100) NOT NULL, " //
                + "name VARCHAR(100) NOT NULL, " //
                + "UNIQUE (email))");
    }
}

```

现在，所有准备工作都已完毕。我们只需要在需要访问数据库的Bean中，注入`JdbcTemplate`即可：

```
@Component
public class UserService {
    @Autowired
    JdbcTemplate jdbcTemplate;
    ...
}

```

### JdbcTemplate用法

Spring提供的`JdbcTemplate`采用Template模式，提供了一系列以回调为特点的工具方法，目的是避免繁琐的`try...catch`语句。

我们以具体的示例来说明JdbcTemplate的用法。

首先我们看`T execute(ConnectionCallback<T> action)`方法，它提供了Jdbc的`Connection`供我们使用：

```
public User getUserById(long id) {
    // 注意传入的是ConnectionCallback:
    return jdbcTemplate.execute((Connection conn) -> {
        // 可以直接使用conn实例，不要释放它，回调结束后JdbcTemplate自动释放:
        // 在内部手动创建的PreparedStatement、ResultSet必须用try(...)释放:
        try (var ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
            ps.setObject(1, id);
            try (var rs = ps.executeQuery()) {
                if (rs.next()) {
                    return new User( // new User object:
                            rs.getLong("id"), // id
                            rs.getString("email"), // email
                            rs.getString("password"), // password
                            rs.getString("name")); // name
                }
                throw new RuntimeException("user not found by id.");
            }
        }
    });
}

```

也就是说，上述回调方法允许获取Connection，然后做任何基于Connection的操作。

我们再看`T execute(String sql, PreparedStatementCallback<T> action)`的用法：

```
public User getUserByName(String name) {
    // 需要传入SQL语句，以及PreparedStatementCallback:
    return jdbcTemplate.execute("SELECT * FROM users WHERE name = ?", (PreparedStatement ps) -> {
        // PreparedStatement实例已经由JdbcTemplate创建，并在回调后自动释放:
        ps.setObject(1, name);
        try (var rs = ps.executeQuery()) {
            if (rs.next()) {
                return new User( // new User object:
                        rs.getLong("id"), // id
                        rs.getString("email"), // email
                        rs.getString("password"), // password
                        rs.getString("name")); // name
            }
            throw new RuntimeException("user not found by id.");
        }
    });
}

```

最后，我们看`T queryForObject(String sql, RowMapper<T> rowMapper, Object... args)`方法：

```
public User getUserByEmail(String email) {
    // 传入SQL，参数和RowMapper实例:
    return jdbcTemplate.queryForObject("SELECT * FROM users WHERE email = ?",
            (ResultSet rs, int rowNum) -> {
                // 将ResultSet的当前行映射为一个JavaBean:
                return new User( // new User object:
                        rs.getLong("id"), // id
                        rs.getString("email"), // email
                        rs.getString("password"), // password
                        rs.getString("name")); // name
            },
            email);
}

```

在`queryForObject()`方法中，传入SQL以及SQL参数后，`JdbcTemplate`会自动创建`PreparedStatement`，自动执行查询并返回`ResultSet`，我们提供的`RowMapper`需要做的事情就是把`ResultSet`的当前行映射成一个JavaBean并返回。整个过程中，使用`Connection`、`PreparedStatement`和`ResultSet`都不需要我们手动管理。

`RowMapper`不一定返回JavaBean，实际上它可以返回任何Java对象。例如，使用`SELECT COUNT(*)`查询时，可以返回`Long`：

```
public long getUsers() {
    return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM users", (ResultSet rs, int rowNum) -> {
        // SELECT COUNT(*)查询只有一列，取第一列数据:
        return rs.getLong(1);
    });
}

```

如果我们期望返回多行记录，而不是一行，可以用`query()`方法：

```
public List<User> getUsers(int pageIndex) {
    int limit = 100;
    int offset = limit * (pageIndex - 1);
    return jdbcTemplate.query("SELECT * FROM users LIMIT ? OFFSET ?",
            new BeanPropertyRowMapper<>(User.class),
            limit, offset);
}

```

上述`query()`方法传入的参数仍然是SQL、SQL参数以及`RowMapper`实例。这里我们直接使用Spring提供的`BeanPropertyRowMapper`。如果数据库表的结构恰好和JavaBean的属性名称一致，那么`BeanPropertyRowMapper`就可以直接把一行记录按列名转换为JavaBean。

如果我们执行的不是查询，而是插入、更新和删除操作，那么需要使用`update()`方法：

```
public void updateUser(User user) {
    // 传入SQL，SQL参数，返回更新的行数:
    if (1 != jdbcTemplate.update("UPDATE users SET name = ? WHERE id = ?", user.getName(), user.getId())) {
        throw new RuntimeException("User not found by id");
    }
}

```

只有一种`INSERT`操作比较特殊，那就是如果某一列是自增列（例如自增主键），通常，我们需要获取插入后的自增值。`JdbcTemplate`提供了一个`KeyHolder`来简化这一操作：

```
public User register(String email, String password, String name) {
    // 创建一个KeyHolder:
    KeyHolder holder = new GeneratedKeyHolder();
    if (1 != jdbcTemplate.update(
        // 参数1:PreparedStatementCreator
        (conn) -> {
            // 创建PreparedStatement时，必须指定RETURN_GENERATED_KEYS:
            var ps = conn.prepareStatement("INSERT INTO users(email, password, name) VALUES(?, ?, ?)",
                    Statement.RETURN_GENERATED_KEYS);
            ps.setObject(1, email);
            ps.setObject(2, password);
            ps.setObject(3, name);
            return ps;
        },
        // 参数2:KeyHolder
        holder)
    ) {
        throw new RuntimeException("Insert failed.");
    }
    // 从KeyHolder中获取返回的自增值:
    return new User(holder.getKey().longValue(), email, password, name);
}

```

`JdbcTemplate`还有许多重载方法，这里我们不一一介绍。需要强调的是，`JdbcTemplate`只是对JDBC操作的一个简单封装，它的目的是尽量减少手动编写`try(resource) {...}`的代码，对于查询，主要通过`RowMapper`实现了JDBC结果集到Java对象的转换。

我们总结一下`JdbcTemplate`的用法，那就是：

*   针对简单查询，优选`query()`和`queryForObject()`，因为只需提供SQL语句、参数和`RowMapper`；
*   针对更新操作，优选`update()`，因为只需提供SQL语句和参数；
*   任何复杂的操作，最终也可以通过`execute(ConnectionCallback)`实现，因为拿到`Connection`就可以做任何JDBC操作。

实际上我们使用最多的仍然是各种查询。如果在设计表结构的时候，能够和JavaBean的属性一一对应，那么直接使用`BeanPropertyRowMapper`就很方便。如果表结构和JavaBean不一致怎么办？那就需要稍微改写一下查询，使结果集的结构和JavaBean保持一致。

例如，表的列名是`office_address`，而JavaBean属性是`workAddress`，就需要指定别名，改写查询如下：

```
SELECT id, email, office_address AS workAddress, name FROM users WHERE email = ?
```

使用`JdbcTemplate`的时候，我们用得最多的方法就是`List<T> query(String, RowMapper, Object...)`。这个`RowMapper`的作用就是把`ResultSet`的一行记录映射为Java Bean。

这种把关系数据库的表记录映射为Java对象的过程就是ORM：Object-Relational Mapping。ORM既可以把记录转换成Java对象，也可以把Java对象转换为行记录。

使用`JdbcTemplate`配合`RowMapper`可以看作是最原始的ORM。如果要实现更自动化的ORM，可以选择成熟的ORM框架，例如[Hibernate](https://hibernate.org/)。

我们来看看如何在Spring中集成Hibernate。

Hibernate作为ORM框架，它可以替代`JdbcTemplate`，但Hibernate仍然需要JDBC驱动，所以，我们需要引入JDBC驱动、连接池，以及Hibernate本身。在Maven中，我们加入以下依赖项：

*   org.springframework:spring-context:6.0.0
*   org.springframework:spring-orm:6.0.0
*   jakarta.annotation:jakarta.annotation-api:2.1.1
*   jakarta.persistence:jakarta.persistence-api:3.1.0
*   org.hibernate:hibernate-core:6.1.4.Final
*   com.zaxxer:HikariCP:5.0.1
*   org.hsqldb:hsqldb:2.7.1

在`AppConfig`中，我们仍然需要创建`DataSource`、引入JDBC配置文件，以及启用声明式事务：

```
@Configuration
@ComponentScan
@EnableTransactionManagement
@PropertySource("jdbc.properties")
public class AppConfig {
    @Bean
    DataSource createDataSource() {
        ...
    }
}

```

为了启用Hibernate，我们需要创建一个`LocalSessionFactoryBean`：

```
public class AppConfig {
    @Bean
    LocalSessionFactoryBean createSessionFactory(@Autowired DataSource dataSource) {
        var props = new Properties();
        props.setProperty("hibernate.hbm2ddl.auto", "update"); // 生产环境不要使用
        props.setProperty("hibernate.dialect", "org.hibernate.dialect.HSQLDialect");
        props.setProperty("hibernate.show_sql", "true");
        var sessionFactoryBean = new LocalSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource);
        // 扫描指定的package获取所有entity class:
        sessionFactoryBean.setPackagesToScan("com.itranswarp.learnjava.entity");
        sessionFactoryBean.setHibernateProperties(props);
        return sessionFactoryBean;
    }
}

```

注意我们在[定制Bean](https://www.liaoxuefeng.com/wiki/1252599548343744/1308043627200545)中讲到过`FactoryBean`，`LocalSessionFactoryBean`是一个`FactoryBean`，它会再自动创建一个`SessionFactory`，在Hibernate中，`Session`是封装了一个JDBC `Connection`的实例，而`SessionFactory`是封装了JDBC `DataSource`的实例，即`SessionFactory`持有连接池，每次需要操作数据库的时候，`SessionFactory`创建一个新的`Session`，相当于从连接池获取到一个新的`Connection`。`SessionFactory`就是Hibernate提供的最核心的一个对象，但`LocalSessionFactoryBean`是Spring提供的为了让我们方便创建`SessionFactory`的类。

注意到上面创建`LocalSessionFactoryBean`的代码，首先用`Properties`持有Hibernate初始化`SessionFactory`时用到的所有设置，常用的设置请参考[Hibernate文档](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#configurations)，这里我们只定义了3个设置：

*   `hibernate.hbm2ddl.auto=update`：表示自动创建数据库的表结构，注意不要在生产环境中启用；
*   `hibernate.dialect=org.hibernate.dialect.HSQLDialect`：指示Hibernate使用的数据库是HSQLDB。Hibernate使用一种HQL的查询语句，它和SQL类似，但真正在“翻译”成SQL时，会根据设定的数据库“方言”来生成针对数据库优化的SQL；
*   `hibernate.show_sql=true`：让Hibernate打印执行的SQL，这对于调试非常有用，我们可以方便地看到Hibernate生成的SQL语句是否符合我们的预期。

除了设置`DataSource`和`Properties`之外，注意到`setPackagesToScan()`我们传入了一个`package`名称，它指示Hibernate扫描这个包下面的所有Java类，自动找出能映射为数据库表记录的JavaBean。后面我们会仔细讨论如何编写符合Hibernate要求的JavaBean。

紧接着，我们还需要创建`HibernateTransactionManager`：

```
public class AppConfig {
    @Bean
    PlatformTransactionManager createTxManager(@Autowired SessionFactory sessionFactory) {
        return new HibernateTransactionManager(sessionFactory);
    }
}

```

`HibernateTransactionManager`是配合Hibernate使用声明式事务所必须的。到此为止，所有的配置都定义完毕，我们来看看如何将数据库表结构映射为Java对象。

考察如下的数据库表：

```
CREATE TABLE user
    id BIGINT NOT NULL AUTO_INCREMENT,
    email VARCHAR(100) NOT NULL,
    password VARCHAR(100) NOT NULL,
    name VARCHAR(100) NOT NULL,
    createdAt BIGINT NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `email` (`email`)
);

```

其中，`id`是自增主键，`email`、`password`、`name`是`VARCHAR`类型，`email`带唯一索引以确保唯一性，`createdAt`存储整型类型的时间戳。用JavaBean表示如下：

```
public class User {
    private Long id;
    private String email;
    private String password;
    private String name;
    private Long createdAt;

    // getters and setters
    ...
}

```

这种映射关系十分易懂，但我们需要添加一些注解来告诉Hibernate如何把`User`类映射到表记录：

```
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(nullable = false, updatable = false)
    public Long getId() { ... }

    @Column(nullable = false, unique = true, length = 100)
    public String getEmail() { ... }

    @Column(nullable = false, length = 100)
    public String getPassword() { ... }

    @Column(nullable = false, length = 100)
    public String getName() { ... }

    @Column(nullable = false, updatable = false)
    public Long getCreatedAt() { ... }
}

```

如果一个JavaBean被用于映射，我们就标记一个`@Entity`。默认情况下，映射的表名是`user`，如果实际的表名不同，例如实际表名是`users`，可以追加一个`@Table(name="users")`表示：

```
@Entity
@Table(name="users)
public class User {
    ...
}

```

每个属性到数据库列的映射用`@Column()`标识，`nullable`指示列是否允许为`NULL`，`updatable`指示该列是否允许被用在`UPDATE`语句，`length`指示`String`类型的列的长度（如果没有指定，默认是`255`）。

对于主键，还需要用`@Id`标识，自增主键再追加一个`@GeneratedValue`，以便Hibernate能读取到自增主键的值。

细心的童鞋可能还注意到，主键`id`定义的类型不是`long`，而是`Long`。这是因为Hibernate如果检测到主键为`null`，就不会在`INSERT`语句中指定主键的值，而是返回由数据库生成的自增值，否则，Hibernate认为我们的程序指定了主键的值，会在`INSERT`语句中直接列出。`long`型字段总是具有默认值`0`，因此，每次插入的主键值总是0，导致除第一次外后续插入都将失败。

`createdAt`虽然是整型，但我们并没有使用`long`，而是`Long`，这是因为使用基本类型会导致findByExample查询会添加意外的条件，这里只需牢记，作为映射使用的JavaBean，所有属性都使用包装类型而不是基本类型。

使用Hibernate时，不要使用基本类型的属性，总是使用包装类型，如Long或Integer。

类似的，我们再定义一个`Book`类：

```
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(nullable = false, updatable = false)
    public Long getId() { ... }

    @Column(nullable = false, length = 100)
    public String getTitle() { ... }

    @Column(nullable = false, updatable = false)
    public Long getCreatedAt() { ... }
}

```

如果仔细观察`User`和`Book`，会发现它们定义的`id`、`createdAt`属性是一样的，这在数据库表结构的设计中很常见：对于每个表，通常我们会统一使用一种主键生成机制，并添加`createdAt`表示创建时间，`updatedAt`表示修改时间等通用字段。

不必在`User`和`Book`中重复定义这些通用字段，我们可以把它们提到一个抽象类中：

```
@MappedSuperclass
public abstract class AbstractEntity {

    private Long id;
    private Long createdAt;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(nullable = false, updatable = false)
    public Long getId() { ... }

    @Column(nullable = false, updatable = false)
    public Long getCreatedAt() { ... }

    @Transient
    public ZonedDateTime getCreatedDateTime() {
        return Instant.ofEpochMilli(this.createdAt).atZone(ZoneId.systemDefault());
    }

    @PrePersist
    public void preInsert() {
        setCreatedAt(System.currentTimeMillis());
    }
}

```

对于`AbstractEntity`来说，我们要标注一个`@MappedSuperclass`表示它用于继承。此外，注意到我们定义了一个`@Transient`方法，它返回一个“虚拟”的属性。因为`getCreatedDateTime()`是计算得出的属性，而不是从数据库表读出的值，因此必须要标注`@Transient`，否则Hibernate会尝试从数据库读取名为`createdDateTime`这个不存在的字段从而出错。

再注意到`@PrePersist`标识的方法，它表示在我们将一个JavaBean持久化到数据库之前（即执行INSERT语句），Hibernate会先执行该方法，这样我们就可以自动设置好`createdAt`属性。

有了`AbstractEntity`，我们就可以大幅简化`User`和`Book`：

```
@Entity
public class User extends AbstractEntity {

    @Column(nullable = false, unique = true, length = 100)
    public String getEmail() { ... }

    @Column(nullable = false, length = 100)
    public String getPassword() { ... }

    @Column(nullable = false, length = 100)
    public String getName() { ... }
}

```

注意到使用的所有注解均来自`jakarta.persistence`，它是JPA规范的一部分。这里我们只介绍使用注解的方式配置Hibernate映射关系，不再介绍传统的比较繁琐的XML配置。通过Spring集成Hibernate时，也不再需要`hibernate.cfg.xml`配置文件，用一句话总结：

使用Spring集成Hibernate，配合JPA注解，无需任何额外的XML配置。

类似`User`、`Book`这样的用于ORM的Java Bean，我们通常称之为Entity Bean。

最后，我们来看看如果对`user`表进行增删改查。因为使用了Hibernate，因此，我们要做的，实际上是对`User`这个JavaBean进行“增删改查”。我们编写一个`UserService`，注入`SessionFactory`：

```
@Component
@Transactional
public class UserService {
    @Autowired
    SessionFactory sessionFactory;
}

```

### Insert操作

要持久化一个`User`实例，我们只需调用`persist()`方法。以`register()`方法为例，代码如下：

```
public User register(String email, String password, String name) {
    // 创建一个User对象:
    User user = new User();
    // 设置好各个属性:
    user.setEmail(email);
    user.setPassword(password);
    user.setName(name);
    // 不要设置id，因为使用了自增主键
    // 保存到数据库:
    sessionFactory.getCurrentSession().persist(user);
    // 现在已经自动获得了id:
    System.out.println(user.getId());
    return user;
}

```

### Delete操作

删除一个`User`相当于从表中删除对应的记录。注意Hibernate总是用`id`来删除记录，因此，要正确设置`User`的`id`属性才能正常删除记录：

```
public boolean deleteUser(Long id) {
    User user = sessionFactory.getCurrentSession().byId(User.class).load(id);
    if (user != null) {
        sessionFactory.getCurrentSession().remove(user);
        return true;
    }
    return false;
}

```

通过主键删除记录时，一个常见的用法是先根据主键加载该记录，再删除。注意到当记录不存在时，`load()`返回`null`。

### Update操作

更新记录相当于先更新`User`的指定属性，然后调用`merge()`方法：

```
public void updateUser(Long id, String name) {
    User user = sessionFactory.getCurrentSession().byId(User.class).load(id);
    user.setName(name);
    sessionFactory.getCurrentSession().merge(user);
}

```

前面我们在定义`User`时，对有的属性标注了`@Column(updatable=false)`。Hibernate在更新记录时，它只会把`@Column(updatable=true)`的属性加入到`UPDATE`语句中，这样可以提供一层额外的安全性，即如果不小心修改了`User`的`email`、`createdAt`等属性，执行`update()`时并不会更新对应的数据库列。但也必须牢记：这个功能是Hibernate提供的，如果绕过Hibernate直接通过JDBC执行`UPDATE`语句仍然可以更新数据库的任意列的值。

最后，我们编写的大部分方法都是各种各样的查询。根据`id`查询我们可以直接调用`load()`，如果要使用条件查询，例如，假设我们想执行以下查询：

```
SELECT * FROM user WHERE email = ? AND password = ?

```

我们来看看可以使用什么查询。

### 使用HQL查询

一种常用的查询是直接编写Hibernate内置的HQL查询：

```
List<User> list = sessionFactory.getCurrentSession()
        .createQuery("from User u where u.email = ?1 and u.password = ?2", User.class)
        .setParameter(1, email).setParameter(2, password)
        .list();

```

和SQL相比，HQL使用类名和属性名，由Hibernate自动转换为实际的表名和列名。详细的HQL语法可以参考[Hibernate文档](https://docs.jboss.org/hibernate/orm/6.1/userguide/html_single/Hibernate_User_Guide.html#query-language)。

除了可以直接传入HQL字符串外，Hibernate还可以使用一种`NamedQuery`，它给查询起个名字，然后保存在注解中。使用`NamedQuery`时，我们要先在`User`类标注：

```
@NamedQueries(
    @NamedQuery(
        // 查询名称:
        name = "login",
        // 查询语句:
        query = "SELECT u FROM User u WHERE u.email = :e AND u.password = :pwd"
    )
)
@Entity
public class User extends AbstractEntity {
    ...
}

```

注意到引入的`NamedQuery`是`jakarta.persistence.NamedQuery`，它和直接传入HQL有点不同的是，占位符使用`:e`和`:pwd`。

使用`NamedQuery`只需要引入查询名和参数：

```
public User login(String email, String password) {
    List<User> list = sessionFactory.getCurrentSession()
        .createNamedQuery("login", User.class) // 创建NamedQuery
        .setParameter("e", email) // 绑定e参数
        .setParameter("pwd", password) // 绑定pwd参数
        .list();
    return list.isEmpty() ? null : list.get(0);
}

```

直接写HQL和使用`NamedQuery`各有优劣。前者可以在代码中直观地看到查询语句，后者可以在`User`类统一管理所有相关查询。

上一节我们讲了在Spring中集成Hibernate。Hibernate是第一个被广泛使用的ORM框架，但是很多小伙伴还听说过JPA：Java Persistence API，这又是啥？

在讨论JPA之前，我们要注意到JavaEE早在1999年就发布了，并且有Servlet、JMS等诸多标准。和其他平台不同，Java世界早期非常热衷于标准先行，各家跟进：大家先坐下来把接口定了，然后，各自回家干活去实现接口，这样，用户就可以在不同的厂家提供的产品进行选择，还可以随意切换，因为用户编写代码的时候只需要引用接口，并不需要引用具体的底层实现（想想JDBC）。

JPA就是JavaEE的一个ORM标准，它的实现其实和Hibernate没啥本质区别，但是用户如果使用JPA，那么引用的就是`jakarta.persistence`这个“标准”包，而不是`org.hibernate`这样的第三方包。因为JPA只是接口，所以，还需要选择一个实现产品，跟JDBC接口和MySQL驱动一个道理。

我们使用JPA时也完全可以选择Hibernate作为底层实现，但也可以选择其它的JPA提供方，比如[EclipseLink](https://www.eclipse.org/eclipselink/)。Spring内置了JPA的集成，并支持选择Hibernate或EclipseLink作为实现。这里我们仍然以主流的Hibernate作为JPA实现为例子，演示JPA的基本用法。

和使用Hibernate一样，我们只需要引入如下依赖：

*   org.springframework:spring-context:6.0.0
*   org.springframework:spring-orm:6.0.0
*   jakarta.annotation:jakarta.annotation-api:2.1.1
*   jakarta.persistence:jakarta.persistence-api:3.1.0
*   org.hibernate:hibernate-core:6.1.4.Final
*   com.zaxxer:HikariCP:5.0.1
*   org.hsqldb:hsqldb:2.7.1

实际上我们这里引入的依赖和上一节集成Hibernate引入的依赖完全一样，因为Hibernate既提供了它自己的接口，也提供了JPA接口，我们用JPA接口就相当于通过JPA操作Hibernate。

然后，在`AppConfig`中启用声明式事务管理，创建`DataSource`：

```
@Configuration
@ComponentScan
@EnableTransactionManagement
@PropertySource("jdbc.properties")
public class AppConfig {
    @Bean
    DataSource createDataSource() { ... }
}

```

使用Hibernate时，我们需要创建一个`LocalSessionFactoryBean`，并让它再自动创建一个`SessionFactory`。使用JPA也是类似的，我们也创建一个`LocalContainerEntityManagerFactoryBean`，并让它再自动创建一个`EntityManagerFactory`：

```
@Bean
public LocalContainerEntityManagerFactoryBean createEntityManagerFactory(@Autowired DataSource dataSource) {
    var emFactory = new LocalContainerEntityManagerFactoryBean();
    // 注入DataSource:
    emFactory.setDataSource(dataSource);
    // 扫描指定的package获取所有entity class:
    emFactory.setPackagesToScan(AbstractEntity.class.getPackageName());
    // 使用Hibernate作为JPA实现:
    emFactory.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
    // 其他配置项:
    var props = new Properties();
    props.setProperty("hibernate.hbm2ddl.auto", "update"); // 生产环境不要使用
    props.setProperty("hibernate.dialect", "org.hibernate.dialect.HSQLDialect");
    props.setProperty("hibernate.show_sql", "true");
    emFactory.setJpaProperties(props);
    return emFactory;
}

```

观察上述代码，除了需要注入`DataSource`和设定自动扫描的`package`外，还需要指定JPA的提供商，这里使用Spring提供的一个`HibernateJpaVendorAdapter`，最后，针对Hibernate自己需要的配置，以`Properties`的形式注入。

最后，我们还需要实例化一个`JpaTransactionManager`，以实现声明式事务：

```
@Bean
PlatformTransactionManager createTxManager(@Autowired EntityManagerFactory entityManagerFactory) {
    return new JpaTransactionManager(entityManagerFactory);
}

```

这样，我们就完成了JPA的全部初始化工作。有些童鞋可能从网上搜索得知JPA需要`persistence.xml`配置文件，以及复杂的`orm.xml`文件。这里我们负责地告诉大家，使用Spring+Hibernate作为JPA实现，无需任何配置文件。

所有Entity Bean的配置和上一节完全相同，全部采用Annotation标注。我们现在只需关心具体的业务类如何通过JPA接口操作数据库。

还是以`UserService`为例，除了标注`@Component`和`@Transactional`外，我们需要注入一个`EntityManager`，但是不要使用`Autowired`，而是`@PersistenceContext`：

```
@Component
@Transactional
public class UserService {
    @PersistenceContext
    EntityManager em;
}

```

我们回顾一下JDBC、Hibernate和JPA提供的接口，实际上，它们的关系如下：

| JDBC | Hibernate | JPA |
| --- | --- | --- |
| DataSource | SessionFactory | EntityManagerFactory |
| Connection | Session | EntityManager |

`SessionFactory`和`EntityManagerFactory`相当于`DataSource`，`Session`和`EntityManager`相当于`Connection`。每次需要访问数据库的时候，需要获取新的`Session`和`EntityManager`，用完后再关闭。

但是，注意到`UserService`注入的不是`EntityManagerFactory`，而是`EntityManager`，并且标注了`@PersistenceContext`。难道使用JPA可以允许多线程操作同一个`EntityManager`？

实际上这里注入的并不是真正的`EntityManager`，而是一个`EntityManager`的代理类，相当于：

```
public class EntityManagerProxy implements EntityManager {
    private EntityManagerFactory emf;
}

```

Spring遇到标注了`@PersistenceContext`的`EntityManager`会自动注入代理，该代理会在必要的时候自动打开`EntityManager`。换句话说，多线程引用的`EntityManager`虽然是同一个代理类，但该代理类内部针对不同线程会创建不同的`EntityManager`实例。

简单总结一下，标注了`@PersistenceContext`的`EntityManager`可以被多线程安全地共享。

因此，在`UserService`的每个业务方法里，直接使用`EntityManager`就很方便。以主键查询为例：

```
public User getUserById(long id) {
    User user = this.em.find(User.class, id);
    if (user == null) {
        throw new RuntimeException("User not found by id: " + id);
    }
    return user;
}

```

与HQL查询类似，JPA使用JPQL查询，它的语法和HQL基本差不多：

```
public User fetchUserByEmail(String email) {
    // JPQL查询:
    TypedQuery<User> query = em.createQuery("SELECT u FROM User u WHERE u.email = :e", User.class);
    query.setParameter("e", email);
    List<User> list = query.getResultList();
    if (list.isEmpty()) {
        return null;
    }
    return list.get(0);
}

```

同样的，JPA也支持`NamedQuery`，即先给查询起个名字，再按名字创建查询：

```
public User login(String email, String password) {
    TypedQuery<User> query = em.createNamedQuery("login", User.class);
    query.setParameter("e", email);
    query.setParameter("pwd", password);
    List<User> list = query.getResultList();
    return list.isEmpty() ? null : list.get(0);
}

```

`NamedQuery`通过注解标注在`User`类上，它的定义和上一节的`User`类一样：

```
@NamedQueries(
    @NamedQuery(
        name = "login",
        query = "SELECT u FROM User u WHERE u.email=:e AND u.password=:pwd"
    )
)
@Entity
public class User {
    ...
}

```

对数据库进行增删改的操作，可以分别使用`persist()`、`remove()`和`merge()`方法，参数均为Entity Bean本身，使用非常简单，这里不再多述。

#### 集成MyBatis

最后更新: 2022/11/16 21:07 / 阅读: 601258

* * *



使用Hibernate或JPA操作数据库时，这类ORM干的主要工作就是把ResultSet的每一行变成Java Bean，或者把Java Bean自动转换到INSERT或UPDATE语句的参数中，从而实现ORM。

而ORM框架之所以知道如何把行数据映射到Java Bean，是因为我们在Java Bean的属性上给了足够的注解作为元数据，ORM框架获取Java Bean的注解后，就知道如何进行双向映射。

那么，ORM框架是如何跟踪Java Bean的修改，以便在`update()`操作中更新必要的属性？

答案是使用[Proxy模式](https://www.liaoxuefeng.com/wiki/1252599548343744/1281319432618017)，从ORM框架读取的User实例实际上并不是User类，而是代理类，代理类继承自User类，但针对每个setter方法做了覆写：

```
public class UserProxy extends User {
    boolean _isNameChanged;

    public void setName(String name) {
        super.setName(name);
        _isNameChanged = true;
    }
}

```

这样，代理类可以跟踪到每个属性的变化。

针对一对多或多对一关系时，代理类可以直接通过getter方法查询数据库：

```
public class UserProxy extends User {
    Session _session;
    boolean _isNameChanged;

    public void setName(String name) {
        super.setName(name);
        _isNameChanged = true;
    }

    /**
     * 获取User对象关联的Address对象:
     */
    public Address getAddress() {
        Query q = _session.createQuery("from Address where userId = :userId");
        q.setParameter("userId", this.getId());
        List<Address> list = query.list();
        return list.isEmpty() ? null : list(0);
    }
}

```

为了实现这样的查询，UserProxy必须保存Hibernate的当前Session。但是，当事务提交后，Session自动关闭，此时再获取`getAddress()`将无法访问数据库，或者获取的不是事务一致的数据。因此，ORM框架总是引入了Attached/Detached状态，表示当前此Java Bean到底是在Session的范围内，还是脱离了Session变成了一个“游离”对象。很多初学者无法正确理解状态变化和事务边界，就会造成大量的`PersistentObjectException`异常。这种隐式状态使得普通Java Bean的生命周期变得复杂。

此外，Hibernate和JPA为了实现兼容多种数据库，它使用HQL或JPQL查询，经过一道转换，变成特定数据库的SQL，理论上这样可以做到无缝切换数据库，但这一层自动转换除了少许的性能开销外，给SQL级别的优化带来了麻烦。

最后，ORM框架通常提供了缓存，并且还分为一级缓存和二级缓存。一级缓存是指在一个Session范围内的缓存，常见的情景是根据主键查询时，两次查询可以返回同一实例：

```
User user1 = session.load(User.class, 123);
User user2 = session.load(User.class, 123);

```

二级缓存是指跨Session的缓存，一般默认关闭，需要手动配置。二级缓存极大的增加了数据的不一致性，原因在于SQL非常灵活，常常会导致意外的更新。例如：

```
// 线程1读取:
User user1 = session1.load(User.class, 123);
...
// 一段时间后，线程2读取:
User user2 = session2.load(User.class, 123);

```

当二级缓存生效的时候，两个线程读取的User实例是一样的，但是，数据库对应的行记录完全可能被修改，例如：

```
-- 给老用户增加100积分:
UPDATE users SET bonus = bonus + 100 WHERE createdAt <= ?

```

ORM无法判断`id=123`的用户是否受该`UPDATE`语句影响。考虑到数据库通常会支持多个应用程序，此UPDATE语句可能由其他进程执行，ORM框架就更不知道了。

我们把这种ORM框架称之为全自动ORM框架。

对比Spring提供的JdbcTemplate，它和ORM框架相比，主要有几点差别：

1.  查询后需要手动提供Mapper实例以便把ResultSet的每一行变为Java对象；
2.  增删改操作所需的参数列表，需要手动传入，即把User实例变为[user.id, user.name, user.email]这样的列表，比较麻烦。

但是JdbcTemplate的优势在于它的确定性：即每次读取操作一定是数据库操作而不是缓存，所执行的SQL是完全确定的，缺点就是代码比较繁琐，构造`INSERT INTO users VALUES (?,?,?)`更是复杂。

所以，介于全自动ORM如Hibernate和手写全部如JdbcTemplate之间，还有一种半自动的ORM，它只负责把ResultSet自动映射到Java Bean，或者自动填充Java Bean参数，但仍需自己写出SQL。[MyBatis](https://mybatis.org/)就是这样一种半自动化ORM框架。

我们来看看如何在Spring中集成MyBatis。

首先，我们要引入MyBatis本身，其次，由于Spring并没有像Hibernate那样内置对MyBatis的集成，所以，我们需要再引入MyBatis官方自己开发的一个与Spring集成的库：

*   org.mybatis:mybatis:3.5.11
*   org.mybatis:mybatis-spring:3.0.0

和前面一样，先创建`DataSource`是必不可少的：

```
@Configuration
@ComponentScan
@EnableTransactionManagement
@PropertySource("jdbc.properties")
public class AppConfig {
    @Bean
    DataSource createDataSource() { ... }
}

```

再回顾一下Hibernate和JPA的`SessionFactory`与`EntityManagerFactory`，MyBatis与之对应的是`SqlSessionFactory`和`SqlSession`：

| JDBC | Hibernate | JPA | MyBatis |
| --- | --- | --- | --- |
| DataSource | SessionFactory | EntityManagerFactory | SqlSessionFactory |
| Connection | Session | EntityManager | SqlSession |

可见，ORM的设计套路都是类似的。使用MyBatis的核心就是创建`SqlSessionFactory`，这里我们需要创建的是`SqlSessionFactoryBean`：

```
@Bean
SqlSessionFactoryBean createSqlSessionFactoryBean(@Autowired DataSource dataSource) {
    var sqlSessionFactoryBean = new SqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(dataSource);
    return sqlSessionFactoryBean;
}

```

因为MyBatis可以直接使用Spring管理的声明式事务，因此，创建事务管理器和使用JDBC是一样的：

```
@Bean
PlatformTransactionManager createTxManager(@Autowired DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}

```

和Hibernate不同的是，MyBatis使用Mapper来实现映射，而且Mapper必须是接口。我们以`User`类为例，在`User`类和`users`表之间映射的`UserMapper`编写如下：

```
public interface UserMapper {
	@Select("SELECT * FROM users WHERE id = #{id}")
	User getById(@Param("id") long id);
}

```

注意：这里的Mapper不是`JdbcTemplate`的`RowMapper`的概念，它是定义访问`users`表的接口方法。比如我们定义了一个`User getById(long)`的主键查询方法，不仅要定义接口方法本身，还要明确写出查询的SQL，这里用注解`@Select`标记。SQL语句的任何参数，都与方法参数按名称对应。例如，方法参数id的名字通过注解`@Param()`标记为`id`，则SQL语句里将来替换的占位符就是`#{id}`。

如果有多个参数，那么每个参数命名后直接在SQL中写出对应的占位符即可：

```
@Select("SELECT * FROM users LIMIT #{offset}, #{maxResults}")
List<User> getAll(@Param("offset") int offset, @Param("maxResults") int maxResults);

```

注意：MyBatis执行查询后，将根据方法的返回类型自动把ResultSet的每一行转换为User实例，转换规则当然是按列名和属性名对应。如果列名和属性名不同，最简单的方式是编写SELECT语句的别名：

```
-- 列名是created_time，属性名是createdAt:
SELECT id, name, email, created_time AS createdAt FROM users

```

执行INSERT语句就稍微麻烦点，因为我们希望传入User实例，因此，定义的方法接口与`@Insert`注解如下：

```
@Insert("INSERT INTO users (email, password, name, createdAt) VALUES (#{user.email}, #{user.password}, #{user.name}, #{user.createdAt})")
void insert(@Param("user") User user);

```

上述方法传入的参数名称是`user`，参数类型是User类，在SQL中引用的时候，以`#{obj.property}`的方式写占位符。和Hibernate这样的全自动化ORM相比，MyBatis必须写出完整的INSERT语句。

如果`users`表的`id`是自增主键，那么，我们在SQL中不传入`id`，但希望获取插入后的主键，需要再加一个`@Options`注解：

```
@Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")
@Insert("INSERT INTO users (email, password, name, createdAt) VALUES (#{user.email}, #{user.password}, #{user.name}, #{user.createdAt})")
void insert(@Param("user") User user);

```

`keyProperty`和`keyColumn`分别指出JavaBean的属性和数据库的主键列名。

执行`UPDATE`和`DELETE`语句相对比较简单，我们定义方法如下：

```
@Update("UPDATE users SET name = #{user.name}, createdAt = #{user.createdAt} WHERE id = #{user.id}")
void update(@Param("user") User user);

@Delete("DELETE FROM users WHERE id = #{id}")
void deleteById(@Param("id") long id);

```

有了`UserMapper`接口，还需要对应的实现类才能真正执行这些数据库操作的方法。虽然可以自己写实现类，但我们除了编写`UserMapper`接口外，还有`BookMapper`、`BonusMapper`……一个一个写太麻烦，因此，MyBatis提供了一个`MapperFactoryBean`来自动创建所有Mapper的实现类。可以用一个简单的注解来启用它：

```
@MapperScan("com.itranswarp.learnjava.mapper")
...其他注解...
public class AppConfig {
    ...
}

```

有了`@MapperScan`，就可以让MyBatis自动扫描指定包的所有Mapper并创建实现类。在真正的业务逻辑中，我们可以直接注入：

```
@Component
@Transactional
public class UserService {
    // 注入UserMapper:
    @Autowired
    UserMapper userMapper;

    public User getUserById(long id) {
        // 调用Mapper方法:
        User user = userMapper.getById(id);
        if (user == null) {
            throw new RuntimeException("User not found by id.");
        }
        return user;
    }
}

```

可见，业务逻辑主要就是通过`XxxMapper`定义的数据库方法来访问数据库。

### XML配置

上述在Spring中集成MyBatis的方式，我们只需要用到注解，并没有任何XML配置文件。MyBatis也允许使用XML配置映射关系和SQL语句，例如，更新`User`时根据属性值构造动态SQL：

```
<update id="updateUser">
  UPDATE users SET
  <set>
    <if test="user.name != null"> name = #{user.name} </if>
    <if test="user.hobby != null"> hobby = #{user.hobby} </if>
    <if test="user.summary != null"> summary = #{user.summary} </if>
  </set>
  WHERE id = #{user.id}
</update>

```

编写XML配置的优点是可以组装出动态SQL，并且把所有SQL操作集中在一起。缺点是配置起来太繁琐，调用方法时如果想查看SQL还需要定位到XML配置中。这里我们不介绍XML的配置方式，需要了解的童鞋请自行阅读[官方文档](https://mybatis.org/mybatis-3/zh/configuration.html)。

使用MyBatis最大的问题是所有SQL都需要全部手写，优点是执行的SQL就是我们自己写的SQL，对SQL进行优化非常简单，也可以编写任意复杂的SQL，或者使用数据库的特定语法，但切换数据库可能就不太容易。好消息是大部分项目并没有切换数据库的需求，完全可以针对某个数据库编写尽可能优化的SQL。

# 参考文章
https://www.w3cschool.cn/wkspring
https://www.runoob.com/w3cnote/basic-knowledge-summary-of-spring.html
http://codepub.cn/2015/06/21/Basic-knowledge-summary-of-Spring
https://dunwu.github.io/spring-tutorial
https://mszlu.com/java/spring
http://c.biancheng.net/spring/aop-module.html