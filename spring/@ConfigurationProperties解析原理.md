# @ConfigurationProperties解析原理

首先可以通过`ConfigurationPropertiesScan`手动开启ConfigurationProperties自动扫描，源码如下

```java
@Import(ConfigurationPropertiesScanRegistrar.class)
@EnableConfigurationProperties
public @interface ConfigurationPropertiesScan {
  // ...
}
```

其通过`ConfigurationPropertiesScanRegistrar`实现扫描basePackages指定包下的ConfigurationProperties Bean，并在容器中注册相关Bean的定义；而`EnableConfigurationProperties`会在容器中注入一个Processor，其会从Environment中解析出ConfigurationProperties Bean的相关属性值并绑定的工作。注意，EnableConfigurationProperties也可以向容器中注册value属性指定的Bean Definition。其源码如下：

```java
@Import(EnableConfigurationPropertiesRegistrar.class)
public @interface EnableConfigurationProperties {

  String VALIDATOR_BEAN_NAME = "configurationPropertiesValidator";

  Class<?>[] value() default {};
}
```



## ConfigurationProperties Bean的扫描

ConfigurationPropertiesScan

```java
@Import(ConfigurationPropertiesScanRegistrar.class)
@EnableConfigurationProperties
public @interface ConfigurationPropertiesScan {
  @AliasFor("value")
  String[] basePackages() default {};
}
```

ConfigurationPropertiesScanRegistrar

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
  // 1 获取需要扫描的包
  Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
  // 2 扫描每个包下的ConfigurationProperties并在registry中注入相关Bean定义
  scan(registry, packagesToScan);
}
```



## ConfigurationProperties Bean的属性绑定

EnableConfigurationPropertiesRegistrar

```java
class EnableConfigurationPropertiesRegistrar implements ImportBeanDefinitionRegistrar {

  @Override
  public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 1 注册基础Bean，完成属性绑定的功能
    registerInfrastructureBeans(registry);
    ConfigurationPropertiesBeanRegistrar beanRegistrar = new ConfigurationPropertiesBeanRegistrar(registry);
    // 2 将EnableConfigurationProperties的value生成BeanDefinition注册到容器中
    getTypes(metadata).forEach(beanRegistrar::register);
  }

  // 解析出EnableConfigurationProperties的value值
  private Set<Class<?>> getTypes(AnnotationMetadata metadata) {
    return metadata.getAnnotations().stream(EnableConfigurationProperties.class)
      .flatMap((annotation) -> Arrays.stream(annotation.getClassArray(MergedAnnotation.VALUE)))
      .filter((type) -> void.class != type).collect(Collectors.toSet());
  }

  @SuppressWarnings("deprecation")
  static void registerInfrastructureBeans(BeanDefinitionRegistry registry) {
    ConfigurationPropertiesBindingPostProcessor.register(registry);
    ConfigurationBeanFactoryMetadata.register(registry);
  }
}
// ConfigurationPropertiesBindingPostProcessor#register
注册ConfigurationPropertiesBindingPostProcessor

// ConfigurationBeanFactoryMetadata#register
注册ConfigurationPropertiesBinder.Factory和ConfigurationPropertiesBinder
```

ConfigurationPropertiesBindingPostProcessor处理ConfigurationProperties bean属性值的绑定

```java
// ConfigurationPropertiesBindingPostProcessor实现了多个Bean lifecycle接口
public class ConfigurationPropertiesBindingPostProcessor
  implements BeanPostProcessor, PriorityOrdered, ApplicationContextAware, InitializingBean {
  // ...
  // ApplicationContextAware接口会获取到ApplicationContext
  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    this.applicationContext = applicationContext;
  }

  // InitializingBean接口中会构造属性绑定需要的类
  @Override
  public void afterPropertiesSet() throws Exception {
    this.registry = (BeanDefinitionRegistry) this.applicationContext.getAutowireCapableBeanFactory();
    // 属性绑定
    this.binder = ConfigurationPropertiesBinder.get(this.applicationContext);
  }
  // BeanPostProcessor 接口拦截每个Bean的创建过程
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    // 绑定属性值
    bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
    return bean;
  }
}

```



这样的BeanDefinition有什么特殊意义？为了指定构建另一个Bean的工厂

```java
static void register(BeanDefinitionRegistry registry) {
  if (!registry.containsBeanDefinition(FACTORY_BEAN_NAME)) {
    GenericBeanDefinition definition = new GenericBeanDefinition();
    definition.setBeanClass(ConfigurationPropertiesBinder.Factory.class);
    definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(ConfigurationPropertiesBinder.FACTORY_BEAN_NAME, definition);
  }
  if (!registry.containsBeanDefinition(BEAN_NAME)) {
    GenericBeanDefinition definition = new GenericBeanDefinition();
    definition.setBeanClass(ConfigurationPropertiesBinder.class);
    definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    definition.setFactoryBeanName(FACTORY_BEAN_NAME);
    definition.setFactoryMethodName("create");
    registry.registerBeanDefinition(ConfigurationPropertiesBinder.BEAN_NAME, definition);
  }
}
// ConfigurationPropertiesBinder这个Bean是通过ConfigurationPropertiesBinder.Factory这个Bean 来创建的
```

>  **factoryBeanName、factoryMethodName**
> 如果本bean是通过其他工厂bean来创建，则这两个字段为对应的工厂bean的名称，和对应的工厂方法的名称

