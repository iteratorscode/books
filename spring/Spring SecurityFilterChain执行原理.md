# Spring SecurityFilterChain执行原理

## Spring security过滤请求的入口--DelegatingFilterProxy

Spring security通过Servlet Filter的方式将资源的认证和授权的功能融入到Servlet生态中。security的入口是一个Filter, 其类型为`DelegatingFilterProxy`。具体源码`DelegatingFilterProxy#doFilter`

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
  throws ServletException, IOException {

  // Lazily initialize the delegate if necessary.
  Filter delegateToUse = this.delegate;
  if (delegateToUse == null) {
    synchronized (this.delegateMonitor) {
      delegateToUse = this.delegate;
      if (delegateToUse == null) {
        WebApplicationContext wac = findWebApplicationContext();
        if (wac == null) {
          throw new IllegalStateException("No WebApplicationContext found: " +
                                          "no ContextLoaderListener or DispatcherServlet registered?");
        }
        delegateToUse = initDelegate(wac);
      }
      this.delegate = delegateToUse;
    }
  }

  // 通过代理模式将请求交给delegate处理
  invokeDelegate(delegateToUse, request, response, filterChain);
}
```

其中主要逻辑是将请求交给delegate来处理

## 过滤器链的真正面纱--FilterChainProxy

通过代码可以知道，delegate的真实对象是`FilterChainProxy`。请求过滤的真实逻辑为`FilterChainProxy#doFilter`

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
  throws IOException, ServletException {
  boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;
  if (!clearContext) {
    doFilterInternal(request, response, chain);
    return;
  }
  try {
    request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
    doFilterInternal(request, response, chain);
  }
  catch (RequestRejectedException ex) {
    this.requestRejectedHandler.handle((HttpServletRequest) request, (HttpServletResponse) response, ex);
  }
  finally {
    SecurityContextHolder.clearContext();
    request.removeAttribute(FILTER_APPLIED);
  }
}
```

其中主要逻辑是通过`doFilterInternal`的方式处理请求

```java
private void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
  throws IOException, ServletException {
  // 1 对请求和响应对象进行包装
  FirewalledRequest firewallRequest = this.firewall.getFirewalledRequest((HttpServletRequest) request);
  HttpServletResponse firewallResponse = this.firewall.getFirewalledResponse((HttpServletResponse) response);
  // 2 获取请求对应的过滤器集合
  List<Filter> filters = getFilters(firewallRequest);
  if (filters == null || filters.size() == 0) {
    firewallRequest.reset();
    chain.doFilter(firewallRequest, firewallResponse);
    return;
  }
  // 3 构建一个虚拟过滤器链对象，这样每个请求处理线程都有自己的过滤器链对象，解决并发问题
  VirtualFilterChain virtualFilterChain = new VirtualFilterChain(firewallRequest, chain, filters);
  // 4 利用过滤器链对当前请求进行过滤
  virtualFilterChain.doFilter(firewallRequest, firewallResponse);
}
```

其中获取过滤器集合的逻辑`getFilters`

```java
private List<Filter> getFilters(HttpServletRequest request) {
  int count = 0;
  for (SecurityFilterChain chain : this.filterChains) {
    if (logger.isTraceEnabled()) {
      logger.trace(LogMessage.format("Trying to match request against %s (%d/%d)", chain, ++count,
                                     this.filterChains.size()));
    }
    if (chain.matches(request)) {
      return chain.getFilters();
    }
  }
  return null;
}
```

遍历所有的`filterChains`，通过`SecurityFilterChain#match`方法来找到`SecurityFilterChain`

## 执行请求进行过滤动作--VirtualFilterChain

将适合请求的过滤器集合构建成一个链，对请求进行过滤 `VirtualFilterChain#doFilter`

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
  if (this.currentPosition == this.size) {
    if (logger.isDebugEnabled()) {
      logger.debug(LogMessage.of(() -> "Secured " + requestLine(this.firewalledRequest)));
    }
    // Deactivate path stripping as we exit the security filter chain
    this.firewalledRequest.reset();
    this.originalChain.doFilter(request, response);
    return;
  }
  this.currentPosition++;
  // 1 获取当前链的下一个Filter
  Filter nextFilter = this.additionalFilters.get(this.currentPosition - 1);
  // 2 使用Filter执行过滤操作，注意此处将chain传递了，所以这个chain中的所有filter都会执行
  nextFilter.doFilter(request, response, this);
}
```



## 参考

- [Spring security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-filters-review)