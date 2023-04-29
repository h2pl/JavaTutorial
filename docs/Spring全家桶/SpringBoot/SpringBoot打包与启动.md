在使用`maven`构建`springboot`项目时，`springboot`相关 jar 包可以使用`parent方式`引入（即在`pom.xml`的`parent`节点引入`springboot`的`GAV`:`org.springframework.boot:spring-boot-starter-parent:2.1.1.RELEASE`），也可以使用`非parent方式`引入（即在 pom 的 dependencyManagement 节点引入`springboot`的`GAV`:`org.springframework.boot:spring-boot-dependencies:2.1.1.RELEASE`）。同时，在打包时，我们可以打成 jar 包，也可以打成 war 包，本文旨在梳理各引入、打包方式的异同。

### 1\. parent 方式引入，打成 jar 包

parent 方式，即在 pom 文件中，将 springboot 的依赖当成项目的 parent 引入，pom 文件示例如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 引入springboot的parent -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-parent
        <version>2.1.1.RELEASE</version>
    </parent>

    <groupId>com.gitee.funcy</groupId>
    springboot-parent-jar
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>springboot parent jar打包方式</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
    </properties>

    <dependencies>
        <!-- springboot 的基础依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter
        </dependency>
        <!-- springboot 的web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- maven编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-compiler-plugin
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <!-- springboot编译插件 -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin
                <!-- 其他内容不用指定，因为 spring-boot-starter-parent 已经指定了-->
            </plugin>
        </plugins>
    </build>

</project>

```

添加一个 controller:

```
package com.gitee.funcy.mavenparent.jar.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * ｛这里添加描述｝
 *
 * @author funcy
 * @date 2019-12-13 10:43 下午
 */
@RestController
public class IndexController {

    @RequestMapping("/")
    public String helloWorld() {
        return "hello world";
    }

}

```

再引入启动类：

```
package com.gitee.funcy.mavenparent.jar;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * ｛这里添加描述｝
 *
 * @author funcy
 * @date 2019-12-13 10:36 下午
 */
@SpringBootApplication
public class Main {

    public static void main(String[] args) {
        SpringApplication.run(Main.class, args);
    }

}

```

运行 Main 方法，请求`http://localhost:8080/`，结果如下：

```
 $ curl http://localhost:8080/
hello world

```

可以看到，项目运行成功。

接着，尝试使用 jar 包启动：

```
# 打包
 mvn clean install -Dmaven.test.skip=true
 # 启动jar包
 java -jar target/springboot-parent-jar-1.0.0.jar

```

可以看到，项目启动成功，请求请求`http://localhost:8080/`，也能显示正确结果。

### 2\. 非 parent 方式引入，打成 jar 包

在实际项目中，项目的 parent 依赖可能给了其他项目，此时 parent 引用就无法进行了，这时我们需要非 parent 引入。非 parent 引入的 pom 如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.gitee.funcy</groupId>
    springboot-jar
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>springboot非parent jar打包方式</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-boot.version>2.1.1.RELEASE</spring-boot.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <!-- Import dependency management from Spring Boot -->
                <groupId>org.springframework.boot</groupId>
                spring-boot-dependencies
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-compiler-plugin
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin
                <version>${spring-boot.version}</version>
                <!-- 与parent引入相比，需要指定goals，否则会打包失败-->
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

再添加一个`ControllerIndexController.java`与启动类`Main.java`，这两个文件与上述示例相同，这里就不作展示了。

运行 Main 方法，请求`http://localhost:8080/`，结果如下：

```
 $ curl http://localhost:8080/
hello world

```

可以看到，项目运行成功。

接着，尝试使用 jar 包启动：

```
# 打包
 mvn clean install -Dmaven.test.skip=true
 # 启动jar包
 java -jar target/springboot-jar-1.0.0.jar

```

可以看到，项目启动成功，请求请求`http://localhost:8080/`，也能显示正确结果。

### 3\. parent 方式引入，打成 war 包

以上两种方式都是打成 jar，为了兼容传统的 servlet 应用，springboot 也支持打包 war 包，parent 引入打包 war 包的 pom 文件如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 引入springboot的parent -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-parent
        <version>2.1.1.RELEASE</version>
    </parent>

    <groupId>com.gitee.funcy</groupId>
    springboot-parent-war
    <version>1.0.0</version>
    <!-- 指定打包方式为war包 -->
    <packaging>war</packaging>
    <name>springboot parent war打包方式</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-test
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-compiler-plugin
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-war-plugin
                <!-- 保持与 spring-boot-dependencies 版本一致 -->
                <version>3.2.2</version>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin
            </plugin>
        </plugins>
    </build>

</project>

```

再添加一个`ControllerIndexController.java`与启动类`Main.java`，这两个文件与上述示例相同，这里就不作展示了。

除此之外，war 包方式还需要添加一个类，用以实现`SpringBootServletInitializer`，该类与启动类`Main.java`位于同一个包下，主要是用来引导 tomcat 等 servlet 容器加载 servlet，内容如下：

```
/**
 * ｛这里添加描述｝
 *
 * @author funcy
 * @date 2019-12-20 1:22 下午
 */
public class StartApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        // 注意这里要指向原先用main方法执行的Application启动类
        return builder.sources(Main.class);
    }
}

```

运行 Main 方法，请求`http://localhost:8080/`，结果如下：

```
 $ curl http://localhost:8080/
hello world

```

可以看到，项目运行成功。

接着，尝试使用 jar 包启动：

```
# 打包
 mvn clean install -Dmaven.test.skip=true
 # 启动jar包
 java -jar target/springboot-parent-war-1.0.0.jar

```

可以看到，项目启动成功，请求请求`http://localhost:8080/`，也能显示正确结果。

### 4\. 非 parent 方式引入，打成 war 包

同样地，打成 war 包时，也可使用非 parent 引入方式：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.gitee.funcy</groupId>
    springboot-war
    <version>1.0.0</version>
    <!-- 指定打包方式为war包 -->
    <packaging>war</packaging>
    <name>springboot非parent war打包方式</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-boot.version>2.1.1.RELEASE</spring-boot.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <!-- Import dependency management from Spring Boot -->
                <groupId>org.springframework.boot</groupId>
                spring-boot-dependencies
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web
        </dependency>
        <!-- 测试war包中 WEB-INF/lib-provided/ 目录内容-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-test
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-compiler-plugin
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <!-- 打成war包时，需要添加该插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-war-plugin
                <!-- 保持与 spring-boot-dependencies 版本一致 -->
                <version>3.2.2</version>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin
                <version>${spring-boot.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

再添加一个`ControllerIndexController.java`、`StartApplication.java`与启动类`Main.java`，这三个文件与上述示例相同，这里就不作展示了。

运行 Main 方法，请求`http://localhost:8080/`，结果如下：

```
 $ curl http://localhost:8080/
hello world

```

可以看到，项目运行成功。

接着，尝试使用 jar 包启动：

```
# 打包
 mvn clean install -Dmaven.test.skip=true
 # 启动jar包
 java -jar target/springboot-war-1.0.0.jar

```

可以看到，项目启动成功，请求请求`http://localhost:8080/`，也能显示正确结果。

### 5\. 总结

springboot 引入及打包方式组合下来有如下四种：

| 打包 / 引入 | parent 方式 | 非 parent 方式 |
| --- | --- | --- |
| jar | parent-jar 方式 | 非 parent-jar 方式 |
| war | parent-war 方式 | 非 parent-war 方式 |

### 1\. 开发时启动

在开发时启动 springboot 应用，指的是直接运行源码，如在开发时在 ide 中运行启动类的 main () 方法。

#### 1.1 在 ide 中执行启动类的`main()`方法

自从有了 springboot 后，web 项目就不必再放到 web 容器中运行了，直接运行项目的`main()`方法就行了：

![](https://oscimg.oschina.net/oscnet/up-040888025c22694b18d7be748b8cbb89f06.png)

启动日志如下：

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 21:11:16.365  INFO 84046 --- [           main] com.gitee.funcy.maven.jar.Main           : Starting Main on l with PID 84046 (/Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-jar/target/classes started by funcy in /Users/funcy/IdeaProjects/myproject/springboot-demo)
2020-01-07 21:11:16.368  INFO 84046 --- [           main] com.gitee.funcy.maven.jar.Main           : No active profile set, falling back to default profiles: default
2020-01-07 21:11:17.468  INFO 84046 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-07 21:11:17.497  INFO 84046 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-07 21:11:17.497  INFO 84046 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/9.0.13
2020-01-07 21:11:17.513  INFO 84046 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/funcy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2020-01-07 21:11:17.605  INFO 84046 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-07 21:11:17.605  INFO 84046 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1206 ms
2020-01-07 21:11:17.861  INFO 84046 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 21:11:18.096  INFO 84046 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-07 21:11:18.100  INFO 84046 --- [           main] com.gitee.funcy.maven.jar.Main           : Started Main in 1.988 seconds (JVM running for 2.34)
2020-01-07 21:11:32.155  INFO 84046 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-01-07 21:11:32.155  INFO 84046 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-01-07 21:11:32.223  INFO 84046 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 68 ms

```

访问`http://localhost:8080/`，结果如下：

```
$ curl http://localhost:8080
hello world

```

以上启动方式**war 与 jar 打包方式**都支持。

#### 1.2`mvn spring-boot:run`启动

这种方式也是源码启动，在命令行界面进入项目对应的源码目录下，然后执行`mvn spring-boot:run`命令：

```
springboot-parent-war $ mvn spring-boot:run
[INFO] Scanning for projects...
[INFO] 
[INFO] ---------------< com.gitee.funcy:springboot-parent-war >----------------
[INFO] Building springboot parent war打包方式 1.0.0
[INFO] --------------------------------[ war ]---------------------------------
[INFO] 
[INFO] >>> spring-boot-maven-plugin:2.1.1.RELEASE:run (default-cli) > test-compile @ springboot-parent-war >>>
[INFO] 
[INFO] --- maven-resources-plugin:3.1.0:resources (default-resources) @ springboot-parent-war ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 0 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ springboot-parent-war ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 3 source files to /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-parent-war/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.1.0:testResources (default-testResources) @ springboot-parent-war ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-parent-war/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:testCompile (default-testCompile) @ springboot-parent-war ---
[INFO] No sources to compile
[INFO] 
[INFO] <<< spring-boot-maven-plugin:2.1.1.RELEASE:run (default-cli) < test-compile @ springboot-parent-war <<<
[INFO] 
[INFO] 
[INFO] --- spring-boot-maven-plugin:2.1.1.RELEASE:run (default-cli) @ springboot-parent-war ---

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 21:40:50.577  INFO 84448 --- [           main] com.gitee.funcy.mavenparent.war.Main     : Starting Main on funcydeMacBook-Pro.local with PID 84448 (/Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-parent-war/target/classes started by funcy in /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-parent-war)
2020-01-07 21:40:50.579  INFO 84448 --- [           main] com.gitee.funcy.mavenparent.war.Main     : No active profile set, falling back to default profiles: default
2020-01-07 21:40:51.311  INFO 84448 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-07 21:40:51.336  INFO 84448 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-07 21:40:51.337  INFO 84448 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/9.0.13
2020-01-07 21:40:51.347  INFO 84448 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/funcy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2020-01-07 21:40:51.406  INFO 84448 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-07 21:40:51.406  INFO 84448 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 800 ms
2020-01-07 21:40:51.582  INFO 84448 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 21:40:51.736  INFO 84448 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-07 21:40:51.739  INFO 84448 --- [           main] com.gitee.funcy.mavenparent.war.Main     : Started Main in 1.39 seconds (JVM running for 3.943)
2020-01-07 21:41:04.068  INFO 84448 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-01-07 21:41:04.069  INFO 84448 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-01-07 21:41:04.076  INFO 84448 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 7 ms

```

可以看到，项目启动成功，请求`http://localhost:8080`，也能获得结果：

```
 $ curl http://localhost:8080
hello world

```

以上启动方式**war 与 jar 打包方式**都支持。

### 2\. jar 包启动

#### 2.1`java -jar`方式启动

对于打成`jar包`的`springboot`项目，使用`java -jar xxx.jar`命令即可启动：

```
:target $ java -jar springboot-jar-1.0.0.jar

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 21:47:47.075  INFO 85080 --- [           main] com.gitee.funcy.maven.jar.Main           : Starting Main on funcydeMacBook-Pro.local with PID 85080 (/Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-jar/target/springboot-jar-1.0.0.jar started by funcy in /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-jar/target)
2020-01-07 21:47:47.077  INFO 85080 --- [           main] com.gitee.funcy.maven.jar.Main           : No active profile set, falling back to default profiles: default
2020-01-07 21:47:48.152  INFO 85080 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-07 21:47:48.186  INFO 85080 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-07 21:47:48.186  INFO 85080 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/9.0.13
2020-01-07 21:47:48.202  INFO 85080 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/funcy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2020-01-07 21:47:48.303  INFO 85080 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-07 21:47:48.303  INFO 85080 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1177 ms
2020-01-07 21:47:48.502  INFO 85080 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 21:47:48.677  INFO 85080 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-07 21:47:48.680  INFO 85080 --- [           main] com.gitee.funcy.maven.jar.Main           : Started Main in 1.977 seconds (JVM running for 2.398)

```

访问`http://localhost:8080`，同样也能获得结果.

#### 2.2`java org.springframework.boot.loader.JarLauncher`方式启动

这种启动方式就魔幻了：好好的一个 jar，要先解压，然后直接运行里面的类，操作如下：

```
target $ unzip -d ./tmp springboot-jar-1.0.0.jar
Archive:  springboot-jar-1.0.0.jar
   creating: ./tmp/META-INF/
  inflating: ./tmp/META-INF/MANIFEST.MF  
   creating: ./tmp/org/
   creating: ./tmp/org/springframework/
   creating: ./tmp/org/springframework/boot/
··· 省略其他内容
target $ cd tmp/
tmp $ java org.springframework.boot.loader.JarLauncher

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 21:56:00.472  INFO 85431 --- [           main] com.gitee.funcy.maven.jar.Main           : Starting Main on funcydeMacBook-Pro.local with PID 85431 (/Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-jar/target/tmp/BOOT-INF/classes started by funcy in /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-jar/target/tmp)
2020-01-07 21:56:00.475  INFO 85431 --- [           main] com.gitee.funcy.maven.jar.Main           : No active profile set, falling back to default profiles: default
2020-01-07 21:56:01.589  INFO 85431 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-07 21:56:01.619  INFO 85431 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-07 21:56:01.619  INFO 85431 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/9.0.13
2020-01-07 21:56:01.634  INFO 85431 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/funcy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2020-01-07 21:56:01.722  INFO 85431 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-07 21:56:01.722  INFO 85431 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1203 ms
2020-01-07 21:56:01.931  INFO 85431 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 21:56:02.154  INFO 85431 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-07 21:56:02.157  INFO 85431 --- [           main] com.gitee.funcy.maven.jar.Main           : Started Main in 2.025 seconds (JVM running for 2.472)

```

总结下，步骤如下：

1.  进入项目`target/`目录
2.  解压`jar包`到`tmp`目录：`unzip -d ./tmp springboot-jar-1.0.0.jar`
3.  进入`tmp目录`：`cd tmp/`
4.  运行：`java org.springframework.boot.loader.JarLauncher`

访问`http://localhost:8080`，也能得到正确结果。

> 注：这种神奇的启动方式在什么情况下会使用呢？我曾经见过一些项目组，为了安全会把生产的配置文件放在服务器上，在部署项目的时候，先解压 jar 包，然后替换相应的配置文件，再运行。这种解压 jar 包、替换配置文件的方式就可以用此启动方式了。当然，这些解压、替换、启动等操作都会写进 shell 脚本里，自动化运行。

### 3\. war 包启动

#### 3.1`java -jar`方式启动

项目都打成`war包`了，还能使用`java -jar`启动？是的，`springboot`就是这么方便：

```
target $ java -jar springboot-war-1.0.0.war

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 22:11:54.284  INFO 85638 --- [           main] com.gitee.funcy.maven.war.Main           : Starting Main on funcydeMacBook-Pro.local with PID 85638 (/Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-war/target/springboot-war-1.0.0.war started by funcy in /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-war/target)
2020-01-07 22:11:54.287  INFO 85638 --- [           main] com.gitee.funcy.maven.war.Main           : No active profile set, falling back to default profiles: default
2020-01-07 22:11:55.257  INFO 85638 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-07 22:11:55.286  INFO 85638 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-07 22:11:55.287  INFO 85638 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/9.0.13
2020-01-07 22:11:55.299  INFO 85638 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/funcy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2020-01-07 22:11:55.711  INFO 85638 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-07 22:11:55.711  INFO 85638 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1379 ms
2020-01-07 22:11:55.873  INFO 85638 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 22:11:56.031  INFO 85638 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-07 22:11:56.034  INFO 85638 --- [           main] com.gitee.funcy.maven.war.Main           : Started Main in 2.066 seconds (JVM running for 2.469)
2020-01-07 22:12:01.189  INFO 85638 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-01-07 22:12:01.190  INFO 85638 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-01-07 22:12:01.195  INFO 85638 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 5 ms

```

看，项目真的跑起来了！

#### 3.2`java org.springframework.boot.loader.WarLauncher`方式启动

`springboot`的`jar包`可以解压，然后运行某个类来启动，`war包`竟然也有这种方法！`jar包`的启动类是`org.springframework.boot.loader.JarLauncher`，相应的`war包`启动类是`org.springframework.boot.loader.WarLauncher`，步骤如下：

1.  进入项目`target/`目录
2.  解压`war包`到`tmp`目录：`unzip -d ./tmp springboot-war-1.0.0.war`
3.  进入`tmp目录`：`cd tmp/`
4.  运行：`java org.springframework.boot.loader.WarLauncher`

过程如下：

```
target $ unzip -d ./tmp springboot-war-1.0.0.war
Archive:  springboot-war-1.0.0.war
   creating: ./tmp/META-INF/
  inflating: ./tmp/META-INF/MANIFEST.MF  
   creating: ./tmp/org/
   creating: ./tmp/org/springframework/
   creating: ./tmp/org/springframework/boot/
··· 省略其他
target $ cd tmp/
tmp $ java org.springframework.boot.loader.WarLauncher

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 22:17:09.637  INFO 85782 --- [           main] com.gitee.funcy.maven.war.Main           : Starting Main on funcydeMacBook-Pro.local with PID 85782 (/Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-war/target/tmp/WEB-INF/classes started by funcy in /Users/funcy/IdeaProjects/myproject/springboot-demo/springboot-maven/springboot-war/target/tmp)
2020-01-07 22:17:09.640  INFO 85782 --- [           main] com.gitee.funcy.maven.war.Main           : No active profile set, falling back to default profiles: default
2020-01-07 22:17:10.576  INFO 85782 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-01-07 22:17:10.603  INFO 85782 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-01-07 22:17:10.604  INFO 85782 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/9.0.13
2020-01-07 22:17:10.616  INFO 85782 --- [           main] o.a.catalina.core.AprLifecycleListener   : The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [/Users/funcy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.]
2020-01-07 22:17:10.725  INFO 85782 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-01-07 22:17:10.725  INFO 85782 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1046 ms
2020-01-07 22:17:10.942  INFO 85782 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 22:17:11.137  INFO 85782 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-01-07 22:17:11.140  INFO 85782 --- [           main] com.gitee.funcy.maven.war.Main           : Started Main in 1.817 seconds (JVM running for 2.183)
2020-01-07 22:17:15.024  INFO 85782 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2020-01-07 22:17:15.024  INFO 85782 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-01-07 22:17:15.029  INFO 85782 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 5 ms

```

可以看到，项目也启动成功了！

#### 3.3 传统方式启动：使用 tomcat 容器

最初的`war包`就是放在 tomcat 等容器中运行的，我们也来试试`war包`在 tomcat 容器中运行情况如何。这里说的 tomcat 容器是指在[tomcat 官网](https://www.oschina.net/action/GoToLink?url=http%3A%2F%2Ftomcat.apache.org%2F "tomcat官网")下载的容器，非`springboot`内置容器。这里我下载的是`apache-tomcat-8.5.47`，过程如下：

```
... 省略tomcat日志输出
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)

2020-01-07 22:28:23.519  INFO 85904 --- [ost-startStop-1] c.g.funcy.maven.war.StartApplication     : Starting StartApplication on funcydeMacBook-Pro.local with PID 85904 (/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/springboot-war-1.0.0/WEB-INF/classes started by funcy in /Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47)
2020-01-07 22:28:23.523  INFO 85904 --- [ost-startStop-1] c.g.funcy.maven.war.StartApplication     : No active profile set, falling back to default profiles: default
2020-01-07 22:28:24.256  INFO 85904 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 676 ms
2020-01-07 22:28:24.655  INFO 85904 --- [ost-startStop-1] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-01-07 22:28:24.920  INFO 85904 --- [ost-startStop-1] c.g.funcy.maven.war.StartApplication     : Started StartApplication in 1.86 seconds (JVM running for 3.98)
07-Jan-2020 22:28:24.974 信息 [localhost-startStop-1] org.apache.jasper.servlet.TldScanner.scanJars 至少有一个JAR被扫描用于TLD但尚未包含TLD。 为此记录器启用调试日志记录，以获取已扫描但未在其中找到TLD的完整JAR列表。 在扫描期间跳过不需要的JAR可以缩短启动时间和JSP编译时间。
07-Jan-2020 22:28:24.999 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployWAR Deployment of web application archive [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/springboot-war-1.0.0.war] has finished in [3,468] ms
07-Jan-2020 22:28:25.000 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory 把web 应用程序部署到目录 [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/docs]
07-Jan-2020 22:28:25.010 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/docs] has finished in [10] ms
07-Jan-2020 22:28:25.010 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory 把web 应用程序部署到目录 [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/manager]
07-Jan-2020 22:28:25.027 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/manager] has finished in [17] ms
07-Jan-2020 22:28:25.027 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory 把web 应用程序部署到目录 [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/examples]
07-Jan-2020 22:28:25.181 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/examples] has finished in [154] ms
07-Jan-2020 22:28:25.181 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory 把web 应用程序部署到目录 [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/ROOT]
07-Jan-2020 22:28:25.191 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/ROOT] has finished in [10] ms
07-Jan-2020 22:28:25.191 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory 把web 应用程序部署到目录 [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/host-manager]
07-Jan-2020 22:28:25.202 信息 [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory [/Users/funcy/Applications/Tomcat/apache-tomcat-8.5.47/webapps/host-manager] has finished in [11] ms
07-Jan-2020 22:28:25.206 信息 [main] org.apache.coyote.AbstractProtocol.start 开始协议处理句柄["http-nio-8080"]
07-Jan-2020 22:28:25.212 信息 [main] org.apache.coyote.AbstractProtocol.start 开始协议处理句柄["ajp-nio-8009"]
07-Jan-2020 22:28:25.213 信息 [main] org.apache.catalina.startup.Catalina.start Server startup in 3717 ms
2020-01-07 22:29:30.754  INFO 85904 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2020-01-07 22:29:30.767  INFO 85904 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Completed initialization in 12 ms

```

请求`http://localhost:8080/springboot-war-1.0.0/`，结果如下：

```
$ curl 'http://localhost:8080/springboot-war-1.0.0/'
hello world

```

可以看到，已经部署成功了。

### 4\. 总结

|  | main () 方法 | mvn 命令 | java -jar | java xxx.WarLauncher | java xxx.JarLauncher | 外置容器 |
| --- | --- | --- | --- | --- | --- | --- |
| war | 支持 | 支持 | 支持 | 支持 | 不支持 | 支持 |
| jar | 支持 | 支持 | 支持 | 不支持 | 支持 | 不支持

### 1\. maven 打包后的文件

进入`springboot-jar/target`目录，使用`tree`命令，目录结构如下：

```
 $ tree
.
├── classes
│ └── com
│     └── gitee
│         └── funcy
│             └── maven
│                 └── jar
│                     ├── Main.class
│                     └── controller
│                         └── IndexController.class
├── generated-sources
│ └── annotations
├── maven-archiver
│ └── pom.properties
├── maven-status
│ └── maven-compiler-plugin
│     └── compile
│         └── default-compile
│             ├── createdFiles.lst
│             └── inputFiles.lst
├── springboot-jar-1.0.0.jar
└── springboot-jar-1.0.0.jar.original

14 directories, 7 files

```

注意`springboot-jar-1.0.0.jar`与`springboot-jar-1.0.0.jar.original`的区别：`springboot-jar-1.0.0.jar.original`属于原始 Maven 打包 jar 文件，该文件仅包含应用本地资源，如编译后的 classes 目录下的资源文件等，未引入第三方依赖资源；而`springboot-jar-1.0.0.jar`引入了第三方依赖资源（主要为 jar 包）。

使用`unzip springboot-jar-1.0.0.jar -d tmp`解压 jar 包，内容如下：

```
 $ tree tmp/
tmp/
├── BOOT-INF
│ ├── classes
│ │ └── com
│ │     └── gitee
│ │         └── funcy
│ │             └── maven
│ │                 └── jar
│ │                     ├── Main.class
│ │                     └── controller
│ │                         └── IndexController.class
│ └── lib
│     ├── classmate-1.4.0.jar
│     ├── hibernate-validator-6.0.13.Final.jar
│     ├── jackson-annotations-2.9.0.jar
│     ├── jackson-core-2.9.7.jar
│     ├── jackson-databind-2.9.7.jar
│     ├── jackson-datatype-jdk8-2.9.7.jar
│     ├── jackson-datatype-jsr310-2.9.7.jar
│     ├── jackson-module-parameter-names-2.9.7.jar
│     ├── javax.annotation-api-1.3.2.jar
│     ├── jboss-logging-3.3.2.Final.jar
│     ├── jul-to-slf4j-1.7.25.jar
│     ├── log4j-api-2.11.1.jar
│     ├── log4j-to-slf4j-2.11.1.jar
│     ├── logback-classic-1.2.3.jar
│     ├── logback-core-1.2.3.jar
│     ├── slf4j-api-1.7.25.jar
│     ├── snakeyaml-1.23.jar
│     ├── spring-aop-5.1.3.RELEASE.jar
│     ├── spring-beans-5.1.3.RELEASE.jar
│     ├── spring-boot-2.1.1.RELEASE.jar
│     ├── spring-boot-autoconfigure-2.1.1.RELEASE.jar
│     ├── spring-boot-starter-2.1.1.RELEASE.jar
│     ├── spring-boot-starter-json-2.1.1.RELEASE.jar
│     ├── spring-boot-starter-logging-2.1.1.RELEASE.jar
│     ├── spring-boot-starter-tomcat-2.1.1.RELEASE.jar
│     ├── spring-boot-starter-web-2.1.1.RELEASE.jar
│     ├── spring-context-5.1.3.RELEASE.jar
│     ├── spring-core-5.1.3.RELEASE.jar
│     ├── spring-expression-5.1.3.RELEASE.jar
│     ├── spring-jcl-5.1.3.RELEASE.jar
│     ├── spring-web-5.1.3.RELEASE.jar
│     ├── spring-webmvc-5.1.3.RELEASE.jar
│     ├── tomcat-embed-core-9.0.13.jar
│     ├── tomcat-embed-el-9.0.13.jar
│     ├── tomcat-embed-websocket-9.0.13.jar
│     └── validation-api-2.0.1.Final.jar
├── META-INF
│ ├── MANIFEST.MF
│ └── maven
│     └── com.gitee.funcy
│         └── springboot-jar
│             ├── pom.properties
│             └── pom.xml
└── org
    └── springframework
        └── boot
            └── loader
                ├── ExecutableArchiveLauncher.class
                ├── JarLauncher.class
                ├── LaunchedURLClassLoader$UseFastConnectionExceptionsEnumeration.class
                ├── LaunchedURLClassLoader.class
                ├── Launcher.class
                ├── MainMethodRunner.class
                ├── PropertiesLauncher$1.class
                ├── PropertiesLauncher$ArchiveEntryFilter.class
                ├── PropertiesLauncher$PrefixMatchingArchiveFilter.class
                ├── PropertiesLauncher.class
                ├── WarLauncher.class
                ├── archive
                │ ├── Archive$Entry.class
                │ ├── Archive$EntryFilter.class
                │ ├── Archive.class
                │ ├── ExplodedArchive$1.class
                │ ├── ExplodedArchive$FileEntry.class
                │ ├── ExplodedArchive$FileEntryIterator$EntryComparator.class
                │ ├── ExplodedArchive$FileEntryIterator.class
                │ ├── ExplodedArchive.class
                │ ├── JarFileArchive$EntryIterator.class
                │ ├── JarFileArchive$JarFileEntry.class
                │ └── JarFileArchive.class
                ├── data
                │ ├── RandomAccessData.class
                │ ├── RandomAccessDataFile$1.class
                │ ├── RandomAccessDataFile$DataInputStream.class
                │ ├── RandomAccessDataFile$FileAccess.class
                │ └── RandomAccessDataFile.class
                ├── jar
                │ ├── AsciiBytes.class
                │ ├── Bytes.class
                │ ├── CentralDirectoryEndRecord.class
                │ ├── CentralDirectoryFileHeader.class
                │ ├── CentralDirectoryParser.class
                │ ├── CentralDirectoryVisitor.class
                │ ├── FileHeader.class
                │ ├── Handler.class
                │ ├── JarEntry.class
                │ ├── JarEntryFilter.class
                │ ├── JarFile$1.class
                │ ├── JarFile$2.class
                │ ├── JarFile$JarFileType.class
                │ ├── JarFile.class
                │ ├── JarFileEntries$1.class
                │ ├── JarFileEntries$EntryIterator.class
                │ ├── JarFileEntries.class
                │ ├── JarURLConnection$1.class
                │ ├── JarURLConnection$JarEntryName.class
                │ ├── JarURLConnection.class
                │ ├── StringSequence.class
                │ └── ZipInflaterInputStream.class
                └── util
                    └── SystemPropertyUtils.class

21 directories, 91 files

```

可以看到，文件中主要分为如下几个目录：

*   `BOOT-INF/classes`目录存放应用编译后的 class 文件；
*   `BOOT-INF/lib`目录存放应用依赖的 jar 包；
*   `META-INF/`目录存放应用依赖的 jar 包；
*   `org/`目录存放 spring boot 相关的 class 文件。

### 2.`java -jar`启动 springboot jar 包

java 官方规定，`java -jar`命令引导的具体启动类必须配置在`MANIFEST.MF`文件中，而根据`jar文件规范`，`MANIFEST.MF`文件必须存放在`/META-INF/`目录下。因此，启动类配置在 jar 包的`/META-INF/MANIFEST.MF`文件中，查看该文件，内容如下：

```
$ cat MANIFEST.MF 
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: fangchengyan
Start-Class: com.gitee.funcy.maven.jar.Main
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Version: 2.1.1.RELEASE
Created-By: Apache Maven 3.6.0
Build-Jdk: 1.8.0_222
Main-Class: org.springframework.boot.loader.JarLauncher

```

发现`Main-Class`属性指向的`Class`为`org.springframework.boot.loader.JarLauncher`，而该类存放在 jar 包的`org/springframework/boot/loader/`目录下，并且项目的引导类定义在`Start-Class`属性性中，该属性并非 java 平台标准`META-INF/MANIFEST.MF`属性。

> 注：
>
> 1.  `org.springframework.boot.loader.JarLauncher`是可执行 jar 的启动器，`org.springframework.boot.loader.WarLauncher`是可执行 war 的启动器。
>
>
> 2.  `org.springframework.boot.loader.JarLauncher`所在的 jar 文件的 Maven GAV 信息为`org.springframework.boot:spring-boot-loader:${springboot-version}`，通常情况下，这个依赖没有必要引入 springboot 项目的 pom.xml 文件。

查看`JarLauncher`源码，如下：

```
public class JarLauncher extends ExecutableArchiveLauncher {

	static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

	static final String BOOT_INF_LIB = "BOOT-INF/lib/";

	public JarLauncher() {
	}

	protected JarLauncher(Archive archive) {
		super(archive);
	}

	@Override
	protected boolean isNestedArchive(Archive.Entry entry) {
		if (entry.isDirectory()) {
			return entry.getName().equals(BOOT_INF_CLASSES);
		}
		return entry.getName().startsWith(BOOT_INF_LIB);
	}

	public static void main(String[] args) throws Exception {
		new JarLauncher().launch(args);
	}

}

```

可以发现，`BOOT-INF/classes/`与`BOOT-INF/lib/`分别使用常量`BOOT_INF_CLASSES`和`BOOT_INF_LIB`表示，并且用于`isNestedArchive(Archive.Entry)`方法判断，从该方法的实现分析，方法参数`Archive.Entry`看似为 jar 文件中的资源，比如`application.properties`。

`Archive.Entry`有两种实现，其中一种为`org.springframework.boot.loader.archive.JarFileArchive.JarFileEntry`，基于`java.util.jar.JarEntry`，表示`FAT JAR`嵌入资源，另一种为`org.springframework.boot.loader.archive.ExplodedArchive.FileEntry`，基于文件系统实现。这也说明了`JarLauncher`支持`JAR`和`文件系统`两种启动方式。

> 文件系统启动方式如下：
>
> 1.  解压 jar 包到`temp`目录：`unzip springboot-jar-1.0.0.jar -d tmp`
> 2.  进入`temp`目录，运行命令：`java org.springframework.boot.loader.JarLauncher`可以看到，项目同样能正常启动。

在`JarLauncher`作为引导类时，当执行`java -jar`命令时，`/META-INF`资源的`Main-Class`属性将调用其`main(String[])`方法，实际上调用的是`JarLauncher#launch(args)`方法，而该方法继承于基类`org.springframework.boot.loader.Launcher`，它们之间的继承关系如下：

*   `org.springframework.boot.loader.Launcher`
  *   `org.springframework.boot.loader.ExecutableArchiveLauncher`
    *   `org.springframework.boot.loader.JarLauncher`
    *   `org.springframework.boot.loader.WarLauncher`

简单来说，springboot jar 启动过程如下：

1.  `java -jar xxx.jar`运行的是`JarLauncher`
2.  `JarLauncher#main(String[])`方法会调用`Launcher#launch(String[])`方法，创建 ClassLoader () 及调用项目的`main`方法
  *   项目主类的获取实现位于`ExecutableArchiveLauncher#getMainClass()`，主要是从`/META-INF/MANIFEST.MF`获取`Start-Class`属性
  *   项目主类的 main () 方法调用位于`MainMethodRunner#run()`，使用反射方式进行调用

### 3.`java -jar`启动 springboot war 包

从上面的分析，我们得到了启动 jar 包的`org.springframework.boot.loader.JarLauncher`以及启动 war 包的`org.springframework.boot.loader.WarLauncher`，这里我们来分析下`WarLauncher`上如何工作的。

`WarLauncher`代码如下：

```
public class WarLauncher extends ExecutableArchiveLauncher {

	private static final String WEB_INF = "WEB-INF/";

	private static final String WEB_INF_CLASSES = WEB_INF + "classes/";

	private static final String WEB_INF_LIB = WEB_INF + "lib/";

	private static final String WEB_INF_LIB_PROVIDED = WEB_INF + "lib-provided/";

	public WarLauncher() {
	}

	protected WarLauncher(Archive archive) {
		super(archive);
	}

	@Override
	public boolean isNestedArchive(Archive.Entry entry) {
		if (entry.isDirectory()) {
			return entry.getName().equals(WEB_INF_CLASSES);
		}
		else {
			return entry.getName().startsWith(WEB_INF_LIB)
					|| entry.getName().startsWith(WEB_INF_LIB_PROVIDED);
		}
	}

	public static void main(String[] args) throws Exception {
		new WarLauncher().launch(args);
	}

}

```

可以看到，`WEB-INF/classes/`、`WEB-INF/lib/`、`WEB-INF/lib-provided/`均为`WarLauncher`的`Class Path`，其中`WEB-INF/classes/`、`WEB-INF/lib/`是传统的 Servlet 应用的 ClassPath 路径，而`WEB-INF/lib-provided/`属性 springboot`WarLauncher`定制实现。那么`WEB-INF/lib-provided/`究竟是干嘛的呢？看到`provided`，我们可以大胆猜想`WEB-INF/lib-provided/`存放的是`pom.xml`文件中，`scope`为`provided`的 jar。

为了验证以上猜想，修改的 pom.xml 文件如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.gitee.funcy</groupId>
    springboot-war
    <version>1.0.0</version>
    <!-- 指定打包方式为war包 -->
    <packaging>war</packaging>
    <name>springboot非parent war打包方式</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-boot.version>2.1.1.RELEASE</spring-boot.version>
        <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <!-- Import dependency management from Spring Boot -->
                <groupId>org.springframework.boot</groupId>
                spring-boot-dependencies
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web
        </dependency>
        <!-- 测试war包中 WEB-INF/lib-provided/ 目录内容-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-test
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-compiler-plugin
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <!-- 打成war包时，需要添加该插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                maven-war-plugin
                <!-- 保持与 spring-boot-dependencies 版本一致 -->
                <version>3.2.2</version>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin
                <version>${spring-boot.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

这里我们添加了 springboot 的测试 jar`org.springframework.boot:spring-boot-test`，并将其`scope`设置为`provided`. 运行 maven 打包命令`mvn clean install -Dmaven.test.skip=true`，可以看到项目能正常打包。

打包完成后，进入`target`目录，运行`java -jar springboot-war-1.0.0.war`，项目能正常启动。

接下来，我们来看看`springboot-war-1.0.0.war`有些啥。首先使用`unzip springboot-war-1.0.0.war -d tmp`命令解压，再使用`tree -h`命令查看文件结构，结果如下

```
 $ tree -h
.
├── [ 128]  META-INF
│ ├── [ 311]  MANIFEST.MF
│ └── [  96]  maven
│     └── [  96]  com.gitee.funcy
│         └── [ 128]  springboot-war
│             ├── [  95]  pom.properties
│             └── [3.3K]  pom.xml
├── [ 160]  WEB-INF
│ ├── [  96]  classes
│ │ └── [  96]  com
│ │     └── [  96]  gitee
│ │         └── [  96]  funcy
│ │             └── [  96]  maven
│ │                 └── [ 160]  war
│ │                     ├── [ 688]  Main.class
│ │                     ├── [ 891]  StartApplication.class
│ │                     └── [  96]  controller
│ │                         └── [ 646]  IndexController.class
│ ├── [1.2K]  lib
│ │ ├── [ 65K]  classmate-1.4.0.jar
│ │ ├── [1.1M]  hibernate-validator-6.0.13.Final.jar
│ │ ├── [ 65K]  jackson-annotations-2.9.0.jar
│ │ ├── [316K]  jackson-core-2.9.7.jar
│ │ ├── [1.3M]  jackson-databind-2.9.7.jar
│ │ ├── [ 33K]  jackson-datatype-jdk8-2.9.7.jar
│ │ ├── [ 98K]  jackson-datatype-jsr310-2.9.7.jar
│ │ ├── [8.4K]  jackson-module-parameter-names-2.9.7.jar
│ │ ├── [ 26K]  javax.annotation-api-1.3.2.jar
│ │ ├── [ 65K]  jboss-logging-3.3.2.Final.jar
│ │ ├── [4.5K]  jul-to-slf4j-1.7.25.jar
│ │ ├── [258K]  log4j-api-2.11.1.jar
│ │ ├── [ 17K]  log4j-to-slf4j-2.11.1.jar
│ │ ├── [284K]  logback-classic-1.2.3.jar
│ │ ├── [461K]  logback-core-1.2.3.jar
│ │ ├── [ 40K]  slf4j-api-1.7.25.jar
│ │ ├── [294K]  snakeyaml-1.23.jar
│ │ ├── [360K]  spring-aop-5.1.3.RELEASE.jar
│ │ ├── [656K]  spring-beans-5.1.3.RELEASE.jar
│ │ ├── [935K]  spring-boot-2.1.1.RELEASE.jar
│ │ ├── [1.2M]  spring-boot-autoconfigure-2.1.1.RELEASE.jar
│ │ ├── [ 413]  spring-boot-starter-2.1.1.RELEASE.jar
│ │ ├── [ 421]  spring-boot-starter-json-2.1.1.RELEASE.jar
│ │ ├── [ 423]  spring-boot-starter-logging-2.1.1.RELEASE.jar
│ │ ├── [ 422]  spring-boot-starter-tomcat-2.1.1.RELEASE.jar
│ │ ├── [ 421]  spring-boot-starter-web-2.1.1.RELEASE.jar
│ │ ├── [1.0M]  spring-context-5.1.3.RELEASE.jar
│ │ ├── [1.2M]  spring-core-5.1.3.RELEASE.jar
│ │ ├── [274K]  spring-expression-5.1.3.RELEASE.jar
│ │ ├── [ 23K]  spring-jcl-5.1.3.RELEASE.jar
│ │ ├── [1.3M]  spring-web-5.1.3.RELEASE.jar
│ │ ├── [782K]  spring-webmvc-5.1.3.RELEASE.jar
│ │ ├── [3.1M]  tomcat-embed-core-9.0.13.jar
│ │ ├── [244K]  tomcat-embed-el-9.0.13.jar
│ │ ├── [257K]  tomcat-embed-websocket-9.0.13.jar
│ │ └── [ 91K]  validation-api-2.0.1.Final.jar
│ └── [  96]  lib-provided
│     └── [194K]  spring-boot-test-2.1.1.RELEASE.jar
└── [  96]  org
    └── [  96]  springframework
        └── [  96]  boot
            └── [ 544]  loader
                ├── [3.5K]  ExecutableArchiveLauncher.class
                ├── [1.5K]  JarLauncher.class
                ├── [1.5K]  LaunchedURLClassLoader$UseFastConnectionExceptionsEnumeration.class
                ├── [5.6K]  LaunchedURLClassLoader.class
                ├── [4.6K]  Launcher.class
                ├── [1.5K]  MainMethodRunner.class
                ├── [ 266]  PropertiesLauncher$1.class
                ├── [1.4K]  PropertiesLauncher$ArchiveEntryFilter.class
                ├── [1.9K]  PropertiesLauncher$PrefixMatchingArchiveFilter.class
                ├── [ 19K]  PropertiesLauncher.class
                ├── [1.7K]  WarLauncher.class
                ├── [ 416]  archive
                │ ├── [ 302]  Archive$Entry.class
                │ ├── [ 437]  Archive$EntryFilter.class
                │ ├── [ 945]  Archive.class
                │ ├── [ 273]  ExplodedArchive$1.class
                │ ├── [1.1K]  ExplodedArchive$FileEntry.class
                │ ├── [1.5K]  ExplodedArchive$FileEntryIterator$EntryComparator.class
                │ ├── [3.7K]  ExplodedArchive$FileEntryIterator.class
                │ ├── [5.1K]  ExplodedArchive.class
                │ ├── [1.7K]  JarFileArchive$EntryIterator.class
                │ ├── [1.1K]  JarFileArchive$JarFileEntry.class
                │ └── [7.2K]  JarFileArchive.class
                ├── [ 224]  data
                │ ├── [ 485]  RandomAccessData.class
                │ ├── [ 282]  RandomAccessDataFile$1.class
                │ ├── [2.6K]  RandomAccessDataFile$DataInputStream.class
                │ ├── [3.2K]  RandomAccessDataFile$FileAccess.class
                │ └── [3.9K]  RandomAccessDataFile.class
                ├── [ 768]  jar
                │ ├── [4.9K]  AsciiBytes.class
                │ ├── [ 616]  Bytes.class
                │ ├── [3.0K]  CentralDirectoryEndRecord.class
                │ ├── [5.1K]  CentralDirectoryFileHeader.class
                │ ├── [4.5K]  CentralDirectoryParser.class
                │ ├── [ 540]  CentralDirectoryVisitor.class
                │ ├── [ 345]  FileHeader.class
                │ ├── [ 12K]  Handler.class
                │ ├── [3.5K]  JarEntry.class
                │ ├── [ 299]  JarEntryFilter.class
                │ ├── [2.0K]  JarFile$1.class
                │ ├── [1.2K]  JarFile$2.class
                │ ├── [1.3K]  JarFile$JarFileType.class
                │ ├── [ 15K]  JarFile.class
                │ ├── [1.6K]  JarFileEntries$1.class
                │ ├── [2.0K]  JarFileEntries$EntryIterator.class
                │ ├── [ 14K]  JarFileEntries.class
                │ ├── [ 702]  JarURLConnection$1.class
                │ ├── [4.2K]  JarURLConnection$JarEntryName.class
                │ ├── [9.6K]  JarURLConnection.class
                │ ├── [3.5K]  StringSequence.class
                │ └── [1.8K]  ZipInflaterInputStream.class
                └── [  96]  util
                    └── [5.1K]  SystemPropertyUtils.class

22 directories, 93 files

```

相比于`FAT JAR`的解压目录，`War`增加了`WEB-INF/lib-provided`，并且该目录仅有一个 jar 文件，即`spring-boot-test-2.1.1.RELEASE.jar`，这正是我们在 pom.xml 文件中设置的`scope`为`provided`的 jar 包。

由此可以得出结论：**`WEB-INF/lib-provided`存放的是`scope`为`provided`的 jar 包**。

我们现来看下`META-INF/MANIFEST.MF`的内容：

```
$ cat META-INF/MANIFEST.MF 
Manifest-Version: 1.0
Built-By: fangchengyan
Start-Class: com.gitee.funcy.maven.war.Main
Spring-Boot-Classes: WEB-INF/classes/
Spring-Boot-Lib: WEB-INF/lib/
Spring-Boot-Version: 2.1.1.RELEASE
Created-By: Apache Maven 3.6.0
Build-Jdk: 1.8.0_222
Main-Class: org.springframework.boot.loader.WarLauncher

```

可以看到，该文件与 jar 包中的`META-INF/MANIFEST.MF`很相似，在文件中同样定义了`Main-Class`与`Start-Class`，这也说明了该 war 可以使用`java -jar xxx.jar`和`java org.springframework.boot.loader.WarLauncher`启动，这也与我们的验证结果一致。

### 4\. tomcat 等外部容器启动 war 包

在 springboo 刚开始推广的时候，我们还是习惯于将项目打成 war 包，然后部署到 tomcat 等 web 容器中运行。那 springboot 的 war 包是如何做到既能用 java 命令启动，又能放在 tomcat 容器中启动呢？这就是之前提到的`WEB-INF/lib-provided`目录的功能了。

传统的`servlet`应用的`class path`路径仅关注`WEB-INF/classes/`和`WEB-INF/lib/`，`WEB-INF/lib-provided/`目录下的 jar 包将被`servlet`容器忽略，如`servlet api`，该 api 由`servlet`容器提供。我们在打包时，可以把`servlet`相关 jar 包的`scope`设置成`provided`，这样就完美实现了`servlet`容器启动与`java`命令启动的兼容：

*   当部署到`servlet`容器中时，`WEB-INF/lib-provided/`目录下的 jar 包就被容器忽略了（由于`servlet`容器本身就提供了`servlet`的相关 jar 包，如果不忽略，就会出现 jar 包重复引入问题）；
*   当使用`java`命令执行时，此时无`servlet`容器提供`servlet`的相关 jar 包，而`WarLauncher`在运行过程中会加载`WEB-INF/lib-provided/`目录下的 jar 包。