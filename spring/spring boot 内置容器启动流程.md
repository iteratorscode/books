## Spring boot内置容器启动流程

### AnnotationConfigServletWebServerApplicationContext

#### 继承关系如下

```java
// 继承ServletWebServerApplicationContext
public class AnnotationConfigServletWebServerApplicationContext extends ServletWebServerApplicationContext
		implements AnnotationConfigRegistry {
  // ...
}
// 进而继承ServletWebServerApplicationContext，直到AbstractApplicationContext
public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
  // ...
}
```

在context实例启动过程中，必然会执行`org.springframework.context.support.AbstractApplicationContext#refresh`方法，其中定义了Spring IOC容器初始化的完整流程，而其中的`org.springframework.context.support.AbstractApplicationContext#onRefresh`流程交给了子类去自定义自己的特殊过程，`AnnotationConfigServletWebApplicationContext`的父类实现了`onRefresh`方法

```java
@Override
protected void onRefresh() {
  // 1 执行父类的onRefresh
  super.onRefresh();
  try {
    // 2 NOTE：执行内置的webServer的创建和启动
    createWebServer();
  }
  catch (Throwable ex) {
    throw new ApplicationContextException("Unable to start web server", ex);
  }
}
```

#### 内置容器的启动代码

```java
private void createWebServer() {
  WebServer webServer = this.webServer;
  ServletContext servletContext = getServletContext();
  // 1 内置容器不存在，创建内置容器
  if (webServer == null && servletContext == null) {
    StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
    // 2 获取WebServerFactory，默认的实现为 TomcatServletWebServerFactory
    ServletWebServerFactory factory = getWebServerFactory();
    createWebServer.tag("factory", factory.getClass().toString());
    // 3 获取webServer实例
    this.webServer = factory.getWebServer(getSelfInitializer());
    createWebServer.end();
    // 4 内置容器启动的后续事件 
    getBeanFactory().registerSingleton("webServerGracefulShutdown",
                                       new WebServerGracefulShutdownLifecycle(this.webServer));
    getBeanFactory().registerSingleton("webServerStartStop",
                                       new WebServerStartStopLifecycle(this, this.webServer));
  }
  else if (servletContext != null) {
    try {
      getSelfInitializer().onStartup(servletContext);
    }
    catch (ServletException ex) {
      throw new ApplicationContextException("Cannot initialize servlet context", ex);
    }
  }
  // TODO：初始化配置源信息
  initPropertySources();
}
```

#### 内置Tomcat创建流程

```java
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
  if (this.disableMBeanRegistry) {
    Registry.disableRegistry();
  }
  // 1 构建tomcat实例
  Tomcat tomcat = new Tomcat();
  // 2 设置baseDir
  File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
  tomcat.setBaseDir(baseDir.getAbsolutePath());
  // 3 设置conector
  Connector connector = new Connector(this.protocol);
  connector.setThrowOnFailure(true);
  tomcat.getService().addConnector(connector);
  customizeConnector(connector);
  tomcat.setConnector(connector);
  tomcat.getHost().setAutoDeploy(false);
  configureEngine(tomcat.getEngine());
  for (Connector additionalConnector : this.additionalTomcatConnectors) {
    tomcat.getService().addConnector(additionalConnector);
  }
  // 4 初始化tomcat的servlet信息，filter信息
  prepareContext(tomcat.getHost(), initializers);
  // 5 构建并启动tomcat
  return getTomcatWebServer(tomcat);
}

// 6 构建、启动
protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
  return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
}
```

#### tomcat的构建启动流程

```java
public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
  Assert.notNull(tomcat, "Tomcat Server must not be null");
  this.tomcat = tomcat;
  this.autoStart = autoStart;
  this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
  // 初始化tomcat
  initialize();
}

private void initialize() throws WebServerException {
  logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
  synchronized (this.monitor) {
    try {
      addInstanceIdToEngineName();

      Context context = findContext();
      context.addLifecycleListener((event) -> {
        if (context.equals(event.getSource()) && Lifecycle.START_EVENT.equals(event.getType())) {
          // Remove service connectors so that protocol binding doesn't
          // happen when the service is started.
          removeServiceConnectors();
        }
      });

      // Start the server to trigger initialization listeners
      this.tomcat.start();

      // We can re-throw failure exception directly in the main thread
      rethrowDeferredStartupExceptions();

      try {
        ContextBindings.bindClassLoader(context, context.getNamingToken(), getClass().getClassLoader());
      }
      catch (NamingException ex) {
        // Naming is not enabled. Continue
      }

      // Unlike Jetty, all Tomcat threads are daemon threads. We create a
      // blocking non-daemon to stop immediate shutdown
      startDaemonAwaitThread();
    }
    catch (Exception ex) {
      stopSilently();
      destroySilently();
      throw new WebServerException("Unable to start embedded Tomcat", ex);
    }
  }
}
```

#### 构建内置容器对象时获取Servlet Bean对象的流程`getSelfInitializer()`

```java
// 返回值时Spring 自定义的ServletContextInitializer
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
  // 通过方法引用构建了一个匿名内部类
  return this::selfInitialize;
}


private void selfInitialize(ServletContext servletContext) throws ServletException {
  prepareWebApplicationContext(servletContext);
  registerApplicationScope(servletContext);
  WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
  // NOTE：获取IOC容器中所有的ServletContextInitializer实例，执行对应的初始化过程
  for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
    beans.onStartup(servletContext);
  }
}
```

#### ServletContextInitializer实例解析`getServletContextInitializerBeans()`

```java
// 1 利用BeanFactory实例构建实现了Collection接口的对象 ServletContextInitializerBeans，
protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
  return new ServletContextInitializerBeans(getBeanFactory());
}

// 2 foreach遍历Collection的底层会调用Collection对象的　iterator, 所以关注  ServletContextInitializerBeans的 iterator()方法

// 3 org.springframework.boot.web.servlet.ServletContextInitializerBeans#iterator
@Override
public Iterator<ServletContextInitializer> iterator() {
  return this.sortedList.iterator();
}

// 4 关注ServletContextInitializerBeans#sortedList中如何添加数据，在构造方法中
@SafeVarargs
@SuppressWarnings("varargs")
public ServletContextInitializerBeans(ListableBeanFactory beanFactory,
                                      Class<? extends ServletContextInitializer>... initializerTypes) {
  this.initializers = new LinkedMultiValueMap<>();
  // 4.1  设置类型，默认为ServletContextInitializer
  this.initializerTypes = (initializerTypes.length != 0) ? Arrays.asList(initializerTypes)
    : Collections.singletonList(ServletContextInitializer.class);
  // 4.2 从容器中解析ServletContextInitializer实例，并添加到initializers中
  addServletContextInitializerBeans(beanFactory);
  // 4.3 从容器中解析ServletRegistrationBeanAdapter、ServletListenerRegistrationBeanAdapter等类型，并添加到initializers中
  addAdaptableBeans(beanFactory);
  // 4.4 对initializers中元素排序
  List<ServletContextInitializer> sortedInitializers = this.initializers.values().stream()
    .flatMap((value) -> value.stream().sorted(AnnotationAwareOrderComparator.INSTANCE))
    .collect(Collectors.toList());
  // 4.5 将initializers中元素赋值给sortedList，用于迭代器遍历
  this.sortedList = Collections.unmodifiableList(sortedInitializers);
  logMappings(this.initializers);
}

// 5 解析ServletContextInitializer实例
private void addServletContextInitializerBean(String beanName, ServletContextInitializer initializer,
			ListableBeanFactory beanFactory) {
  if (initializer instanceof ServletRegistrationBean) {
    Servlet source = ((ServletRegistrationBean<?>) initializer).getServlet();
    // 5.1 解析容器中Servlet类型的Bean
    addServletContextInitializerBean(Servlet.class, beanName, initializer, beanFactory, source);
  }
  else if (initializer instanceof FilterRegistrationBean) {
    // 5.2 解析容器中Filter类型的Bean
    Filter source = ((FilterRegistrationBean<?>) initializer).getFilter();
    addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
  }
  else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
    String source = ((DelegatingFilterProxyRegistrationBean) initializer).getTargetBeanName();
    // 5.3 解析容器中Filter类型的Bean
    addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
  }
  else if (initializer instanceof ServletListenerRegistrationBean) {
    EventListener source = ((ServletListenerRegistrationBean<?>) initializer).getListener();
    // 5.4 解析容器中EventListener类型的Bean
    addServletContextInitializerBean(EventListener.class, beanName, initializer, beanFactory, source);
  }
  else {
    // 解析容器中ServletContextInitializer类型的Bean
    addServletContextInitializerBean(ServletContextInitializer.class, beanName, initializer, beanFactory,
                                     initializer);
  }
}
```
