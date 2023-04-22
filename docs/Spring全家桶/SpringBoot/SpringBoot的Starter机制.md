starter是SpringBoot中的一个新发明，它有效的降低了项目开发过程的复杂程度，对于简化开发操作有着非常好的效果。本文转载了一片文章，详细介绍了spring boot stater是什么？它的作用是什么？

Spring Boot Starter是在SpringBoot组件中被提出来的一种概念，stackoverflow上面已经有人概括了这个starter是什么东西，想看完整的回答戳[这里](https://stackoverflow.com/a/28273660)（https://stackoverflow.com/questions/28273543/what-are-spring-boot-starter-jars/28273660#28273660）

![](https://images2018.cnblogs.com/blog/697611/201804/697611-20180409110042391-1447358002.png)

大概意思就是说starter是一种对依赖的synthesize（合成），这是什么意思呢？我可以举个例子来说明。

### 　 ? 传统的做法

在没有starter之前，假如我想要在Spring中使用jpa，那我可能需要做以下操作：

1.  在Maven中引入使用的数据库的依赖（即JDBC的jar）
2.  引入jpa的依赖
3.  在xxx.xml中配置一些属性信息
4.  反复的调试直到可以正常运行

需要注意的是，这里操作在我们**_每次新建一个需要用到jpa的项目的时候都需要重复的做一次_**。也许你在第一次自己建立项目的时候是在Google上自己搜索了一番，花了半天时间解决掉了各种奇怪的问题之后，jpa终于能正常运行了。有些有经验的人会在OneNote上面把这次建立项目的过程给记录下来，包括操作的步骤以及需要用到的配置文件的内容，在下一次再创建jpa项目的时候，就不需要再次去Google了，只需要照着笔记来，之后再把所有的配置文件copy&paste就可以了。

像上面这样的操作也不算不行，事实上我们在没有starter之前都是这么干的，但是这样做有几个问题：

1.  如果过程比较繁琐，这样一步步操作会增加出错的可能性
2.  不停地copy&paste不符合[Don’t repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)精神
3.  在第一次配置的时候（尤其如果开发者比较小白），需要花费掉大量的时间

### 　　使用Spring Boot Starter提升效率

starter的主要目的就是为了解决上面的这些问题。

starter的理念：starter会把所有用到的依赖都给包含进来，避免了开发者自己去引入依赖所带来的麻烦。需要注意的是不同的starter是为了解决不同的依赖，所以它们内部的实现可能会有很大的差异，例如jpa的starter和Redis的starter可能实现就不一样，这是因为starter的本质在于synthesize，这是一层在逻辑层面的抽象，也许这种理念有点类似于Docker，因为它们都是在做一个“包装”的操作，如果你知道Docker是为了解决什么问题的，也许你可以用Docker和starter做一个类比。

starter的实现：虽然不同的starter实现起来各有差异，但是他们基本上都会使用到两个相同的内容：ConfigurationProperties和AutoConfiguration。因为Spring Boot坚信“约定大于配置”这一理念，所以我们使用ConfigurationProperties来保存我们的配置，并且这些配置都可以有一个默认值，即在我们没有主动覆写原始配置的情况下，默认值就会生效，这在很多情况下是非常有用的。除此之外，starter的ConfigurationProperties还使得所有的配置属性被聚集到一个文件中（一般在resources目录下的application.properties），这样我们就告别了Spring项目中XML地狱。

starter的整体逻辑：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/697611-20180409110236645-2097376440.png)

上面的starter依赖的jar和我们自己手动配置的时候依赖的jar并没有什么不同，所以我们可以认为starter其实是把这一些繁琐的配置操作交给了自己，而把简单交给了用户。除了帮助用户去除了繁琐的构建操作，在“约定大于配置”的理念下，ConfigurationProperties还帮助用户减少了无谓的配置操作。并且因为?`application.properties`?文件的存在，即使需要自定义配置，所有的配置也只需要在一个文件中进行，使用起来非常方便。

了解了starter其实就是帮助用户简化了配置的操作之后，要理解starter和被配置了starter的组件之间并不是竞争关系，而是辅助关系，即我们可以给一个组件创建一个starter来让最终用户在使用这个组件的时候更加的简单方便。基于这种理念，我们可以给任意一个现有的组件创建一个starter来让别人在使用这个组件的时候更加的简单方便，事实上Spring Boot团队已经帮助现有大部分的流行的组件创建好了它们的starter，你可以在[这里](https://github.com/spring-projects/spring-boot/tree/v1.5.7.RELEASE/spring-boot-starters)查看这些starter的列表。

用了springboot 那么久了居然都还没自定义过starter，想想都觉得羞愧，所以今天来玩一下。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416200541.png)

# SpringBoot starter

SpringBoot中的starter是一种非常重要的机制，能够抛弃以前繁杂的配置，将其统一集成进starter，应用者只需要在maven中引入starter依赖，SpringBoot就能自动扫描到要加载的信息并启动相应的默认配置。starter让我们摆脱了各种依赖库的处理，需要配置各种信息的困扰。SpringBoot会自动通过classpath路径下的类发现需要的Bean，并注册进IOC容器。SpringBoot提供了针对日常企业应用研发各种场景的spring-boot-starter依赖模块。所有这些依赖模块都遵循着约定成俗的默认配置，并允许我们调整这些配置，即遵循“约定大于配置”的理念。

# 自定义starter

日常工作中有时有一些独立于业务之外的功能或模块，可能这个项目在用，另一个项目也要用，如果每次都重新集成的话就会很麻烦，这时我们只要把这些功能或模块封装成一个个starter的话，在使用的时候引入进去就很方便了。

## 自定义starter步骤

其实自定义starter很简单，大致需要以下5步：

1.  新建两个模块，命名规范： springboot自带的starter命名规范为spring-boot-starter-xxx， 自定义的starter命名规范为xxx-spring-boot-starter

● xxx-spring-boot-autoconfigure：自动配置核心代码
● xxx-spring-boot-starter：管理依赖
如果不需要将自动配置代码和依赖项管理分离开来，则可以将它们组合到一个模块中。只不过springboot官方建议将两个模块分开。
2\. 引入spring-boot-autoconfigure依赖
3\. 创建自定义的XXXProperties 类: 这个类的属性根据需要是要出现在配置文件中的。
4\. 创建自定义的XXXAutoConfiguration类：这个类要配置自动配置时的一些逻辑，同时也要让XXXProperties 类生效。
5\. 创建自定义的spring.factories文件：在resources/META-INF创建一个spring.factories文件和spring-configuration-metadata.json，spring-configuration-metadata.json文件是用于在填写配置文件时的智能提示，可要可不要，有的话提示起来更友好。spring.factories用于导入自动配置类，必须要有

## 实现

我这里为了方便就只创建一个模块了，

1.  创建一个模块，命名为spring-boot-starter-my-starter，对应pom文件

```
	<groupId>com.example</groupId>
	spring-boot-starter-my-starter
	<version>1.0</version>
	<name>my-starter</name>
复制代码
```

1.  引入spring-boot-autoconfigure依赖 我这里使用的spring-boot-autoconfigure版本是2.6.2

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-autoconfigure
        <version>2.6.2</version>
    </dependency>
</dependencies>
复制代码
```

1.  创建自定义的XXXProperties 类

```
@ConfigurationProperties(prefix = "com.arron")
public class MyStarterProperties {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
复制代码
```

再创建一个MyStarterConfig用于读取MyStarterProperties 里的属性

```
public class MyStarterConfig {

    private MyStarterProperties myStarterProperties;

    private String name;

    public MyStarterConfig(MyStarterProperties myStarterProperties) {
        this.myStarterProperties = myStarterProperties;
    }

    public String getName() {
        return myStarterProperties.getName();
    }

    public void setName(String name) {
        this.name = name;
    }
}
复制代码
```

1.  创建自定义的XXXAutoConfiguration类

```
@Configuration
// EnableConfigurationProperties value数组中的配置类起作用
@EnableConfigurationProperties(value = {MyStarterProperties.class})
public class MyStarterAutoConfiguration {

    @Autowired
    private MyStarterProperties myStarterProperties;

    @Bean
    @ConditionalOnMissingBean(MyStarterConfig.class)
    public MyStarterConfig myStarterConfig(){
        return new MyStarterConfig(myStarterProperties);
    }

}
复制代码
```

1.  在resources/META-INF创建一个spring.factories文件

spring.factories

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.example.myStarter.MyStarterAutoConfiguration
复制代码
```

spring-configuration-metadata.json

```
{
  "group": [
    {
      "name": "com.arron",
      "type": "com.example.myStarter.MyStarterProperties",
      "sourceType": "com.example.myStarter.MyStarterProperties"
    }
  ],
  "properties": [
    {
      "name": "com.arron.name",
      "type": "java.lang.String",
      "description": "my start name",
      "sourceType": "com.example.myStarter.MyStarterProperties",
      "defaultValue": "MyStarterProperties name"
    }
  ]
}
复制代码
```

## 打包测试

找到如图maven，点击install，安装到本地 ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230416200527082.png)

然后新建一个项目导包进行测试，创建项目过程就不介绍了。

1.  引入依赖

```
	<dependency>
       <groupId>com.example</groupId>
       spring-boot-starter-my-starter
       <version>1.0</version>
   </dependency>
复制代码
```

1.  配置文件添加属性：

```
com:
  arron:
    name: myname
复制代码
```

1.  单元测试：

```
@RunWith(SpringRunner.class)
@SpringBootTest
class RabbitmqApplicationTests {
    @Autowired
    private MyStarterConfig myStarterConfig;

    @Test
    public void testMyStarter(){
        String name = myStarterConfig.getName();
        System.out.println(name);
    }
}
复制代码
```

控制台输出：

```
myname
复制代码
```

至此，一个简单自定义的springboot starter就完成了。

# 注解解释

下面这些注解在自定义starter是可能会用到。

*   @Conditional：按照一定的条件进行判断，满足条件给容器注册bean
*   @ConditionalOnMissingBean：给定的在bean不存在时,则实例化当前Bean
*   @ConditionalOnProperty：配置文件中满足定义的属性则创建bean，否则不创建
*   @ConditionalOnBean：给定的在bean存在时,则实例化当前Bean
*   @ConditionalOnClass： 当给定的类名在类路径上存在，则实例化当前Bean
*   @ConditionalOnMissingClass ：当给定的类名在类路径上不存在，则实例化当前Bean



作者：索码理
链接：https://juejin.cn/post/7127468724046528525
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

