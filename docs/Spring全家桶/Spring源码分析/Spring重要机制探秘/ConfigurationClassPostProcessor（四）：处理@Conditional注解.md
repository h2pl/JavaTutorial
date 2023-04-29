本文是 `ConfigurationClassPostProcessor` 分析的第四篇，主要是分析 spring 对 `@Conditional` 注解的处理流程。

## 5\. spring 是如何处理 @Conditional 注解的？

### 5.1 `@Conditional` 的处理流程

在前面分析 `ConfigurationClassParser#processConfigurationClass` 方法时，有这么一行：

```
class ConfigurationClassParser {
    protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
        // 判断是否需要跳过处理，针对于 @Conditional 注解，判断是否满足条件
        if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), 
                ConfigurationPhase.PARSE_CONFIGURATION)) {
            return;
        }
        ...
    }
    ...
}

```

`conditionEvaluator.shouldSkip(...)` 方法就是用来处理 `@Conditional` 注解的，关于这个方法的处理流程，我们晚点再分析，我们先来看看什么是 `@Conditional` 注解：

```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {

    /**
     * 条件类
     */
    Class<? extends Condition>[] value();

}

```

`@Conditional` 注解非常简单，仅有一个属性，返回值是 `Class[]`，且必须是 `Condition` 的子类。我们再来看看 `Condition`：

```
public interface Condition {

    /**
     * 在这里指定匹配逻辑
     */
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}

```

`Condition` 接口仅有一个 `matches` 方法，我们可以在其中指定匹配逻辑。

接着我们来看看 `conditionEvaluator.shouldSkip(...)` 的处理流程：

```
class ConditionEvaluator {
    public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, 
            @Nullable ConfigurationPhase phase) {
        // 是否标记 @Conditional
        if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
            return false;
        }
        // 判断传入的 phase
        if (phase == null) {
            if (metadata instanceof AnnotationMetadata &&
                    ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
                return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
            }
            return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
        }

        // 实例化 condition，其放入 conditions 中
        List<Condition> conditions = new ArrayList<>();
        // 1\. getConditionClasses(metadata)：获取 @Conditional 中指定的判断类
        for (String[] conditionClasses : getConditionClasses(metadata)) {
            for (String conditionClass : conditionClasses) {
                // 2\. 实例化操作（用到的还是反射），统一放到 conditions 中
                Condition condition = getCondition(conditionClass, this.context.getClassLoader());
                conditions.add(condition);
            }
        }
        // 3\. 排序上面得到的 condition 实例
        AnnotationAwareOrderComparator.sort(conditions);
        // 遍历上面得到的 conditions
        for (Condition condition : conditions) {
            ConfigurationPhase requiredPhase = null;
            if (condition instanceof ConfigurationCondition) {
                requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
            }
            // 4\. 调用 Condition#matches 方法进行判断
            if ((requiredPhase == null || requiredPhase == phase) && 
                    !condition.matches(this.context, metadata)) {
                return true;
            }
        }

        return false;
    }

    ...
}

```

该方法的处理流程如下：

1.  获取 `@Conditional` 中指定的判断类，就是 `@Conditional` 的 `value` 属性值；
2.  使用 反射实例化第 1 步中 得到的判断类，并 保存到 `conditions`（这是个 `List`） 中；
3.  对第 2 步得到的 `conditions` 进行排序；
4.  遍历第 3 步得到的 `conditions`，调用 `Condition#matches` 方法进行匹配。

`@Conditional` 的处理还是非常简单的，接下来我们来看看它的使用示例。

### 5.2 `@Conditional` 使用示例

#### 示例 1：当指定的类存在时，才创建 spring bean

这里我们实现个功能：当指定的类存在时，才进行 spring bean 的创建、初始化，代码如下：

1. 准备一个简单的 bean:

   ```
   public class BeanObj {
   
   }
   
   ```

2. 实现 `Condition` 接口，在这里处理判断逻辑

   ```
   public class MyCondition implements Condition {
       @Override
       public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
           String className = "java.lang.Object";
           try {
               // 判断类是否存在
               Class.forName(className);
               return true;
           } catch (ClassNotFoundException e) {
               return false;
           }
       }
   }
   
   ```

   在 `matches(...)` 方法中 ，先是指定 `className` 为 `java.lang.Object`，然后判断其是否存在，判断方式也 十分简单 ，就是使用 java 的反射机制：`Class.forName(...)`，当类不存在时，会抛出 `ClassNotFoundException`，我们可以捕获该异常，从而就知道类存在不存在了。

3. 准备配置类

   ```
   @ComponentScan
   public class BeanConfigs {
       @Bean
       @Conditional(MyCondition.class)
       public BeanObj beanObj() {
           return new BeanObj();
       }
   }
   
   ```

   配置类比较简单，需要注意的是 `beanObj()` 上的 `@Conditional` 注解，指定的是条件匹配类是 `MyCondition`，匹配操作就是在这个类中进行的。

4. 主类

   ```
   public class Demo06Main {
       public static void main(String[] args) {
           ApplicationContext context 
                   = new AnnotationConfigApplicationContext(BeanConfigs.class);
           try {
               Object obj = context.getBean("beanObj");
               System.out.println("beanObj 存在！");
           } catch (Exception e) {
               System.out.println("beanObj 不存在！");
           }
       }
   }
   
   ```

运行，结果如下：

```
beanObj 存在！

```

在 `MyCondition#matches` 中，我们判断的是当前项目中是否存在 `java.lang.Object`，显然这是存在的，因此 `beanObj` 会在 spring 容器中，接着我们换下 `className`:

```
public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 更换类名
        String className = "java.lang.Object111";
        ...
    }
}

```

显然，`java.lang.Object111` 是不存在于当前项目中的，运行，结果如下：

```
beanObj 不存在！

```

#### 示例 2：改进示例 1 功能

在示例 1 中，我们通过在 `MyCondition#matches` 修改 `className` 来改变 `beanObj` 在容器中的存在情况，那如果项目中有非常多的类需要按类的存在与否进行加载，我们是实现多个 `MyCondition` 吗？

比如，类 `A` 需要根据类 `A1` 的存在与否来判断是否进行初始化，类 `B` 需要根据类 `B1` 的存在与否来判断是否进行初始化，类 `C` 需要根据类 `C1` 的存在与否来判断是否进行初始化... 我们是否需要分别为`类A`、`类B`、`类C` 实现 `Condition`，在各自的 `match(...)` 方法中进行判断吗？

实际上，我们并不需要这么做，这里通过 spring 组合注解的方法来完成以上功能。

1. 准备一个 bean，这与示例 1 并无区别

   ```
   public class BeanObj {
   
   }
   
   ```

2. 准备注解 `@ConditionalForClass`，该注解组合了 `@Conditional` 的功能，处理条件匹配的类为 `MyCondition`，`className` 属性就是必须存在的类：

```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 组合了 @Conditional 的功能，处理条件匹配的类为 MyCondition
@Conditional(MyCondition.class)
public @interface ConditionalForClass {

    /**
     * 这里指定必须存在的类
     */
    String className();

}

```

1. 准备 `MyCondition`，注意与示例的差别在于，`className` 不是在该方法中定义，而是由 `@ConditionalForClass` 传入：

   ```
   public class MyCondition implements Condition {
       /**
        * 这里处理匹配条件，注意与示例1中的区别
        */
       @Override
       public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
           // 获取 @ConditionalForClass 注解的所有属性值
           Map<String, Object> annotationAttributes = metadata.getAnnotationAttributes(
                   ConditionalForClass.class.getName());
           // 获取className的属性值，就是 @ConditionalForClass 的 className 属性
           String className = (String)annotationAttributes.get("className");
           if(null == className || className.length() <= 0) {
               return true;
           }
           try {
               // 判断类是否存在
               Class.forName(className);
               return true;
           } catch (ClassNotFoundException e) {
               return false;
           }
       }
   }
   
   ```

2. 准备配置类，此时的条件注解为 `@ConditionalForClass`：

   ```
   @ComponentScan
   public class BeanConfigs {
       @Bean
       /**
        * 在 @ConditionalForClass 中指定了依赖的类
        */
       @ConditionalForClass(className = "java.lang.Object")
       public BeanObj beanObj() {
           return new BeanObj();
       }
   }
   
   ```

3. 最后是主类，与示例 1 并不区别：

   ```
   public class Demo07Main {
   
       public static void main(String[] args) {
           ApplicationContext context 
                   = new AnnotationConfigApplicationContext(BeanConfigs.class);
           try {
               Object obj = context.getBean("beanObj");
               System.out.println("beanObj 存在！");
           } catch (Exception e) {
               System.out.println("beanObj 不存在！");
           }
       }
   }
   
   ```

以上类通过自定义注解 `@ConditionalForClass` 来指定，当类 `java.lang.Object` 存在时，`beanObj` 才会被添加到 spring 容器中，这个条件显然成立，运行，结果如下：

```
beanObj 存在！

```

我们再来调整下 `@ConditionalForClass` 的 `className` 值：

```
@ComponentScan
public class BeanConfigs {
    @Bean
    // 仅修改了@ConditionalForClass的className值，其他条件不变
    @ConditionalForClass(className = "java.lang.Object1111")
    public BeanObj beanObj() {
        return new BeanObj();
    }
}

```

这里将 `@ConditionalForClass` 的 `className` 值调整为 `java.lang.Object1111`，显然这个类并不在当前项目中，运行结果如下：

```
beanObj 不存在！

```

结果也与我们的期望一致。

让我们回到本节开头的问题：比如，类 `A` 需要根据类 `A1` 的存在与否来判断是否进行初始化，类 `B` 需要根据类 `B1` 的存在与否来判断是否进行初始化，类 `C` 需要根据类 `C1` 的存在与否来判断是否进行初始化... 我们是否需要分别为`类A`、`类B`、`类C` 实现 `Condition`，在各自的 `match(...)` 方法中进行判断吗？

有了 `@ConditionalForClass` 注解后，我们并不需要这么麻烦，只需要在各自的 `@Bean` 方法上添加 `@ConditionalForClass` 就行了，像这样：

```
@Bean
@ConditionalForClass(className = "A1")
public A a() {
    return new A();
}

@Bean
@ConditionalForClass(className = "B1")
public B b() {
    return new B();
}

@Bean
@ConditionalForClass(className = "B1")
public C c() {
    return new C();
}

...

```

注意体会 `@ConditionalForClass` 的实现，springboot 中的 `@ConditionalOnClass` 就是按这种思路实现的。

### 5.3 总结

本文主要分析了 spring 处理 `@Conditional` 的流程，逻辑比较简单，最终调用的是 `Condition#matches` 方法进行匹配操作的，而匹配操作由 `Condition` 的实现类自行指定。

为了更好地说明 `@Conditional` 的使用，本文准备了两个使用示例，特别是示例 2，需要特别体会，springboot 中的 `@ConditionalOnClass` 正是基于示例 2 的思路进行实现的，另外，springboot 中的多个条件注解也是对 `@Conditional` 的扩展。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4873444](https://my.oschina.net/funcy/blog/4873444) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_