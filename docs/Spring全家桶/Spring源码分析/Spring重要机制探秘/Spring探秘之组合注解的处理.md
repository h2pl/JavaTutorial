### 1\. 什么是组合注解？

在 spring 中，有一类特别的注解：组合注解。举例来说，springmvc 中，`@Controller` 注解用来配置访问路径等，`@ResponseBody` 注解用来表明不做视图渲染，直接展示方法的运行结果（一般是转成 json 返回），而 `@RestController` 组合了两者的功能，可以配置访问路径，同时也可以直接展示方法的运行结果，代码如下：

```
@Controller
@ResponseBody
public @interface RestController {
    /**
     * 注解别名
     */
    @AliasFor(annotation = Controller.class)
    String value() default "";

}

```

可以看到，`@RestController` 上标记了两个注解：`@Controller` 与 `@ResponseBody`，这样它就同时拥有了两者的功能。

再来看一个例子，spring 中，我们在标识一个类为 spring bean 的时候，可以用到这些注解：`@Component`、`@Repository`、`@Service` 等，再进一步看其代码，发现 `@Repository`、`@Service` 中都有 `@Component`：

```
@Component
public @interface Repository {
    @AliasFor(annotation = Component.class)
    String value() default "";
}

@Component
public @interface Service {
    @AliasFor(annotation = Component.class)
    String value() default "";
}

```

也就是说，`@Repository`、`@Service` 都组合了 `@Component` 的功能！

实际上，如果我们自己写一个注解，像这样：

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface MyComponent {

    @AliasFor(annotation = Component.class)
    String value() default "";

}

```

然后这样使用：

```
@MyComponent("beanObj3")
public class BeanObj3 {
    ...
}

```

spring 依然会把 `BeanObj3` 初始化为 spring bean。

那么 spring 是如何做到这一步的呢？实际上，spring 在处理 `@MyComponent` 时，会判断该注解中是否包含 `@Component` 注解，如果包含，就获取该注解的配置，然后按 `@Component` 的处理逻辑来进行处理。

同样地，spring 在处理 `@RestController` 时，如果当前是处理 `@Controller` 的逻辑，就从 `@RestController` 中获取 `@Controller` 的配置然后进行处理，如果当前是处理 `@ResponseBody` 逻辑，就从 `@RestController` 中获取 `@ResponseBody` 的配置然后进行处理。

### 2\. 递归获取指定类的所有注解

问题又来了：组合注解中的注解要怎么获取呢？

如果按照 jdk 提供的方法，像这样：

```
RestController annotation = BeanObj3.class.getAnnotation(MyComponent.class);

```

得到的 `annotation` 必定为 `null`，原因是 `Class#getAnnotation` 方法只能获取到类上直接出现的注解，`BeanObj3` 是没有直接出现 `@Component` 的，因此得到的结果为 null，办法也许你也想到了，就是继续往下读取 "注解的注解"，用代码示意下，类似这样：

```
public class AnnotationHandler {

    /**
     * 存放jdk提供的元注解
     */
    private static Set<Class<?>> metaAnnotations = new HashSet<>();
    static {
        metaAnnotations.add(Target.class);
        metaAnnotations.add(Documented.class);
        metaAnnotations.add(Retention.class);
    }

    public static void main(String[] args) {
        List<Class<?>> list = getAnnotations(BeanObj3.class);
        System.out.println(list);
    }

    /**
     * 获取操作，递归调用
     */
    public static List<Class<?>> getAnnotations(Class<?> cls) {
        // 用来存放该类上的所有注解，包括注解的注解
        List<Class<?>> list = new ArrayList<>();
        // 调用 doGetAnnotations(...) 获取
        doGetAnnotations(list, cls);
        return list;
    }

    /**
     * 获取注解的具体操作
     */
    private static void doGetAnnotations(List<Class<?>> list, Class<?> cls) {
        // 获取所有的注解
        Annotation[] annotations = cls.getAnnotations();
        if(annotations != null && annotations.length > 0) {
            for(Annotation annotation : annotations) {
                // 获取注解的类型
                Class<?> annotationType = annotation.annotationType();
                // 过滤jdk提供的元注解
                if(metaAnnotations.contains(annotationType)) {
                    continue;
                }
                // 递归调用
                doGetAnnotations(list, annotationType);
            }
        }
        // 如果是注解，就添加到 list 中
        if(cls.isAnnotation()) {
            list.add(cls);
        }
    }
}

```

我们要获取 `BeanObj3` 上所有注解，就可以这样操作了：

```
// 得到 BeanObj3 上的所有注解，包括“注解的注解”
List<Class<?>> list = AnnotationHandler.getAnnotations(BeanObj3.class);
// 判断 BeanObj3 的注解中是否包含 @Component
list.contains(Component.class);

```

以上 demo 还是比较粗糙，首先是 jdk 的元注解，这里只排除了三个，这三个都是在 `@Component` 中出现的，处理 `@Component` 之上的注解读取已经足够了；其次也是最重要的，就是没有获取注解的数据。在 spring 中，注解并不只是一个标记，还可以定义一系列数量，像这样：

```
// 定义 spring bean 的名称为 beanObj3
@MyComponent("beanObj3")
public class BeanObj3 {
    ...
}

```

而 `AnnotationHandler` 并不能获取到注解的数据！

接下来我们来看看 spring 是怎么做到注解数据的读取的。

### 3\. spring 读取注解信息

spring 5.2 中，对于注解信息的读取有提供了三个类：

*   `AnnotationMetadataReadingVisitor`：注解数据的读取类，基于 asm 实现，不过在 spring5.2 中已经废弃（标记了 `@Deprecated`），建议使用 `SimpleAnnotationMetadataReadingVisitor`，因此本文不作分析
*   `SimpleAnnotationMetadataReadingVisitor`：注解数据的读取类，基于 asm 实现，spring 5.2 中新增的类，用于替代 `AnnotationMetadataReadingVisitor`，需要注解的是，`SimpleAnnotationMetadataReadingVisitor` 的访问级别是默认的，无法在所在包之外访问，同时它也是 `final` 的，不能被继承，因此我们无法直接操作它，不过 spring 提供了一个类：`SimpleMetadataReaderFactory`，通过它就可以使用 `SimpleAnnotationMetadataReadingVisitor` 了
*   `StandardAnnotationMetadata`：注解数据的读取类，基于反射实现

#### 3.1 `SimpleAnnotationMetadataReadingVisitor`

spring 并没有提供直接操作 `SimpleAnnotationMetadataReadingVisitor` 的机会，而是封装到 `SimpleMetadataReaderFactory` 了，我们先来看看这个类：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-f2926fbc022517409bfb6f6641c87a193c2.png)

可以看到，`SimpleMetadataReaderFactory` 的类主要分为两部分：

1.  构造方法
2.  资源的获取

这里我们直接看获取的获取，也就是 `getMetadataReader(...)` 方法：

`getMetadataReader(Resource resource)`: 根据 `Resource` 读取数据 `getMetadataReader(String className)`: 根据类名读取数据，传入的是全限定类名（即 “包名。类名”），从代码来看，这个类名最终也会转化为 `Resource`，然后调用 `getMetadataReader(Resource)` 进行读取

这两个方法的返回值都是 `MetadataReader`，这是个啥呢？我们继续往下看.

##### `MetadataReader`

`MetadataReader` 的部分方法如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-4a1184918664e8a4bd363d6399ed4092692.png)

可以看到，它是个接口（这里返回的具体类型就是 `SimpleMetadataReader` 了），里面就 3 个方法：

*   `getResource()`: 获取资源
*   `getClassMetadata()`: 获取类的元数据
*   `getAnnotationMetadata()`: 获取注解的元数据

由于是获取注解的信息，这里我们只关注 `getAnnotationMetadata()` 方法：

```
AnnotationMetadata getAnnotationMetadata();

```

这个方法返回的是 `AnnotationMetadata`，这又是个啥？

##### `AnnotationMetadata`

先来看看它的方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fc19d57bbba3d70da04fd32e6eef511ad4f.png)

这些方法分为两类：

*   `getXxx(...)`：根据注解获取对应的信息
*   `hasXxx(...)`：判断是否包含某注解

如果进一步看这几个方法的默认实现，发现都调用 `getAnnotations()` 方法：

```
public interface AnnotationMetadata extends ClassMetadata, AnnotatedTypeMetadata {

    default Set<String> getAnnotationTypes() {
        // 调用了 getAnnotations()
    return getAnnotations().stream()
        .filter(MergedAnnotation::isDirectlyPresent)
        .map(annotation -> annotation.getType().getName())
        .collect(Collectors.toCollection(LinkedHashSet::new));
    }

    default Set<String> getMetaAnnotationTypes(String annotationName) {
        // 调用了 getAnnotations()
    MergedAnnotation<?> annotation = getAnnotations().get(annotationName, 
        MergedAnnotation::isDirectlyPresent);
    if (!annotation.isPresent()) {
        return Collections.emptySet();
    }
    return MergedAnnotations.from(annotation.getType(), SearchStrategy.INHERITED_ANNOTATIONS)
                .stream()
        .map(mergedAnnotation -> mergedAnnotation.getType().getName())
        .collect(Collectors.toCollection(LinkedHashSet::new));
    }

    default boolean hasAnnotation(String annotationName) {
        // 调用了 getAnnotations()
    return getAnnotations().isDirectlyPresent(annotationName);
    }

    ...
}

```

再进一步查看 `getAnnotations()` 方法，进入了 `AnnotatedTypeMetadata`：

```
public interface AnnotatedTypeMetadata {

    /**
     * 获取注解
     */
    MergedAnnotations getAnnotations();

    ...

}

```

这一样看，似乎注解得到的终极类就是 `MergedAnnotations` 了？我们继续探索。

##### `MergedAnnotations`

`MergedAnnotations` 的部分注释如下：

> Provides access to a collection of merged annotations, usually obtained from a source such as a {@link Class} or {@link Method}.
>
> 提供对组合注解的集合的访问，这些注解通常是从 Class 或 Method 之类的来源获得的。

看来，`MergedAnnotations` 才是最终的组合注解的集合了，我们来看看它的几个方法：

```
// 判断注解是否存在，会从所有的注解中判断
<A extends Annotation> boolean isPresent(Class<A> annotationType);

// 判断注解是否存在，会从所有的注解中判断，与上面的方法不同的是，这里传入的是字符串
boolean isPresent(String annotationType);

// 判断直接注解是否存在，也就是只判断当前类上有没有该注解，不判断注解的注解
<A extends Annotation> boolean isDirectlyPresent(Class<A> annotationType);

// 功能同上，这里传入的类型是字符串，格式为"包名.类名"
boolean isDirectlyPresent(String annotationType);

// 获取注解
<A extends Annotation> MergedAnnotation<A> get(Class<A> annotationType);

// 获取注解，这里传入的类型是字符串，格式为"包名.类名"
<A extends Annotation> MergedAnnotation<A> get(String annotationType);

```

从方法上大致可以看出，`MergedAnnotations` 是组合注解的集合，提供的注解可以判断某注解是否存在，也可以获取其中的某个注解。

##### `MergedAnnotation`

`MergedAnnotations` 是注解的集合，那这个集合中放的是啥呢？从它的 `get(...)` 方法来看，它存放的是 `MergedAnnotation`，我们再来看看 `MergedAnnotation` 支持的方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a3a749f4546d63b49bfd25d2fc2c73dcb6f.png)

从以上的方法可以看到，`MergedAnnotation` 就是注解的数据抽象，它提供了丰富的 api 用来获取注解的数据。

##### 使用示例

下面来看个示例：

```
// 得到 SimpleMetadataReaderFactory 实例，最终调用的是 SimpleAnnotationMetadataReadingVisitor 来读取
SimpleMetadataReaderFactory readerFactory = new SimpleMetadataReaderFactory();
MetadataReader metadataReader = readerFactory.getMetadataReader(BeanObj3.class.getName());
AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();

// AnnotationMetadata 提供了许多的操作，重点关注注解相关的
Set<String> annotationTypes = annotationMetadata.getAnnotationTypes();
System.out.println("-------------");
annotationTypes.forEach(type -> System.out.println(type));
System.out.println("-------------");

// 这里是直接获取，BeanObj3 上直接标记 @MyComponent的，返回的是true
boolean exist1 = annotationMetadata.hasAnnotation(MyComponent.class.getName());
System.out.println("hasAnnotation @MyComponent:" + exist1);

// 这里是直接获取，BeanObj3 上是没有直接标记 @Component的，返回的是false
boolean exist2 = annotationMetadata.hasAnnotation(Component.class.getName());
System.out.println("hasAnnotation @Component:" + exist2);

// 获取 MergedAnnotations
MergedAnnotations annotations = annotationMetadata.getAnnotations();
System.out.println("-------------");
annotations.forEach(annotationMergedAnnotation -> System.out.println(annotationMergedAnnotation));
System.out.println("-------------");

// 这里是直接获取，BeanObj3 上是没有直接标记 @Component的，返回的是false
boolean directlyPresent = annotations.isDirectlyPresent(Component.class);
System.out.println("directlyPresent Component:" + directlyPresent);

// 判断有没有这个注解，BeanObj3 上的@MyComponent中，标记了 @Component 的，返回的是true
boolean present = annotations.isPresent(Component.class);
System.out.println("present Component:" + present);

// 获取 @Component 注解
MergedAnnotation<Component> mergedAnnotation = annotations.get(Component.class);
// 由于 @MyComponent 的 value() 加了 @AliasFor(annotation = Component.class)
// 因此这里得到的 value 是 beanObj3 （BeanObj3里这么指定的：@MyComponent("beanObj3")）
String value = mergedAnnotation.getString("value");
System.out.println("Component value:" + value);

// 将 @Component 的注解的数据转换为 AnnotationAttributes
AnnotationAttributes annotationAttributes = mergedAnnotation.asAnnotationAttributes();
System.out.println(annotationAttributes);

```

运行，结果如下：

```
-------------
org.springframework.learn.explore.demo01.MyComponent
-------------
hasAnnotation @MyComponent:true
hasAnnotation @Component:false
-------------
@org.springframework.learn.explore.demo01.MyComponent(value=beanObj3)
@org.springframework.stereotype.Component(value=beanObj3)
@org.springframework.stereotype.Indexed()
-------------
directlyPresent Component:false
present Component:true
Component value:beanObj3
{value=beanObj3}

```

##### 补充：`AnnotationAttributes`

补充说明下 `AnnotationAttributes`：

```
public class AnnotationAttributes extends LinkedHashMap<String, Object> {
    ...
}

```

它实现了 `LinkedHashMap`，提供的部分方法如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a8c8dc975790719c37a8e41a4b0067c517c.png)

从这里不难看出，`AnnotationAttributes` 就是包含注解所有属性值的 map，key 为属性名，value 为属性值。

#### 3.2 `StandardAnnotationMetadata`

我们接着来看看 `StandardAnnotationMetadata`:

```
public class StandardAnnotationMetadata extends StandardClassMetadata 
        implements AnnotationMetadata {

    /**
     * Create a new {@code StandardAnnotationMetadata} wrapper for the given Class.
     * @param introspectedClass the Class to introspect
     * @see #StandardAnnotationMetadata(Class, boolean)
     * @deprecated since 5.2 in favor of the factory method 
     *   {@link AnnotationMetadata#introspect(Class)}
     */
    @Deprecated
    public StandardAnnotationMetadata(Class<?> introspectedClass) {
    this(introspectedClass, false);
    }

    ...
}

```

`StandardAnnotationMetadata` 实现了 `AnnotationMetadata` 接口，对于注解的操作与上面介绍的 `AnnotationMetadata` 并无太大区别，这里就不赘述了。

从 `StandardAnnotationMetadata` 的构造方法来看，它已经废弃了，让我们使用 `AnnotationMetadata#introspect(Class)` 来获取 `StandardAnnotationMetadata` 的实例，于是，我们可以像这样来操作：

```
// 获取到的 annotationMetadata 实际上是 StandardAnnotationMetadata
AnnotationMetadata annotationMetadata = AnnotationMetadata.introspect(BeanObj3.class);

//----------- 以下内容与SimpleAnnotationMetadataReadingVisitor 一模一样

// AnnotationMetadata 提供了许多的操作，重点关注注解相关的
Set<String> annotationTypes = annotationMetadata.getAnnotationTypes();
System.out.println("-------------");
annotationTypes.forEach(type -> System.out.println(type));
System.out.println("-------------");

// 这里是直接获取，BeanObj3 上直接标记 @MyComponent的，返回的是true
boolean exist1 = annotationMetadata.hasAnnotation(MyComponent.class.getName());
System.out.println("hasAnnotation @MyComponent:" + exist1);

// 这里是直接获取，BeanObj3 上是没有直接标记 @Component的，返回的是false
boolean exist2 = annotationMetadata.hasAnnotation(Component.class.getName());
System.out.println("hasAnnotation @Component:" + exist2);

// 获取 MergedAnnotations
MergedAnnotations annotations = annotationMetadata.getAnnotations();
System.out.println("-------------");
annotations.forEach(annotationMergedAnnotation -> System.out.println(annotationMergedAnnotation));
System.out.println("-------------");

// 这里是直接获取，BeanObj3 上是没有直接标记 @Component的，返回的是false
boolean directlyPresent = annotations.isDirectlyPresent(Component.class);
System.out.println("directlyPresent Component:" + directlyPresent);

// 判断有没有这个注解，BeanObj3 上的@MyComponent中，标记了 @Component 的，返回的是true
boolean present = annotations.isPresent(Component.class);
System.out.println("present Component:" + present);

// 获取 @Component 注解
MergedAnnotation<Component> mergedAnnotation = annotations.get(Component.class);
// 由于 @MyComponent 的 value() 加了 @AliasFor(annotation = Component.class)
// 因此这里得到的 value 是 beanObj3 （BeanObj3里这么指定的：@MyComponent("beanObj3")）
String value = mergedAnnotation.getString("value");
System.out.println("Component value:" + value);

// 将 @Component 的注解的数据转换为 AnnotationAttributes
AnnotationAttributes annotationAttributes = mergedAnnotation.asAnnotationAttributes();
System.out.println(annotationAttributes);

```

运行结果如下：

```
-------------
org.springframework.learn.explore.demo01.MyComponent
-------------
hasAnnotation @MyComponent:true
hasAnnotation @Component:false
-------------
@org.springframework.learn.explore.demo01.MyComponent(value=beanObj3)
@org.springframework.stereotype.Component(value=beanObj3)
@org.springframework.stereotype.Indexed()
-------------
directlyPresent Component:false
present Component:true
Component value:beanObj3
{value=beanObj3}

```

从上面的示例来看，我们历经千辛万苦，最终得到了 `MergedAnnotations`，然后通过它来判断注解是否存在、获取注解的值。

#### 3.3 两者的使用场景

`SimpleAnnotationMetadataReadingVisitor` 与 `StandardAnnotationMetadata` 的主要区别在于，`SimpleAnnotationMetadataReadingVisitor` 是基于 asm 的实现，`StandardAnnotationMetadata` 是基于反射的实现，那我们在使用时，应该要怎么选呢？

由于基于反射是要先加类加载到 jvm 中的，因此我的判断是，**如果当前类没有加载到 jvm 中，就使用 `SimpleAnnotationMetadataReadingVisitor`，如果类已经加载到 jvm 中了，两者皆可使用**。

事实上，在 spring 包扫描阶段，读取类上的注解时，使用的都是 `SimpleAnnotationMetadataReadingVisitor`，因为此时类并没有加载到 jvm，如果使用 `StandardAnnotationMetadata` 读取，就会导致类提前加载。类提前加载有什么问题呢？java 类是按需加载的，有的类可能在整个 jvm 生命周期内都没用到，如果全都加载了，就白白浪费内存了。

### 4\. spring 提供的注解工具类

在前面的示例中，我们是这样读取注解的：

```
// 读取 annotationMetadata，也可以使用 SimpleMetadataReaderFactory 读取
AnnotationMetadata annotationMetadata = AnnotationMetadata.introspect(BeanObj3.class);
MergedAnnotations annotations = annotationMetadata.getAnnotations();

// 判断注解是否存在
boolean present = annotations.isPresent(Component.class);
// 获取注解的属性
MergedAnnotation<Component> mergedAnnotation = annotations.get(Component.class);
AnnotationAttributes annotationAttributes = mergedAnnotation.asAnnotationAttributes();

```

相对来说，获取注解的属性步骤比较多，聪明如你，就想到可以将这些步骤封装到一个方法中进行处理，spring 也是这么做的，这就得介绍 spring 中与注解相关的两个类：`AnnotationUtils` 与 `AnnotatedElementUtils`。`AnnotationUtils` 是直接获取注解的值，不会处理属性覆盖，而 `AnnotatedElementUtils` 会处理属性覆盖。

什么是属性覆盖呢？

举例来说，`@MyComponent` 长这样：

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 注意Component指定的值：123
@Component("123")
public @interface MyComponent {

    @AliasFor(annotation = Component.class)
    String value() default "";

}

```

在 `@MyComponent` 注解中，我们指定了 `@Component` 的 `value` 值为 “123”，然后又这么指定 `@MyComponent` 的 `value` 值：

```
@MyComponent("beanObj3")
public class BeanObj3 {

    ...
}    

```

最终 spring 初始化得到的 `BeanObj3` 的名称是 `123` 还是 `beanObj3` 呢？从我们设置 `@MyComponent` 的 `value` 为 `beanObj3` 来说，当然是希望 bean 的名称为 `beanObj3`，而最终 spring 也是这么做的，这就是属性覆盖了：`@MyComponent` 的 `value` 覆盖了 `@Component` 的 `value` 值。

`AnnotationUtils`/`AnnotatedElementUtils` 与上面介绍的 `SimpleAnnotationMetadataReadingVisitor`/`StandardAnnotationMetadata` 是何关系呢？

在我们使用 `SimpleAnnotationMetadataReadingVisitor`/`StandardAnnotationMetadata` 时，我们需要得到 `MergedAnnotations` 再进行一系列操作（判断注解是否存在、获取注解的属性值等），如果进入 `AnnotationUtils`/`AnnotatedElementUtils` 的源码，就会发现它们的相关方法也是操作 `MergedAnnotations` 类，比如获取注解：

`AnnotationUtils#getAnnotation(AnnotatedElement, Class<A>)` 方法：

```
public static <A extends Annotation> A getAnnotation(AnnotatedElement annotatedElement, 
        Class<A> annotationType) {
    if (AnnotationFilter.PLAIN.matches(annotationType) ||
            AnnotationsScanner.hasPlainJavaAnnotationsOnly(annotatedElement)) {
        return annotatedElement.getAnnotation(annotationType);
    }
    // 通过操作 MergedAnnotations 进行获取
    return MergedAnnotations.from(annotatedElement, 
            SearchStrategy.INHERITED_ANNOTATIONS, RepeatableContainers.none())
            .get(annotationType).withNonMergedAttributes()
            .synthesize(AnnotationUtils::isSingleLevelPresent).orElse(null);
}

```

`AnnotatedElementUtils#getAllMergedAnnotations(AnnotatedElement, Class<A>)` 方法：

```
public static <A extends Annotation> Set<A> getAllMergedAnnotations(
        AnnotatedElement element, Class<A> annotationType) {
    return getAnnotations(element).stream(annotationType)
            .collect(MergedAnnotationCollectors.toAnnotationSet());
}

// AnnotatedElementUtils#getAnnotations 方法，也是操作 MergedAnnotations 的方法
private static MergedAnnotations getAnnotations(AnnotatedElement element) {
    return MergedAnnotations.from(element, SearchStrategy.INHERITED_ANNOTATIONS, 
            RepeatableContainers.none());
}

```

因此，`AnnotationUtils`/`AnnotatedElementUtils` 与 `SimpleAnnotationMetadataReadingVisitor`/`StandardAnnotationMetadata` 底层都是操作 `MergedAnnotations` 类的。

#### 4.1 `AnnotationUtils`

`AnnotationUtils` 支持的部分方法如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-51b0f496180a16d947b8f83285370cabd8f.png)

我们来实际使用下这些方法：

```
// 在 BeanObj3 获取 @Component
Annotation annotation = AnnotationUtils.getAnnotation(BeanObj3.class, Component.class);
if(null == annotation) {
    System.out.println("注解不存在！");
    return;
}
System.out.println("annotation: " + annotation);

// 获取 AnnotationAttributes
AnnotationAttributes annotationAttributes
        = AnnotationUtils.getAnnotationAttributes(BeanObj3.class, annotation);
System.out.println("AnnotationAttributes: " + annotationAttributes);

// 获取 annotationAttributeMap
Map<String, Object> annotationAttributeMap = AnnotationUtils.getAnnotationAttributes(annotation);
System.out.println("annotationAttributeMap: " + annotationAttributeMap);

// 获取value的值
Object value = AnnotationUtils.getValue(annotation, "value");
System.out.println("value: " + value);

```

结果如下：

```
annotation: @org.springframework.stereotype.Component(value=123)
AnnotationAttributes: {value=123}
annotationAttributeMap: {value=123}
value: 123

```

从结果来看，直接通过 `AnnotationUtils.getAnnotation(...)` 也是能获取到 `@Component` 注解的，尽管 `BeanObj3` 并没有直接标记 `@Component`. 需要注意的是，这样获取到的 `@Component` 的 `value` 值是 "123"，并不是 `@MyComponent` 设置的 `beanObj3`，这也证明了 `AnnotationUtils` 获取属性值时并不进行属性覆盖操作。

#### 4.2 `AnnotatedElementUtils`

`AnnotatedElementUtils` 支持的部分方法如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-9815fe697f5ccf14d3aa215952bfcbcc3ab.png)

给个示例吧：

```
// 1\. 判断是否有 Component 注解
boolean result = AnnotatedElementUtils.hasAnnotation(BeanObj3.class, Component.class);
System.out.println("hasAnnotation: " + result);

// 2\. 获取 attributeMap，可以看到的是，获取 @Component 与 @MyComponent 得到的结果不一样
// Component attributeMap: {value=[123]}
MultiValueMap<String, Object> attributeMap1 = AnnotatedElementUtils
        .getAllAnnotationAttributes(BeanObj3.class, Component.class.getName());
System.out.println("Component attributeMap: " + attributeMap1);
// MyComponent attributeMap: {value=[beanObj3]}
MultiValueMap<String, Object> attributeMap2 = AnnotatedElementUtils
        .getAllAnnotationAttributes(BeanObj3.class, MyComponent.class.getName());
System.out.println("MyComponent attributeMap: " + attributeMap2);

// 3\. 获取所有的 @Component 注解，value=beanObj3
Set<Component> mergedAnnotations = AnnotatedElementUtils
        .getAllMergedAnnotations(BeanObj3.class, Component.class);
System.out.println("mergedAnnotations: " + mergedAnnotations);

// 4\. 获取属性值，{value=beanObj3}
AnnotationAttributes attributes = AnnotatedElementUtils
        .getMergedAnnotationAttributes(BeanObj3.class, Component.class);
System.out.println("attributes: " + attributes);

// 5\. 获取 MyComponent 上的注解
Set<String> types = AnnotatedElementUtils
        .getMetaAnnotationTypes(BeanObj3.class, MyComponent.class);
System.out.println("types: " + types);

```

结果如下：

```
hasAnnotation: true
Component attributeMap: {value=[123]}
MyComponent attributeMap: {value=[beanObj3]}
mergedAnnotations: [@org.springframework.stereotype.Component(value=beanObj3)]
attributes: {value=beanObj3}
types: [org.springframework.stereotype.Component, org.springframework.stereotype.Indexed]

```

从代码来看，在得到的 `Set<Component>` 与 `AnnotationAttributes` 中，属性值已经合并了.

在选择使用 `AnnotationUtils` 还是 `AnnotatedElementUtils` 时，可以根据要不要属性覆盖来选择，如果需要处理属性覆盖，就使用 `AnnotatedElementUtils`，如果不需要，就使用 `AnnotationUtils` 吧！

### 5\. 总结

本文介绍了 spring 处理注解的操作，主要介绍了 `SimpleAnnotationMetadataReadingVisitor` 与 `StandardAnnotationMetadata` 的区别与使用方法。由于这两个类使用起来步骤比较多，文中又介绍了 spring 提供的两个工具类：`AnnotationUtils` 与 `AnnotatedElementUtils`，如果需要处理属性覆盖，需要使用 `AnnotatedElementUtils`，如果不需要，就使用 `AnnotationUtils`。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4633161](https://my.oschina.net/funcy/blog/4633161) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_