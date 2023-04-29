public interface BeanFactory {

    /**
     * factoryBean使用
     */
    String FACTORY_BEAN_PREFIX = "&";

    /**
     * 根据名称获取bean
     */
    Object getBean(String name) throws BeansException;

    /**
     * 根据名称获取bean
     */
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    /**
     * 根据名称获取bean
     */
    Object getBean(String name, Object... args) throws BeansException;

    /**
     * 根据类型获取bean
     */
    <T> T getBean(Class<T> requiredType) throws BeansException;

    /**
     * 根据类型获取bean
     */
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    /**
     * 获取BeanProvider
     */
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    /**
     * 获取BeanProvider
     */
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    /**
     * 是否包含bean
     */
    boolean containsBean(String name);

    /**
     * 是否为单例bean
     */
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

    /**
     * 是否为原型bean
     */
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    /**
     * 判断类型是否匹配
     */
    boolean isTypeMatch(String name, ResolvableType typeToMatch) 
            throws NoSuchBeanDefinitionException;

    /**
     * 判断类型是否匹配
     */
    boolean isTypeMatch(String name, Class<?> typeToMatch) 
            throws NoSuchBeanDefinitionException;

    /**
     * 根据名称获取bean的类型
     */
    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    /**
     * 根据名称获取bean的类型
     */
    @Nullable
    Class<?> getType(String name, boolean allowFactoryBeanInit) 
            throws NoSuchBeanDefinitionException;

    /**
     * 根据bean名称获取bean的别名
     */
    String[] getAliases(String name);

}
