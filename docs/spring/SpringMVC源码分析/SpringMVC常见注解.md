## 1. 概述
   在本教程中，我们将探索 org.springframework.web.bind.annotation 包中的 Spring Web 注释。

## 2. @RequestMapping

简单地说，@RequestMapping 标记@Controller 类内部的请求处理程序方法； 它可以使用配置：

路径，或其别名、名称和值：方法映射到哪个 URL
方法：兼容的 HTTP 方法
params：根据 HTTP 参数的存在、不存在或值过滤请求
headers：根据 HTTP 标头的存在、不存在或值过滤请求
消耗：该方法可以在 HTTP 请求正文中消耗哪些媒体类型
produces：该方法可以在 HTTP 响应正文中生成哪些媒体类型
这是一个简单的示例：

````
@Controller
class VehicleController {

    @RequestMapping(value = "/vehicles/home", method = RequestMethod.GET)
    String home() {
        return "home";
    }
}
````
如果我们在类级别应用此注释，我们可以为 @Controller 类中的所有处理程序方法提供默认设置。 唯一的例外是 Spring 不会使用方法级别设置覆盖但会附加两个路径部分的 URL。

例如，下面的配置和上面的效果是一样的：

````
@Controller
@RequestMapping(value = "/vehicles", method = RequestMethod.GET)
class VehicleController {

    @RequestMapping("/home")
    String home() {
        return "home";
    }
}
````

此外，@GetMapping、@PostMapping、@PutMapping、@DeleteMapping 和@PatchMapping 是@RequestMapping 的不同变体，其HTTP 方法已分别设置为GET、POST、PUT、DELETE 和PATCH。

这些从 Spring 4.3 版本开始可用。

## 3 @RequestBody

让我们继续@RequestBody——它将 HTTP 请求的主体映射到一个对象：

````
@PostMapping("/save")
void saveVehicle(@RequestBody Vehicle vehicle) {
// ...
}
````
反序列化是自动的，取决于请求的内容类型。

## 4 @PathVariable
接下来说说@PathVariable。

此注释指示方法参数绑定到 URI 模板变量。 我们可以使用 @RequestMapping 注释指定 URI 模板，并使用 @PathVariable 将方法参数绑定到模板部分之一。

我们可以使用名称或其别名，即值参数来实现这一点：

````
@RequestMapping("/{id}")
Vehicle getVehicle(@PathVariable("id") long id) {
// ...
}
````
如果模板中部分的名称与方法参数的名称相匹配，我们就不必在注释中指定它：

````
@RequestMapping("/{id}")
Vehicle getVehicle(@PathVariable long id) {
// ...
}
````
此外，我们可以通过将所需的参数设置为 false 来将路径变量标记为可选：

````
@RequestMapping("/{id}")
Vehicle getVehicle(@PathVariable(required = false) long id) {
// ...
}
````
## 5. @RequestParam
   We use @RequestParam for accessing HTTP request parameters:
````
@RequestMapping
Vehicle getVehicleByParam(@RequestParam("id") long id) {
// ...
}
````
它具有与 @PathVariable 注释相同的配置选项。

除了这些设置之外，当 Spring 在请求中发现没有值或为空值时，我们可以使用 @RequestParam 指定注入值。 为此，我们必须设置 defaultValue 参数。

提供默认值隐式设置 required 为 false：
````
@RequestMapping("/buy")
Car buyCar(@RequestParam(defaultValue = "5") int seatCount) {
// ...
}
````
除了参数之外，我们还可以访问其他 HTTP 请求部分：cookie 和标头。 

我们可以分别使用注解@CookieValue 和@RequestHeader 来访问它们。


## 6. Response Handling Annotations
在接下来的部分中，我们将看到在 Spring MVC 中操作 HTTP 响应的最常见注释。

### 6.1 @ResponseBody
如果我们用@ResponseBody 标记请求处理程序方法，Spring 会将方法的结果视为响应本身：

````
@ResponseBody
@RequestMapping("/hello")
String hello() {
return "Hello World!";
}
````
如果我们用这个注解来注解 @Controller 类，所有请求处理程序方法都将使用它。

### 6.2 @ExceptionHandler

使用此注释，我们可以声明一个自定义错误处理程序方法。 当请求处理程序方法抛出任何指定的异常时，Spring 调用此方法。

捕获的异常可以作为参数传递给方法：
````
@ExceptionHandler(IllegalArgumentException.class)
void onIllegalArgumentException(IllegalArgumentException exception) {
// ...
}
````

### 6.3 @ResponseStatus
如果我们使用此注释对请求处理程序方法进行注释，则可以指定响应的所需 HTTP 状态。 我们可以使用 code 参数或其别名 value 参数来声明状态代码。

此外，我们可以使用 reason 参数提供原因。

我们也可以将它与@ExceptionHandler 一起使用：

@ExceptionHandler(IllegalArgumentException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
void onIllegalArgumentException(IllegalArgumentException exception) {
// ...
}

有关 HTTP 响应状态的更多信息，请访问本文。

## 7 其他 Web注解
一些注释不直接管理 HTTP 请求或响应。 在接下来的部分中，我们将介绍最常见的。

### 7.1 @Controller
我们可以使用@Controller 定义一个Spring MVC 控制器。 有关更多信息，请访问我们关于 Spring Bean Annotations 的文章。

### 7.2 @RestController
@RestController 结合了@Controller 和@ResponseBody。

因此，以下声明是等效的：

````
@Controller
@ResponseBody
class VehicleRestController {
// ...
}
````

````
@RestController
class VehicleRestController {
// ...
}
````
### 7.3 @ModelAttribute
通过这个注解，我们可以通过提供模型键来访问已经存在于 MVC @Controller 模型中的元素：

````
@PostMapping("/assemble")
void assembleVehicle(@ModelAttribute("vehicle") Vehicle vehicleInModel) {
// ...
}
````
与@PathVariable 和@RequestParam 一样，如果参数具有相同的名称，我们不必指定模型键：

````
@PostMapping("/assemble")
void assembleVehicle(@ModelAttribute Vehicle vehicle) {
// ...
}
````
此外，@ModelAttribute还有一个用途：如果我们用它注解一个方法，Spring会自动将方法的返回值添加到模型中：

````
@ModelAttribute("vehicle")
Vehicle getVehicle() {
// ...
}
````
和以前一样，我们不必指定模型键，Spring 默认使用方法的名称：
````
@ModelAttribute
Vehicle vehicle() {
// ...
}
````
在 Spring 调用请求处理程序方法之前，它会调用类中所有 @ModelAttribute 注释的方法。

有关 @ModelAttribute 的更多信息，请参阅本文。

### 7.4 @CrossOrigin
@CrossOrigin 为带注释的请求处理程序方法启用跨域通信：

````
@CrossOrigin
@RequestMapping("/hello")
String hello() {
return "Hello World!";
}
````
如果我们用它标记一个类，它适用于其中的所有请求处理程序方法。

我们可以使用此注释的参数微调 CORS 行为。

有关详细信息，请访问本文。


# 参考文章
https://www.baeldung.com/spring-annotations
