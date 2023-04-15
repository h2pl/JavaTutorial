# ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAB4AAAAeCAYAAAA7MK6iAAAAAXNSR0IArs4c6QAABKFJREFUSA3tVl1oFVcQnrMbrak3QUgkya1akpJYcrUtIqW1JvFBE9LiQ5v6JmJpolbMg32rVrhgoYK0QiMY6i9Y6EMaW5D+xFJaTYItIuK2Kr3+BJNwkxBj05sQY3b3nM6cs2dv9t7NT/vQJw/sndk5M/PNzJkzewGerP+pAmy+ON8lLzUJgA8ZYxYIYZmGYRnctDaWvJJAmTtfP1pvXsBCCPP8QFcCaRkZYACgDZFO4stNIcBCajEOlmmC9XpJ9bAGCaPaPmzPl32dvLSVu3BWCTQs0XQQ6g0DYgwLIoAZbBCdW/i+781o1VVlm/410mw4h06Y7bIPHNyWDyL4FHkX03Q8SrzNhZTZriieckWt7cL6MM85YcLpsi/7O9/iXFT6MswI0DmmpkSaJ0qLxFIm3+i1THHB3zmBH3PYx9CcykcLOeQVVa7QtdxTgQgEleX2AjHYfwA+2ddV77ruGoJUbhGDI09YSNXyMpUt5ylOzxgbUmtOp7NmbNt8v3arjTBfYELmLUV+M+nSawNNAUqpT3ClJWg5I3BLT+cGW/DXNGCa6tx1aakCGEigArTn4TDIPdrXXYKCZNrHLMCOEPvHBlLQ99s9eHB7EB6NTki73CVPQ2F5MSx/uRQixfmq7rK0wYD8w8E905bnPDfwoWs/rfv93NWN/ZfvwsLIU7A09gxECyISeGJkHAau98L97tuw7NXnoPyNF8FcYGLGKsOs0mN3OEyec9esGW/ZEl945dTP34wlR2FZVQWU1q0Cw8Tr7p+hgLLNL0FPxx/Q35mA8aEUrH6nCgwEl0tn7wUiZYJnNRh6DK4UH/k0lfyrsBKdPVv/AriGIQcEDQZ65LBAGe2Rzui9Ybjz7XUppz1/uKBbyVPGkN3ZAeC6hr0x7Nr38N5+EqkoOm17xpoqR9ohQF55ERSvr4Dkr3chNfC3DMzGJlNBElW8w9nsGQvhNGIzDkXzCg8cLK951xHsFBlTJspJNi3ZFIMF2AeDV3q8DNOB+YHi6QTrChDIWDBRi5U5f+ZMfJLu3ccrqxtdxk4SKH336LFxSmkqefwU5T8fhdSdQf9IVKD6aNiwI/hnmcAZ91isYMJIaCUCx9W098+LgruikeTqzqqxKPUwqJyCPJiyemVVZBOijDGjD38Os0jOiSPL1z3SPjXNANbiNPXAdzTfukjjuknNBbyz3nwgTd3AVFqUJ5hpHlq9MveLnWwttUfoygBmvVjuikxND3znrhsELnZk7k+OjIGxeNEkomyLVta0xxn+HZhjBc4YZ/AFjHjz9u3xRZl2BN4aq9nFwWh16IrQ1aHHEd3j1+4/dB9OtH4e29A2H1DyHQRmOSfQZ1Fy7MHBTGB6J/Djq6p3OxyO2cB+4Car7v/o3GXgfAkj23+x9ID1Teoamo/SXcbvSf2PX7Vc8DdCmE1vN9di+32P9/5YR3vLnhCVGUWBjEkr3yh4H8v9CzmsbdhzOKzsJKM90iFdaTMjRPhGVsakRvOaRidljo6H6G7j+ctrJpsP+4COhDIl0La2+FS4+5mlocBaXY5QnGZysIBYoeSsl5qQzrSj/cgNrfuEzlWBfwA+EjrZyWUvpAAAAABJRU5ErkJggg==)Spring 校验



Java API 规范(`JSR303`)定义了`Bean`校验的标准`validation-api`，但没有提供实现。`hibernate validation`是对这个规范的实现，并增加了校验注解如`@Email`、`@Length`等。`Spring Validation`是对`hibernate validation`的二次封装，用于支持`spring mvc`参数自动校验。

## [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8)快速入门

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E5%BC%95%E5%85%A5%E4%BE%9D%E8%B5%96)引入依赖

如果 spring-boot 版本小于 2.3.x，spring-boot-starter-web 会自动传入 hibernate-validator 依赖。如果 spring-boot 版本大于 2.3.x，则需要手动引入依赖：



```
<dependency>
  <groupId>org.hibernate.validator</groupId>
  hibernate-validator-parent
  <version>6.2.5.Final</version>
</dependency>

```



对于 web 服务来说，为防止非法参数对业务造成影响，在 Controller 层一定要做参数校验的！大部分情况下，请求参数分为如下两种形式：

*   POST、PUT 请求，使用 requestBody 传递参数；
*   GET 请求，使用 requestParam/PathVariable 传递参数。

实际上，不管是 requestBody 参数校验还是方法级别的校验，最终都是调用 Hibernate Validator 执行校验，Spring Validation 只是做了一层封装。

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E6%A0%A1%E9%AA%8C%E7%A4%BA%E4%BE%8B)校验示例

（1）在实体上标记校验注解



```
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User implements Serializable {

    @NotNull
    private Long id;

    @NotBlank
    @Size(min = 2, max = 10)
    private String name;

    @Min(value = 1)
    @Max(value = 100)
    private Integer age;

}

```



（2）在方法参数上声明校验注解



```
@Slf4j
@Validated
@RestController
@RequestMapping("validate1")
public class ValidatorController {

    /**
     * {@link RequestBody} 参数校验
     */
    @PostMapping(value = "save")
    public DataResult<Boolean> save(@Valid @RequestBody User entity) {
        log.info("保存一条记录：{}", JSONUtil.toJsonStr(entity));
        return DataResult.ok(true);
    }

    /**
     * {@link RequestParam} 参数校验
     */
    @GetMapping(value = "queryByName")
    public DataResult<User> queryByName(
        @RequestParam("username")
        @NotBlank
        @Size(min = 2, max = 10)
        String name
    ) {
        User user = new User(1L, name, 18);
        return DataResult.ok(user);
    }

    /**
     * {@link PathVariable} 参数校验
     */
    @GetMapping(value = "detail/{id}")
    public DataResult<User> detail(@PathVariable("id") @Min(1L) Long id) {
        User user = new User(id, "李四", 18);
        return DataResult.ok(user);
    }

}

```



（3）如果请求参数不满足校验规则，则会抛出?`ConstraintViolationException`?或?`MethodArgumentNotValidException`?异常。

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E7%BB%9F%E4%B8%80%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86)统一异常处理

在实际项目开发中，通常会用统一异常处理来返回一个更友好的提示。



```
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 处理所有不可知的异常
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.OK)
    @ExceptionHandler(Throwable.class)
    public Result handleException(Throwable e) {
        log.error("未知异常", e);
        return new Result(ResultStatus.HTTP_SERVER_ERROR.getCode(), e.getMessage());
    }

    /**
     * 统一处理请求参数校验异常(普通传参)
     *
     * @param e ConstraintViolationException
     * @return {@link DataResult}
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler({ ConstraintViolationException.class })
    public Result handleConstraintViolationException(final ConstraintViolationException e) {
        log.error("ConstraintViolationException", e);
        List<String> errors = new ArrayList<>();
        for (ConstraintViolation<?> violation : e.getConstraintViolations()) {
            Path path = violation.getPropertyPath();
            List<String> pathArr = StrUtil.split(path.toString(), ',');
            errors.add(pathArr.get(0) + " " + violation.getMessage());
        }
        return new Result(ResultStatus.REQUEST_ERROR.getCode(), CollectionUtil.join(errors, ","));
    }

    /**
     * 处理参数校验异常
     *
     * @param e MethodArgumentNotValidException
     * @return {@link DataResult}
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler({ MethodArgumentNotValidException.class })
    private Result handleMethodArgumentNotValidException(final MethodArgumentNotValidException e) {
        log.error("MethodArgumentNotValidException", e);
        List<String> errors = new ArrayList<>();
        for (ObjectError error : e.getBindingResult().getAllErrors()) {
            errors.add(((FieldError) error).getField() + " " + error.getDefaultMessage());
        }
        return new Result(ResultStatus.REQUEST_ERROR.getCode(), CollectionUtil.join(errors, ","));
    }

}

```



## [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E8%BF%9B%E9%98%B6%E4%BD%BF%E7%94%A8)进阶使用

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E5%88%86%E7%BB%84%E6%A0%A1%E9%AA%8C)分组校验

在实际项目中，可能多个方法需要使用同一个 DTO 类来接收参数，而不同方法的校验规则很可能是不一样的。这个时候，简单地在 DTO 类的字段上加约束注解无法解决这个问题。因此，spring-validation 支持了分组校验的功能，专门用来解决这类问题。

（1）定义分组



```
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
public @interface AddCheck { }

@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
public @interface EditCheck { }

```



（2）在实体上标记校验注解



```
@Data
public class User2 {

    @NotNull(groups = EditCheck.class)
    private Long id;

    @NotNull(groups = { AddCheck.class, EditCheck.class })
    @Size(min = 2, max = 10, groups = { AddCheck.class, EditCheck.class })
    private String name;

    @IsMobile(message = "不是有效手机号", groups = { AddCheck.class, EditCheck.class })
    private String mobile;

}

```



（3）在方法上根据不同场景进行校验分组



```
@Slf4j
@Validated
@RestController
@RequestMapping("validate2")
public class ValidatorController2 {

    /**
     * {@link RequestBody} 参数校验
     */
    @PostMapping(value = "add")
    public DataResult<Boolean> add(@Validated(AddCheck.class) @RequestBody User2 entity) {
        log.info("添加一条记录：{}", JSONUtil.toJsonStr(entity));
        return DataResult.ok(true);
    }

    /**
     * {@link RequestBody} 参数校验
     */
    @PostMapping(value = "edit")
    public DataResult<Boolean> edit(@Validated(EditCheck.class) @RequestBody User2 entity) {
        log.info("编辑一条记录：{}", JSONUtil.toJsonStr(entity));
        return DataResult.ok(true);
    }

}

```



### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E5%B5%8C%E5%A5%97%E6%A0%A1%E9%AA%8C)嵌套校验

前面的示例中，DTO 类里面的字段都是基本数据类型和 String 类型。但是实际场景中，有可能某个字段也是一个对象，这种情况先，可以使用嵌套校验。 post 比如，上面保存 User 信息的时候同时还带有 Job 信息。需要注意的是，此时 DTO 类的对应字段必须标记@Valid 注解。



```
@Data
public class UserDTO {

    @Min(value = 10000000000000000L, groups = Update.class)
    private Long userId;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 2, max = 10, groups = {Save.class, Update.class})
    private String userName;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String account;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String password;

    @NotNull(groups = {Save.class, Update.class})
    @Valid
    private Job job;

    @Data
    public static class Job {

        @Min(value = 1, groups = Update.class)
        private Long jobId;

        @NotNull(groups = {Save.class, Update.class})
        @Length(min = 2, max = 10, groups = {Save.class, Update.class})
        private String jobName;

        @NotNull(groups = {Save.class, Update.class})
        @Length(min = 2, max = 10, groups = {Save.class, Update.class})
        private String position;
    }

    /**
     * 保存的时候校验分组
     */
    public interface Save {
    }

    /**
     * 更新的时候校验分组
     */
    public interface Update {
    }
}
复制代码

```



嵌套校验可以结合分组校验一起使用。还有就是嵌套集合校验会对集合里面的每一项都进行校验，例如`List<Job>`字段会对这个 list 里面的每一个 Job 对象都进行校验

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E8%87%AA%E5%AE%9A%E4%B9%89%E6%A0%A1%E9%AA%8C%E6%B3%A8%E8%A7%A3)自定义校验注解

（1）自定义校验注解?`@IsMobile`



```
@Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE })
@Retention(RUNTIME)
@Constraint(validatedBy = MobileValidator.class)
public @interface IsMobile {

    String message();

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

}

```



（2）实现?`ConstraintValidator`?接口，编写?`@IsMobile`?校验注解的解析器



```
import cn.hutool.core.util.StrUtil;
import io.github.dunwu.spring.core.validation.annotation.IsMobile;
import io.github.dunwu.tool.util.ValidatorUtil;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

public class MobileValidator implements ConstraintValidator<IsMobile, String> {

    @Override
    public void initialize(IsMobile isMobile) { }

    @Override
    public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        if (StrUtil.isBlank(s)) {
            return false;
        } else {
            return ValidatorUtil.isMobile(s);
        }
    }

}

```



### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E8%87%AA%E5%AE%9A%E4%B9%89%E6%A0%A1%E9%AA%8C)自定义校验

可以通过实现?`org.springframework.validation.Validator`?接口来自定义校验。

有以下要点

*   实现?`supports`?方法
*   实现?`validate`?方法
    *   通过?`Errors`?对象收集错误
        *   `ObjectError`：对象（Bean）错误：
        *   `FieldError`：对象（Bean）属性（Property）错误
    *   通过?`ObjectError`?和?`FieldError`?关联?`MessageSource`?实现获取最终的错误文案



```
package io.github.dunwu.spring.core.validation;

import io.github.dunwu.spring.core.validation.annotation.Valid;
import io.github.dunwu.spring.core.validation.config.CustomValidatorConfig;
import io.github.dunwu.spring.core.validation.entity.Person;
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.ValidationUtils;
import org.springframework.validation.Validator;

import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Component
public class CustomValidator implements Validator {

    private final CustomValidatorConfig validatorConfig;

    public CustomValidator(CustomValidatorConfig validatorConfig) {
        this.validatorConfig = validatorConfig;
    }

    /**
     * 本校验器只针对 Person 对象进行校验
     */
    @Override
    public boolean supports(Class<?> clazz) {
        return Person.class.equals(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        ValidationUtils.rejectIfEmpty(errors, "name", "name.empty");

        List<Field> fields = getFields(target.getClass());
        for (Field field : fields) {
            Annotation[] annotations = field.getAnnotations();
            for (Annotation annotation : annotations) {
                if (annotation.annotationType().getAnnotation(Valid.class) != null) {
                    try {
                        ValidatorRule validatorRule = validatorConfig.findRule(annotation);
                        if (validatorRule != null) {
                            validatorRule.valid(annotation, target, field, errors);
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    private List<Field> getFields(Class<?> clazz) {
        // 声明Field数组
        List<Field> fields = new ArrayList<>();
        // 如果class类型不为空
        while (clazz != null) {
            // 添加属性到属性数组
            Collections.addAll(fields, clazz.getDeclaredFields());
            clazz = clazz.getSuperclass();
        }
        return fields;
    }

}

```



### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E5%BF%AB%E9%80%9F%E5%A4%B1%E8%B4%A5-fail-fast)快速失败(Fail Fast)

Spring Validation 默认会校验完所有字段，然后才抛出异常。可以通过一些简单的配置，开启 Fali Fast 模式，一旦校验失败就立即返回。



```
@Bean
public Validator validator() {
    ValidatorFactory validatorFactory = Validation.byProvider(HibernateValidator.class)
            .configure()
            // 快速失败模式
            .failFast(true)
            .buildValidatorFactory();
    return validatorFactory.getValidator();
}

```



## [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#spring-%E6%A0%A1%E9%AA%8C%E5%8E%9F%E7%90%86)Spring 校验原理

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#spring-%E6%A0%A1%E9%AA%8C%E4%BD%BF%E7%94%A8%E5%9C%BA%E6%99%AF)Spring 校验使用场景

*   Spring 常规校验（Validator）
*   Spring 数据绑定（DataBinder）
*   Spring Web 参数绑定（WebDataBinder）
*   Spring WebMVC/WebFlux 处理方法参数校验

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#validator-%E6%8E%A5%E5%8F%A3%E8%AE%BE%E8%AE%A1)Validator 接口设计

*   接口职责
    *   Spring 内部校验器接口，通过编程的方式校验目标对象
*   核心方法
    *   `supports(Class)`：校验目标类能否校验
    *   `validate(Object,Errors)`：校验目标对象，并将校验失败的内容输出至 Errors 对象
*   配套组件
    *   错误收集器：`org.springframework.validation.Errors`
    *   Validator 工具类：`org.springframework.validation.ValidationUtils`

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#errors-%E6%8E%A5%E5%8F%A3%E8%AE%BE%E8%AE%A1)Errors 接口设计

*   接口职责
    *   数据绑定和校验错误收集接口，与 Java Bean 和其属性有强关联性
*   核心方法
    *   `reject`?方法（重载）：收集错误文案
    *   `rejectValue`?方法（重载）：收集对象字段中的错误文案
*   配套组件
    *   Java Bean 错误描述：`org.springframework.validation.ObjectError`
    *   Java Bean 属性错误描述：`org.springframework.validation.FieldError`

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#errors-%E6%96%87%E6%A1%88%E6%9D%A5%E6%BA%90)Errors 文案来源

Errors 文案生成步骤

*   选择 Errors 实现（如：`org.springframework.validation.BeanPropertyBindingResult`）
*   调用 reject 或 rejectValue 方法
*   获取 Errors 对象中 ObjectError 或 FieldError
*   将 ObjectError 或 FieldError 中的 code 和 args，关联 MessageSource 实现（如：`ResourceBundleMessageSource`）

### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#spring-web-%E6%A0%A1%E9%AA%8C%E5%8E%9F%E7%90%86)spring web 校验原理

#### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#requestbody-%E5%8F%82%E6%95%B0%E6%A0%A1%E9%AA%8C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)RequestBody 参数校验实现原理

在 spring-mvc 中，`RequestResponseBodyMethodProcessor`?是用于解析?`@RequestBody`?标注的参数以及处理`@ResponseBody`?标注方法的返回值的。其中，执行参数校验的逻辑肯定就在解析参数的方法?`resolveArgument()`?中：



```
@Override
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
    NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

    parameter = parameter.nestedIfOptional();
    Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
    String name = Conventions.getVariableNameForParameter(parameter);

    if (binderFactory != null) {
        WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
        if (arg != null) {
            // 尝试进行参数校验
            validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                // 如果存在校验错误，则抛出 MethodArgumentNotValidException
                throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
            }
        }
        if (mavContainer != null) {
            mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
        }
    }

    return adaptArgumentIfNecessary(arg, parameter);
}

```



可以看到，resolveArgument()调用了 validateIfApplicable()进行参数校验。



```
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
    // 获取参数注解，如 @RequestBody、@Valid、@Validated
    Annotation[] annotations = parameter.getParameterAnnotations();
    for (Annotation ann : annotations) {
        // 先尝试获取 @Validated 注解
        Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
        // 如果标注了 @Validated，直接开始校验。
        // 如果没有，那么判断参数前是否有 Valid 开头的注解。
        if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
            Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
            Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
            // 执行校验
            binder.validate(validationHints);
            break;
        }
    }
}

```



以上代码，就解释了 Spring 为什么能同时支持?`@Validated`、`@Valid`?两个注解。

接下来，看一下 WebDataBinder.validate() 的实现：



```
@Override
public void validate(Object target, Errors errors, Object... validationHints) {
    if (this.targetValidator != null) {
        processConstraintViolations(
            // 此处调用 Hibernate Validator 执行真正的校验
            this.targetValidator.validate(target, asValidationGroups(validationHints)), errors);
    }
}

```



通过上面代码，可以看出 Spring 校验实际上是基于 Hibernate Validator 的封装。

#### [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E6%96%B9%E6%B3%95%E7%BA%A7%E5%88%AB%E7%9A%84%E5%8F%82%E6%95%B0%E6%A0%A1%E9%AA%8C%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)方法级别的参数校验实现原理

Spring 支持根据方法去进行拦截、校验，原理就在于应用了 AOP 技术。具体来说，是通过?`MethodValidationPostProcessor`?动态注册 AOP 切面，然后使用?`MethodValidationInterceptor`?对切点方法织入增强。



```
public class MethodValidationPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessorimplements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        // 为所有 @Validated 标注的 Bean 创建切面
        Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
        // 创建 Advisor 进行增强
        this.advisor = new DefaultPointcutAdvisor(pointcut, createMethodValidationAdvice(this.validator));
    }

    // 创建 Advice，本质就是一个方法拦截器
    protected Advice createMethodValidationAdvice(@Nullable Validator validator) {
        return (validator != null ? new MethodValidationInterceptor(validator) : new MethodValidationInterceptor());
    }
}

```



接着看一下?`MethodValidationInterceptor`：



```
public class MethodValidationInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 无需增强的方法，直接跳过
        if (isFactoryBeanMetadataMethod(invocation.getMethod())) {
            return invocation.proceed();
        }
        // 获取分组信息
        Class<?>[] groups = determineValidationGroups(invocation);
        ExecutableValidator execVal = this.validator.forExecutables();
        Method methodToValidate = invocation.getMethod();
        Set<ConstraintViolation<Object>> result;
        try {
            // 方法入参校验，最终还是委托给 Hibernate Validator 来校验
            result = execVal.validateParameters(
                invocation.getThis(), methodToValidate, invocation.getArguments(), groups);
        }
        catch (IllegalArgumentException ex) {
            ...
        }
        // 有异常直接抛出
        if (!result.isEmpty()) {
            throw new ConstraintViolationException(result);
        }
        // 真正的方法调用
        Object returnValue = invocation.proceed();
        // 对返回值做校验，最终还是委托给Hibernate Validator来校验
        result = execVal.validateReturnValue(invocation.getThis(), methodToValidate, returnValue, groups);
        // 有异常直接抛出
        if (!result.isEmpty()) {
            throw new ConstraintViolationException(result);
        }
        return returnValue;
    }
}

```



实际上，不管是 requestBody 参数校验还是方法级别的校验，最终都是调用 Hibernate Validator 执行校验，Spring Validation 只是做了一层封装。

## [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E9%97%AE%E9%A2%98)问题

**Spring 有哪些校验核心组件**？

*   检验器：`org.springframework.validation.Validator`
*   错误收集器：`org.springframework.validation.Errors`
*   Java Bean 错误描述：`org.springframework.validation.ObjectError`
*   Java Bean 属性错误描述：`org.springframework.validation.FieldError`
*   Bean Validation 适配：`org.springframework.validation.beanvalidation.LocalValidatorFactoryBean`

## [#](https://dunwu.github.io/spring-tutorial/pages/fe6aad/#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)参考资料

*   [Spring 官方文档之 Core Technologies(opens new window)](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans)
*   [《小马哥讲 Spring 核心编程思想》(opens new window)](https://time.geekbang.org/course/intro/265)
*   https://juejin.cn/post/6856541106626363399


# 参考文章
https://www.w3cschool.cn/wkspring
https://www.runoob.com/w3cnote/basic-knowledge-summary-of-spring.html
http://codepub.cn/2015/06/21/Basic-knowledge-summary-of-Spring
https://dunwu.github.io/spring-tutorial
https://mszlu.com/java/spring
http://c.biancheng.net/spring/aop-module.html