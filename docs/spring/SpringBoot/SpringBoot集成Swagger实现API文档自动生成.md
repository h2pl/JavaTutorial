

目录

*   [Swagger的介绍](https://www.cnblogs.com/progor/p/13297904.html#swagger%E7%9A%84%E4%BB%8B%E7%BB%8D)
    *   [优点与缺点](https://www.cnblogs.com/progor/p/13297904.html#%E4%BC%98%E7%82%B9%E4%B8%8E%E7%BC%BA%E7%82%B9)
*   [添加swagger](https://www.cnblogs.com/progor/p/13297904.html#%E6%B7%BB%E5%8A%A0swagger)
    *   [1.添加依赖包：](https://www.cnblogs.com/progor/p/13297904.html#1%E6%B7%BB%E5%8A%A0%E4%BE%9D%E8%B5%96%E5%8C%85)
    *   [2.配置Swagger:](https://www.cnblogs.com/progor/p/13297904.html#2%E9%85%8D%E7%BD%AEswagger)
    *   [3.测试](https://www.cnblogs.com/progor/p/13297904.html#3%E6%B5%8B%E8%AF%95)
*   [场景：](https://www.cnblogs.com/progor/p/13297904.html#%E5%9C%BA%E6%99%AF)
    *   [定义接口组](https://www.cnblogs.com/progor/p/13297904.html#%E5%AE%9A%E4%B9%89%E6%8E%A5%E5%8F%A3%E7%BB%84)
    *   [定义接口](https://www.cnblogs.com/progor/p/13297904.html#%E5%AE%9A%E4%B9%89%E6%8E%A5%E5%8F%A3)
    *   [定义接口请求参数](https://www.cnblogs.com/progor/p/13297904.html#%E5%AE%9A%E4%B9%89%E6%8E%A5%E5%8F%A3%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0)
        *   [场景一：请求参数是实体类。](https://www.cnblogs.com/progor/p/13297904.html#%E5%9C%BA%E6%99%AF%E4%B8%80%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0%E6%98%AF%E5%AE%9E%E4%BD%93%E7%B1%BB)
        *   [场景二：请求参数是非实体类。](https://www.cnblogs.com/progor/p/13297904.html#%E5%9C%BA%E6%99%AF%E4%BA%8C%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0%E6%98%AF%E9%9D%9E%E5%AE%9E%E4%BD%93%E7%B1%BB)
    *   [定义接口响应](https://www.cnblogs.com/progor/p/13297904.html#%E5%AE%9A%E4%B9%89%E6%8E%A5%E5%8F%A3%E5%93%8D%E5%BA%94)
        *   [响应是实体类：](https://www.cnblogs.com/progor/p/13297904.html#%E5%93%8D%E5%BA%94%E6%98%AF%E5%AE%9E%E4%BD%93%E7%B1%BB)
        *   [响应是非实体类：](https://www.cnblogs.com/progor/p/13297904.html#%E5%93%8D%E5%BA%94%E6%98%AF%E9%9D%9E%E5%AE%9E%E4%BD%93%E7%B1%BB)
*   [Swagger UI增强](https://www.cnblogs.com/progor/p/13297904.html#swagger-ui%E5%A2%9E%E5%BC%BA)
    *   [UI对比：](https://www.cnblogs.com/progor/p/13297904.html#ui%E5%AF%B9%E6%AF%94)
    *   [使用](https://www.cnblogs.com/progor/p/13297904.html#%E4%BD%BF%E7%94%A8)
    *   [优点](https://www.cnblogs.com/progor/p/13297904.html#%E4%BC%98%E7%82%B9)
*   [整合Spring Security注意](https://www.cnblogs.com/progor/p/13297904.html#%E6%95%B4%E5%90%88spring-security%E6%B3%A8%E6%84%8F)
*   [对于token的处理](https://www.cnblogs.com/progor/p/13297904.html#%E5%AF%B9%E4%BA%8Etoken%E7%9A%84%E5%A4%84%E7%90%86)
*   [Swagger的安全管理](https://www.cnblogs.com/progor/p/13297904.html#swagger%E7%9A%84%E5%AE%89%E5%85%A8%E7%AE%A1%E7%90%86)



* * *

# Swagger的介绍

?你可能尝试过写完一个接口后，自己去创建接口文档，或者修改接口后修改接口文档。多了之后，你肯定会发生一个操作，那就是忘记了修改文档或者创建文档（除非你们公司把接口文档和写接口要求得很紧密?忘记写文档就扣工资？，否则两个分离的工作总是有可能遗漏的）。而swagger就是一个在你写接口的时候自动帮你生成接口文档的东西，只要你遵循它的规范并写一些接口的说明注解即可。

## 优点与缺点

?优点：

*   自动生成文档，只需要在接口中使用注解进行标注，就能生成对应的接口文档。
*   自动更新文档，由于是动态生成的，所以如果你修改了接口，文档也会自动对应修改（如果你也更新了注解的话）。这样就不会发送我修改了接口，却忘记更新接口文档的情况。
*   支持在线调试，swagger提供了在线调用接口的功能。

?缺点：

*   不能创建测试用例，所以他暂时不能帮你处理完所有的事情。他只能提供一个简单的在线调试，如果你想存储你的测试用例，可以使用Postman或者YAPI这样支持创建测试用户的功能。
*   要遵循一些规范，它不是任意规范的。比如说，你可能会返回一个json数据，而这个数据可能是一个Map格式的，那么我们此时不能标注这个Map格式的返回数据的每个字段的说明，而如果它是一个实体类的话，我们可以通过标注类的属性来给返回字段加说明。也比如说，对于swagger，不推荐在使用GET方式提交数据的时候还使用Body，仅推荐使用query参数、header参数或者路径参数，当然了这个限制只适用于在线调试。
*   没有接口文档更新管理，虽然一个接口更新之后，可能不会关心旧版的接口信息，但你“可能”想看看旧版的接口信息，例如有些灰度更新发布的时候可能还会关心旧版的接口。那么此时只能由后端去看看有没有注释留下了，所以可以考虑接口文档大更新的时候注释旧版的，然后写下新版的。【当然这个问题可以通过导出接口文档来对比。】
*   虽然现在Java的实体类中有不少模型，po,dto,vo等，模型的区分是为了屏蔽一些多余参数，比如一个用户登录的时候只需要username,password，但查权限的时候需要连接上权限表的信息，而如果上述两个操作都是使用了User这个实体的话，在文档中就会自动生成了多余的信息，这就要求了你基于模型来创建多个实体类，比如登录的时候一个LoginForm，需要用户-权限等信息的时候才使用User类。（当然了，这个问题等你会swagger之后你就大概就会怎么规避这个问题了。）

?上面的缺点好像写的有点多，你可能会觉得swagger这个坑有点大。但其实主要是规范问题，而规范问题有时候又会提高你的代码规范性，这个就见仁见智了，你以前可能什么接口的参数都使用一个类，而现在swagger要求你分开后，某种层次上提高了你的代码规范性。

?注：以下代码示例基于Spring Boot。完整代码可以参考：[swagger-demo](https://github.com/alprogor/swagger-demo)

* * *

# 添加swagger

?这里先讲添加swagger，也就是先整合进来，至于怎么使用，下面的“场景”中再讲解。

## 1.添加依赖包：

?注意，这里的前提是已经导入了spring boot的web包。

复制代码

```
        <dependency>
            <groupId>io.springfox</groupId>
            springfox-swagger2
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            springfox-swagger-ui
            <version>2.9.2</version>
        </dependency>

```

## 2.配置Swagger:

要使用swagger，我们必须对swagger进行配置，我们需要创建一个swagger的配置类，比如可以命名为SwaggerConfig.java

复制代码

```
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration // 标明是配置类
@EnableSwagger2 //开启swagger功能
public class SwaggerConfig {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)  // DocumentationType.SWAGGER_2 固定的，代表swagger2
//                .groupName("分布式任务系统") // 如果配置多个文档的时候，那么需要配置groupName来分组标识
                .apiInfo(apiInfo()) // 用于生成API信息
                .select() // select()函数返回一个ApiSelectorBuilder实例,用来控制接口被swagger做成文档
                .apis(RequestHandlerSelectors.basePackage("com.example.controller")) // 用于指定扫描哪个包下的接口
                .paths(PathSelectors.any())// 选择所有的API,如果你想只为部分API生成文档，可以配置这里
                .build();
    }

    /**
     * 用于定义API主界面的信息，比如可以声明所有的API的总标题、描述、版本
     * @return
     */
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("XX项目API") //  可以用来自定义API的主标题
                .description("XX项目SwaggerAPI管理") // 可以用来描述整体的API
                .termsOfServiceUrl("") // 用于定义服务的域名
                .version("1.0") // 可以用来定义版本。
                .build(); //
    }
}

```

## 3.测试

运行我们的Spring Boot项目，（我默认是8080端口，如果你不一样，请注意修改后续的url），访问`http://localhost:8080/swagger-ui.html`
然后你就可以看到一个如下的界面，由于我们暂时没有配置接口数据，所以下面显示`No operations defined in spec!`

![20200711013419](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200711013419.png)

?下面我们将介绍如何定义接口，以及在swagger UI界面中的内容。

* * *

# 场景：

## 定义接口组

接口有时候应该是分组的，而且大部分都是在一个controller中的，比如用户管理相关的接口应该都在UserController中，那么不同的业务的时候，应该定义/划分不同的接口组。接口组可以使用`@Api`来划分。
比如：

复制代码

```
@Api(tags = "角色管理") //  tags：你可以当作是这个组的名字。
@RestController
public class RoleController {
}

```

和

复制代码

```
@Api(tags = "用户管理") //  tags：你可以当作是这个组的名字。
@RestController
public class UserController {
}

```

?你也可以理解成基于tags来分组，就好像一些文章里面的标签一样，使用标签来分类。
?如果这个Controller下（接口组）下面没有接口，那么在swagger ui中是不会显示的，如果有的话就会这样显示：
![20200712022545](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712022545.png)

## 定义接口

使用了`@Api`来标注一个Controller之后，如果下面有接口，那么就会默认生成文档，但没有我们自定义的说明：

复制代码

```
@Api(tags = "用户管理")
@RestController
public class UserController {
    // 注意，对于swagger，不要使用@RequestMapping，
    // 因为@RequestMapping支持任意请求方式，swagger会为这个接口生成7种请求方式的接口文档
    @GetMapping("/info") 
    public String info(String id){
        return "aaa";
    }
}

```

![20200711015840](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200711015840.png)

![20200711020149](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200711020149.png)

我们可以使用`@ApiOperation`来描述接口，比如：

复制代码

```
    @ApiOperation(value = "用户测试",notes = "用户测试notes")
    @GetMapping("/test")
    public String test(String id){
        return "test";
    }

```

![20200711021112](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200711021112.png)
常用配置项：

*   value：可以当作是接口的简称
*   notes：接口的描述
*   tags：可以额外定义接口组，比如这个接口外层已经有`@Api(tags = "用户管理")`，将接口划分到了“用户管理”中，但你可以额外的使用tags，例如`tags = "角色管理"`让角色管理中也有这个接口文档。

## 定义接口请求参数

上面使用了`@ApiOperation`来了描述接口，但其实还缺少接口请求参数的说明，下面我们分场景来讲。
?注意一下，**对于GET方式，swagger不推荐使用body方式来传递数据**，也就是不希望在GET方式时使用json、form-data等方式来传递，这时候最好使用路径参数或者url参数。(?虽然POSTMAN等是支持的)，所以如果接口传递的数据是json或者form-data方式的，还是使用POST方式好。

### 场景一：请求参数是实体类。

此时我们需要使用`@ApiModel`来标注实体类，然后在接口中定义入参为实体类即可：

*   @ApiModel：用来标类
    *   常用配置项：
        *   value：实体类简称
        *   description：实体类说明
*   @ApiModelProperty：用来描述类的字段的意义。
    *   常用配置项：
        *   value：字段说明
        *   example：设置请求示例（Example Value）的默认值，如果不配置，当字段为string的时候，此时请求示例中默认值为"".
        *   name：用新的字段名来替代旧的字段名。
        *   allowableValues：限制值得范围，例如`{1,2,3}`代表只能取这三个值；`[1,5]`代表取1到5的值；`(1,5)`代表1到5的值，不包括1和5；还可以使用infinity或-infinity来无限值，比如`[1, infinity]`代表最小值为1，最大值无穷大。
        *   required：标记字段是否必填，默认是false,
        *   hidden：用来隐藏字段，默认是false，如果要隐藏需要使用true，因为字段默认都会显示，就算没有`@ApiModelProperty`。

复制代码

```
// 先使用@ApiModel来标注类
@ApiModel(value="用户登录表单对象",description="用户登录表单对象")
public class LoginForm {
    // 使用ApiModelProperty来标注字段属性。
    @ApiModelProperty(value = "用户名",required = true,example = "root")
    private String username;
    @ApiModelProperty(value = "密码",required = true,example = "123456")
    private String password;

    // 此处省略入参赋值时需要的getter,setter,swagger也需要这个
}

```

定义成入参：

复制代码

```
    @ApiOperation(value = "登录接口",notes = "登录接口的说明")
    @PostMapping("/login")
    public LoginForm login(@RequestBody LoginForm loginForm){
        return loginForm;
    }

```

效果：

![20200711181038](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200711181038.png)

### 场景二：请求参数是非实体类。

（**再说一次：对于GET方式，swagger不推荐使用body方式来传递数据，所以虽然Spring MVC可以自动封装参数，但对于GET请求还是不要使用form-data，json等方式传递参数，除非你使用Postman来测试接口，swagger在线测试是不支持这个操作的**）
对于非实体类参数，可以使用`@ApiImplicitParams`和`@ApiImplicitParam`来声明请求参数。
`@ApiImplicitParams`用在方法头上，`@ApiImplicitParam`定义在`@ApiImplicitParams`里面，一个`@ApiImplicitParam`对应一个参数。
`@ApiImplicitParam`常用配置项：

*   name：用来定义参数的名字，也就是字段的名字,可以与接口的入参名对应。**如果不对应，也会生成，所以可以用来定义额外参数！**
*   value：用来描述参数
*   required：用来标注参数是否必填
*   paramType有path,query,body,form,header等方式，但对于对于非实体类参数的时候，常用的只有path,query,header；body和form是不常用的。body不适用于多个零散参数的情况，只适用于json对象等情况。【如果你的接口是`form-data`,`x-www-form-urlencoded`的时候可能不能使用swagger页面API调试，但可以在后面讲到基于BootstrapUI的swagger增强中调试，基于BootstrapUI的swagger支持指定`form-data`或`x-www-form-urlencoded`】

示例一：声明入参是URL参数

复制代码

```
    // 使用URL query参数
    @ApiOperation(value = "登录接口2",notes = "登录接口的说明2")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "username",//参数名字
                    value = "用户名",//参数的描述
                    required = true,//是否必须传入
                    //paramType定义参数传递类型：有path,query,body,form,header
                    paramType = "query"
                    )
            ,
            @ApiImplicitParam(name = "password",//参数名字
                    value = "密码",//参数的描述
                    required = true,//是否必须传入
                    paramType = "query"
                    )
    })
    @PostMapping(value = "/login2")
    public LoginForm login2(String username,String password){
        System.out.println(username+":"+password);
        LoginForm loginForm = new LoginForm();
        loginForm.setUsername(username);
        loginForm.setPassword(password);
        return loginForm;
    }

```

示例二：声明入参是URL路径参数

复制代码

```
    // 使用路径参数
    @PostMapping("/login3/{id1}/{id2}")
    @ApiOperation(value = "登录接口3",notes = "登录接口的说明3")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id1",//参数名字
                    value = "用户名",//参数的描述
                    required = true,//是否必须传入
                    //paramType定义参数传递类型：有path,query,body,form,header
                    paramType = "path"
            )
            ,
            @ApiImplicitParam(name = "id2",//参数名字
                    value = "密码",//参数的描述
                    required = true,//是否必须传入
                    paramType = "path"
            )
    })
    public String login3(@PathVariable Integer id1,@PathVariable Integer id2){
        return id1+":"+id2;
    }

```

示例三：声明入参是header参数

复制代码

```
    // 用header传递参数
    @PostMapping("/login4")
    @ApiOperation(value = "登录接口4",notes = "登录接口的说明4")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "username",//参数名字
                    value = "用户名",//参数的描述
                    required = true,//是否必须传入
                    //paramType定义参数传递类型：有path,query,body,form,header
                    paramType = "header"
            )
            ,
            @ApiImplicitParam(name = "password",//参数名字
                    value = "密码",//参数的描述
                    required = true,//是否必须传入
                    paramType = "header"
            )
    })
    public String login4( @RequestHeader String username,
                          @RequestHeader String password){
        return username+":"+password;
    }

```

示例四：声明文件上传参数

复制代码

```
    // 有文件上传时要用@ApiParam，用法基本与@ApiImplicitParam一样，不过@ApiParam用在参数上
    // 或者你也可以不注解，swagger会自动生成说明
    @ApiOperation(value = "上传文件",notes = "上传文件")
    @PostMapping(value = "/upload")
    public String upload(@ApiParam(value = "图片文件", required = true)MultipartFile uploadFile){
        String originalFilename = uploadFile.getOriginalFilename();

        return originalFilename;
    }

    // 多个文件上传时，**swagger只能测试单文件上传**
    @ApiOperation(value = "上传多个文件",notes = "上传多个文件")
    @PostMapping(value = "/upload2",consumes = "multipart/*", headers = "content-type=multipart/form-data")
    public String upload2(@ApiParam(value = "图片文件", required = true,allowMultiple = true)MultipartFile[] uploadFile){
        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < uploadFile.length; i++) {
            System.out.println(uploadFile[i].getOriginalFilename());
            sb.append(uploadFile[i].getOriginalFilename());
            sb.append(",");
        }
        return sb.toString();
    }

    // 既有文件，又有参数
    @ApiOperation(value = "既有文件，又有参数",notes = "既有文件，又有参数")
    @PostMapping(value = "/upload3")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "name",
                    value = "图片新名字",
                    required = true
            )
    })
    public String upload3(@ApiParam(value = "图片文件", required = true)MultipartFile uploadFile,
                          String name){
        String originalFilename = uploadFile.getOriginalFilename();

        return originalFilename+":"+name;
    }

```

## 定义接口响应

定义接口响应，是方便查看接口文档的人能够知道接口返回的数据的意义。

### 响应是实体类：

前面在定义接口请求参数的时候有提到使用`@ApiModel`来标注类，如果接口返回了这个类，那么这个类上的说明也会作为响应的说明：

复制代码

```
    // 返回被@ApiModel标注的类对象
    @ApiOperation(value = "实体类响应",notes = "返回数据为实体类的接口")
    @PostMapping("/role1")
    public LoginForm role1(@RequestBody LoginForm loginForm){
        return loginForm;
    }

```

![20200712000406](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712000406.png)

### 响应是非实体类：

swagger无法对非实体类的响应进行详细说明，只能标注响应码等信息。是通过`@ApiResponses`和`@ApiResponse`来实现的。
`@ApiResponses`和`@ApiResponse`可以与`@ApiModel`一起使用。

复制代码

```
    // 其他类型的,此时不能增加字段注释，所以其实swagger推荐使用实体类
    @ApiOperation(value = "非实体类",notes = "非实体类")
    @ApiResponses({
            @ApiResponse(code=200,message = "调用成功"),
            @ApiResponse(code=401,message = "无权限" )
    }
    )
    @PostMapping("/role2")
    public String role2(){
        return " {\n" +
                " name:\"广东\",\n" +
                "     citys:{\n" +
                "         city:[\"广州\",\"深圳\",\"珠海\"]\n" +
                "     }\n" +
                " }";
    }

```

![20200712013503](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712013503.png)

* * *

# Swagger UI增强

你可能会觉得现在这个UI不是很好看，现在有一些第三方提供了一些Swagger UI增强，比较流行的是`swagger-bootstrap-ui`，我们这里以`swagger-bootstrap-ui`为例。

## UI对比：

![20200712013653](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712013653.png)

![20200712013723](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712013723.png)

## 使用

1.添加依赖包：

复制代码

```
        <!--引入swagger-->
        <dependency>
            <groupId>io.springfox</groupId>
            springfox-swagger2
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            springfox-swagger-ui
            <version>2.9.2</version>
        </dependency>
        <!-- 引入swagger-bootstrap-ui依赖包-->
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            swagger-bootstrap-ui
            <version>1.8.7</version>
        </dependency>

```

2.在swagger配置类中增加注解`@EnableSwaggerBootstrapUI`:

复制代码

```
@Configuration // 标明是配置类
@EnableSwagger2 //开启swagger功能
@EnableSwaggerBootstrapUI // 开启SwaggerBootstrapUI
public class SwaggerConfig {
    // 省略配置内容
}

```

3.访问API：`http://localhost:8080/doc.html`，即可预览到基于bootstarp的Swagger UI界面。

## 优点

1.?界面好看了一点

2.上面说过了，基于BootstrapUI的swagger支持指定`form-data`或`x-www-form-urlencoded`：
![20200712024858](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712024858.png)

3.支持复制单个API文档和导出全部API文档：
![20200712025020](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712025020.png)

![20200712025044](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20200712025044.png)

* * *

# 整合Spring Security注意

在Spring Boot整合Spring Security和Swagger的时候，需要配置拦截的路径和放行的路径，注意是放行以下几个路径。

复制代码

```
.antMatchers("/swagger**/**").permitAll()
.antMatchers("/webjars/**").permitAll()
.antMatchers("/v2/**").permitAll()
.antMatchers("/doc.html").permitAll() // 如果你用了bootstarp的Swagger UI界面，加一个这个。

```

* * *

# 对于token的处理

在swagger中只支持了简单的调试，但对于一些接口，我们测试的时候可能需要把token信息写到header中，目前好像没看到可以自定义加请求头的地方？
?方法一：
　　如果你使用了Swagger BootstrapUI，那么你可以在“文档管理”中增加全局参数，这包括了添加header参数。

?方法二：在swagger配置类中增加全局参数配置：

复制代码

```
        //如果有额外的全局参数，比如说请求头参数，可以这样添加
        ParameterBuilder parameterBuilder = new ParameterBuilder();
        List<Parameter> parameters = new ArrayList<Parameter>();
        parameterBuilder.name("authorization").description("令牌")
                .modelRef(new ModelRef("string")).parameterType("header").required(false).build();
        parameters.add(parameterBuilder.build());
        return new Docket(DocumentationType.SWAGGER_2)  // DocumentationType.SWAGGER_2 固定的，代表swagger2
                .apiInfo(apiInfo()) // 用于生成API信息
                .select() // select()函数返回一个ApiSelectorBuilder实例,用来控制接口被swagger做成文档
                .apis(RequestHandlerSelectors.basePackage("com.example.controller")) // 用于指定扫描哪个包下的接口
                .paths(PathSelectors.any())// 选择所有的API,如果你想只为部分API生成文档，可以配置这里
                .build().globalOperationParameters(parameters);

```

?方法三：使用`@ApiImplicitParams`来额外标注一个请求头参数，例如：

复制代码

```
    // 如果需要额外的参数，非本方法用到，但过滤器要用,类似于权限token
    @PostMapping("/login6")
    @ApiOperation(value = "带token的接口",notes = "带token的接口")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "authorization",//参数名字
                    value = "授权token",//参数的描述
                    required = true,//是否必须传入
                    paramType = "header"
            )
            ,
            @ApiImplicitParam(name = "username",//参数名字
                    value = "用户名",//参数的描述
                    required = true,//是否必须传入
                    paramType = "query"
            )
    })
    public String login6(String username){
        return username;
    }

```

* * *

# Swagger的安全管理

1.如果你整合了权限管理，可以给swagger加上权限管理，要求访问swagger页面输入用户名和密码，这些是spring security和shiro的事了，这里不讲。

2.如果你仅仅是不想在正式环境中可以访问，可以在正式环境中关闭Swagger自动配置，这就不会有swagger页面了。使用`@Profile({"dev","test"})`注解来限制只在dev或者test下启用Swagger自动配置。
然后在Spring Boot配置文件中修改当前profile`spring.profiles.active=release`，重启之后，此时无法访问`http://localhost:8080/swagger-ui.html`

* * *



作者：[progor](https://www.cnblogs.com/progor/)
本文为作者原创，转载请注明出处

