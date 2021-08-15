# Feign工作原理

## Spring cloud open feign整合原理

### FeignClient的自动注册

1. `@EnableFeignClients`通过代码`@Import(FeignClientRegistar.class)`实现FeignClient的自动注册
2. `FeignClientRegistar`扫描类路径下的所有被`@FeignClient`注解的类，通过动态代理的方式生成一个接口的实现类，注入的Spring容器中。代码位于`FeignClientsRegistrar#registerBeanDefinitions`

### 1 registerFeignClients

```java
public void registerFeignClients(AnnotationMetadata metadata,
                                 BeanDefinitionRegistry registry) {
  // 1 获取类路径扫描器
  ClassPathScanningCandidateComponentProvider scanner = getScanner();
  scanner.setResourceLoader(this.resourceLoader);

  // 2 解析出basePackages
  // 2.1 从EnableFeignClients中配置的FeignClient所在的包路径
  // 2.2 扫描类路径来解析FeignClient
  // 2.1 和 2.2只有一个会生效
  Set<String> basePackages;

  Map<String, Object> attrs = metadata
    .getAnnotationAttributes(EnableFeignClients.class.getName());
  AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
    FeignClient.class);
  final Class<?>[] clients = attrs == null ? null
    : (Class<?>[]) attrs.get("clients");
  if (clients == null || clients.length == 0) {
    scanner.addIncludeFilter(annotationTypeFilter);
    basePackages = getBasePackages(metadata);
  }
  else {
    final Set<String> clientClasses = new HashSet<>();
    basePackages = new HashSet<>();
    for (Class<?> clazz : clients) {
      basePackages.add(ClassUtils.getPackageName(clazz));
      clientClasses.add(clazz.getCanonicalName());
    }
    AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
      @Override
      protected boolean match(ClassMetadata metadata) {
        String cleaned = metadata.getClassName().replaceAll("\\$", ".");
        return clientClasses.contains(cleaned);
      }
    };
    scanner.addIncludeFilter(
      new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
  }

  for (String basePackage : basePackages) {
    // 3 找出basePackage中的FeignClient声明类
    Set<BeanDefinition> candidateComponents = scanner
      .findCandidateComponents(basePackage);
    for (BeanDefinition candidateComponent : candidateComponents) {
      if (candidateComponent instanceof AnnotatedBeanDefinition) {
        // verify annotated class is an interface
        AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
        AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
       
        Map<String, Object> attributes = annotationMetadata
          .getAnnotationAttributes(
          FeignClient.class.getCanonicalName());

        // 4 获取服务名 serviceName
        String name = getClientName(attributes);
        // 5 设置每个客户端自定义的额配置，主要包括 encoder, decoder, contract
        registerClientConfiguration(registry, name,
                                    attributes.get("configuration"));

        // 6 注册client到ioc容器中
        registerFeignClient(registry, annotationMetadata, attributes);
      }
    }
  }
}
```

### 2 registerFeignClient

注册client的Bean Definition到容器中

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
                                 AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
  String className = annotationMetadata.getClassName();
  // 1 注册生成client FactoryBean工厂类：FeignClientFactoryBean
  BeanDefinitionBuilder definition = BeanDefinitionBuilder
    .genericBeanDefinition(FeignClientFactoryBean.class);
  validate(attributes);
  // 2 设置client的各个组件
  definition.addPropertyValue("url", getUrl(attributes));
  definition.addPropertyValue("path", getPath(attributes));
  String name = getName(attributes);
  definition.addPropertyValue("name", name);
  String contextId = getContextId(attributes);
  definition.addPropertyValue("contextId", contextId);
  definition.addPropertyValue("type", className);
  definition.addPropertyValue("decode404", attributes.get("decode404"));
  definition.addPropertyValue("fallback", attributes.get("fallback"));
  definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
  definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

  String alias = contextId + "FeignClient";
  AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

  boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
  // null

  beanDefinition.setPrimary(primary);

  String qualifier = getQualifier(attributes);
  if (StringUtils.hasText(qualifier)) {
    alias = qualifier;
  }

  BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
                                                         new String[] { alias });
  // 4 注册client到容器中
  BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

### 3 FeignClientFactoryBean#getObject

通过FactoryBean工厂方法生成bean实例

```java
@Override
public Object getObject() throws Exception {
  return getTarget();
}

<T> T getTarget() {
  // 1. 从容器中获取feign的上下文feignContext：openfeign.FeignContext
  // note: feignContext主要是具有一个feign client的通用配置信息，
  // 同时具有Spring容器的上下文，从而可以方便的获取容器中的其他bean信息
  FeignContext context = this.applicationContext.getBean(FeignContext.class);
  // 2. 从上下文中获取client的Builder对象：设置编解码器和契约对象contract
  // note: Sentinel实现了自己的FeignBuilder, Hystrix也实现了自己的FeignBuilder, 
  // 以此来将各自的服务保护能力融合到feign的生态中
  Feign.Builder builder = feign(context);

  if (!StringUtils.hasText(this.url)) {
    if (!this.name.startsWith("http")) {
      this.url = "http://" + this.name;
    }
    else {
      this.url = this.name;
    }
    this.url += cleanPath();
    // 默认设置的url为空，所以会具有客户端负载均衡能力
    // 3 ribbon提供的客户端负载均衡能力在此处体现
    return (T) loadBalance(builder, context,
                           new HardCodedTarget<>(this.type, this.name, this.url));
  }
  if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
    this.url = "http://" + this.url;
  }
  String url = this.url + cleanPath();
  Client client = getOptional(context, Client.class);
  if (client != null) {
    if (client instanceof LoadBalancerFeignClient) {
      // not load balancing because we have a url,
      // but ribbon is on the classpath, so unwrap
      client = ((LoadBalancerFeignClient) client).getDelegate();
    }
    if (client instanceof FeignBlockingLoadBalancerClient) {
      // not load balancing because we have a url,
      // but Spring Cloud LoadBalancer is on the classpath, so unwrap
      client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
    }
    builder.client(client);
  }
  Targeter targeter = get(context, Targeter.class);
  // 4  targeter.target 创建客户端的代理对象
  // note: Sentinel提供自己的SentinelInvocationHandler, 
  // Hystrix提供自己的HystrixInvocationHandler, 从而提供服务的熔断和降级保护能力
  // 使用的是JDK Proxy机制，所以client在执行过程中都会执行InvocationHandler的invoke方法
  // feign.Feign#newInstance 创建client对象
  return (T) targeter.target(this, builder, context,
                             new HardCodedTarget<>(this.type, this.name, url));
}
```

## Feign执行原理

Feign client在IOC容器中的实例对象，都是通过JDK Proxy的方式生成的，执行过程的入口就是`invoke`方法，以Sentinel的为例

```java
@Override
public Object invoke(final Object proxy, final Method method, final Object[] args)
  throws Throwable {
  if ("equals".equals(method.getName())) {
    try {
      Object otherHandler = args.length > 0 && args[0] != null
        ? Proxy.getInvocationHandler(args[0]) : null;
      return equals(otherHandler);
    }
    catch (IllegalArgumentException e) {
      return false;
    }
  }
  else if ("hashCode".equals(method.getName())) {
    return hashCode();
  }
  else if ("toString".equals(method.getName())) {
    return toString();
  }

  Object result;
  MethodHandler methodHandler = this.dispatch.get(method);
  // only handle by HardCodedTarget
  if (target instanceof Target.HardCodedTarget) {
    Target.HardCodedTarget hardCodedTarget = (Target.HardCodedTarget) target;
    MethodMetadata methodMetadata = SentinelContractHolder.METADATA_MAP
      .get(hardCodedTarget.type().getName()
           + Feign.configKey(hardCodedTarget.type(), method));
    // resource default is HttpMethod:protocol://url
    if (methodMetadata == null) {
      result = methodHandler.invoke(args);
    }
    else {
      // 1 构建resourceName
      String resourceName = methodMetadata.template().method().toUpperCase()
        + ":" + hardCodedTarget.url() + methodMetadata.template().path();
      Entry entry = null;
      try {
        // 2 访问资源链
        ContextUtil.enter(resourceName);
        entry = SphU.entry(resourceName, EntryType.OUT, 1, args);
        result = methodHandler.invoke(args);
      }
      catch (Throwable ex) {
        // 3 进行服务的限流和熔断处理
        if (!BlockException.isBlockException(ex)) {
          Tracer.trace(ex);
        }
        if (fallbackFactory != null) {
          try {
            Object fallbackResult = fallbackMethodMap.get(method)
              .invoke(fallbackFactory.create(ex), args);
            return fallbackResult;
          }
          catch (IllegalAccessException e) {
            // shouldn't happen as method is public due to being an
            // interface
            throw new AssertionError(e);
          }
          catch (InvocationTargetException e) {
            throw new AssertionError(e.getCause());
          }
        }
        else {
          // throw exception if fallbackFactory is null
          throw ex;
        }
      }
      finally {
        if (entry != null) {
          entry.exit(1, args);
        }
        ContextUtil.exit();
      }
    }
  }
  else {
    // other target type using default strategy
    result = methodHandler.invoke(args);
  }

  return result;
}
```

