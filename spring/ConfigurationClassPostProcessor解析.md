# ConfigurationClassPostProcessor解析

`ConfigurationClassPostProcessor`通过实现接口`BeanDefinitionRegistryPostProcessor`, 从而具有了对BeanDefinition对象进行拦截增强的能力

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, ApplicationStartupAware, BeanClassLoaderAware, EnvironmentAware {
      // ...
}
```

BeanDefinition的处理入口

```java
// ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
  // ...
  
  // 此方法会处理BeanDefinition
  processConfigBeanDefinitions(registry);
}
```

处理`Configuration`类的流程

```java
// ConfigurationClassPostProcessor#processConfigBeanDefinitions
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
  List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
  // 1 获取当前所有的beanDefinition
  String[] candidateNames = registry.getBeanDefinitionNames();

  for (String beanName : candidateNames) {
    BeanDefinition beanDef = registry.getBeanDefinition(beanName);
    // 2 找出被@Configuration标注的配置类，加入到候选集configCandidates中
    if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
      configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
    }
  }

  // 3 如果没有@Configuration类，直接返回
  if (configCandidates.isEmpty()) {
    return;
  }

  // 4 根据@Order值对@Configuration标注的类排序
  configCandidates.sort((bd1, bd2) -> {
    int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
    int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
    return Integer.compare(i1, i2);
  });
  // ...
  // 5 解析每一个 @Configuration class
  ConfigurationClassParser parser = new ConfigurationClassParser(
    this.metadataReaderFactory, this.problemReporter, this.environment,
    this.resourceLoader, this.componentScanBeanNameGenerator, registry);

  Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
  Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
  do {
    StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
    // 6 解析
    parser.parse(candidates);
    parser.validate();

    Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
    configClasses.removeAll(alreadyParsed);

    // 7 初始化DefinitionReader
    if (this.reader == null) {
      this.reader = new ConfigurationClassBeanDefinitionReader(
        registry, this.sourceExtractor, this.resourceLoader, this.environment,
        this.importBeanNameGenerator, parser.getImportRegistry());
    }
    // 8 加载@Configuration类中的BeanDefinition到registry中
    this.reader.loadBeanDefinitions(configClasses);
    alreadyParsed.addAll(configClasses);
    processConfig.tag("classCount", () -> String.valueOf(configClasses.size())).end();

    candidates.clear();
    if (registry.getBeanDefinitionCount() > candidateNames.length) {
      String[] newCandidateNames = registry.getBeanDefinitionNames();
      Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
      Set<String> alreadyParsedClasses = new HashSet<>();
      for (ConfigurationClass configurationClass : alreadyParsed) {
        alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
      }
      // 9 解析beanDefinition中的Configuration类，继续循环♻️扫描
      for (String candidateName : newCandidateNames) {
        if (!oldCandidateNames.contains(candidateName)) {
          BeanDefinition bd = registry.getBeanDefinition(candidateName);
          if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
              !alreadyParsedClasses.contains(bd.getBeanClassName())) {
            candidates.add(new BeanDefinitionHolder(bd, candidateName));
          }
        }
      }
      candidateNames = newCandidateNames;
    }
  }
  while (!candidates.isEmpty());
  // ...
}
```

将`Configuration`class中包含的Bean定义相关配置解析道configuration class model中

```java
//ConfigurationClassParser#parse
public void parse(Set<BeanDefinitionHolder> configCandidates) {
  for (BeanDefinitionHolder holder : configCandidates) {
    BeanDefinition bd = holder.getBeanDefinition();
    try {
      // 以下1、2、3都会调用相同的方法processConfigurationClass统一处理
      if (bd instanceof AnnotatedBeanDefinition) {
        // 1 处理AnnotatedBeanDefinition
        parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
      }
      else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
        // 2 处理AbstractBeanDefinition
        parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
      }
      else {
        // 3 处理Bean
        parse(bd.getBeanClassName(), holder.getBeanName());
      }
    }
    catch (BeanDefinitionStoreException ex) {
      throw ex;
    }
    catch (Throwable ex) {
      throw new BeanDefinitionStoreException(
        "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
    }
  }
  // 4 解析@Configuration类中的@Import注解，
  // 主要处理ImportSelector, DeferredImportSelector, Configuration
  // 最终会完善configClass上的相关属性，后期loadBeanDefinition使用
  // 比如： importBeanDefinitionRegistrars，importedResources，beanMethods等
  this.deferredImportSelectorHandler.process();
}

```

真正的Configuration类处理流程

```java
// ConfigurationClassParser#processConfigurationClass
protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
  // ...

  // 递归处理configuration class 以及他继承的父 superclass上的Configuration.
  SourceClass sourceClass = asSourceClass(configClass, filter);
  do {
    sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
  }
  while (sourceClass != null);

  // 将已经解析过的缓存起来
  this.configurationClasses.put(configClass, configClass);
}
// doProcessConfigurationClass
@Nullable
protected final SourceClass doProcessConfigurationClass(
  ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
  throws IOException {
  // 1 处理@Component注解类中的内部类
  if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
    // 解析每个配置类的内部类中被@configuration标注的
    processMemberClasses(configClass, sourceClass, filter);
  }

  // 2 处理 @PropertySource 注解
  for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
    sourceClass.getMetadata(), PropertySources.class,
    org.springframework.context.annotation.PropertySource.class)) {
    if (this.environment instanceof ConfigurableEnvironment) {
      processPropertySource(propertySource);
    }
  }

  // 3 处理 @ComponentScan注解
  Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
    sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
  if (!componentScans.isEmpty() &&
      !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
    for (AnnotationAttributes componentScan : componentScans) {
      // The config class is annotated with @ComponentScan -> perform the scan immediately
      Set<BeanDefinitionHolder> scannedBeanDefinitions =
        this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
      // Check the set of scanned definitions for any further config classes and parse recursively if needed
      for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
        BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
        if (bdCand == null) {
          bdCand = holder.getBeanDefinition();
        }
        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
          parse(bdCand.getBeanClassName(), holder.getBeanName());
        }
      }
    }
  }

  // 4 处理@Import注解
  // 主要处理ImportSelector, DeferredImportSelector, Configuration， 以及任意的一个类（这个类会被当作Configuration处理）
  processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

  // 5 处理@ImportResource注解
  AnnotationAttributes importResource =
    AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
  if (importResource != null) {
    String[] resources = importResource.getStringArray("locations");
    Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
    for (String resource : resources) {
      String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
      configClass.addImportedResource(resolvedResource, readerClass);
    }
  }

  // 6 处理@Bean注解
  Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
  for (MethodMetadata methodMetadata : beanMethods) {
    configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
  }

  // 7 处理configClass实现的接口中的方法，解析出接口中的@Bean方法，接口方法可以提供默认值
  processInterfaces(configClass, sourceClass);

  // 8 递归处理父类
  if (sourceClass.getMetadata().hasSuperClass()) {
    String superclass = sourceClass.getMetadata().getSuperClassName();
    if (superclass != null && !superclass.startsWith("java") &&
        !this.knownSuperclasses.containsKey(superclass)) {
      this.knownSuperclasses.put(superclass, configClass);
      // Superclass found, return its annotation metadata and recurse
      return sourceClass.getSuperClass();
    }
  }

  // No superclass -> processing is complete
  return null;
}

```

`Configuration`class解析完成后，会将该类中的Bean定义注册到registry中，注册流程如下

```java
// ConfigurationClassBeanDefinitionReader#loadBeanDefinitions
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
  TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
  // 遍历每一个configClass执行加载操作
  for (ConfigurationClass configClass : configurationModel) {
    loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
  }
}
// loadBeanDefinitionsForConfigurationClass
private void loadBeanDefinitionsForConfigurationClass(
  ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

  if (trackedConditionEvaluator.shouldSkip(configClass)) {
    String beanName = configClass.getBeanName();
    if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
      this.registry.removeBeanDefinition(beanName);
    }
    this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
    return;
  }

  // 加载不同情形下的几种BeanDefinition
  if (configClass.isImported()) {
    registerBeanDefinitionForImportedConfigurationClass(configClass);
  }
  // @Bean方法转换成对应的BeanDefinition
  for (BeanMethod beanMethod : configClass.getBeanMethods()) {
    loadBeanDefinitionsForBeanMethod(beanMethod);
  }

  // ImportedResource转换成对应的BeanDefinition
  loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
  // 执行ImportBeanDefinitionRegistrar转换成对应的BeanDefinition
  loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```



