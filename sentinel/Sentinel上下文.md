# Sentinel上下文

## Context

`Context`是对资源操作的上下文呢，每个资源操作必须属于一个Context。如果代码中没有指定Context，则会创建一个name为sentinel_default_context的默认上下文。

**一个Context的生命周期中可以包含多个资源操作。**Context的生命周期中的最后一个资源在exit()时会清理该context，这也就意味着这个Context生命周期结束了。

### Context的特点

- 每个Context都有一个EntranceNode入口节点。（一般每个请求对应着一个Context上下文。）
- 不同名称的Context具有不同的EntranceNode节点，但是同名的Context会使用同一个EntranceNode节点
- 一个Context中可以包含对多个资源的操作
- 每个资源对应着一个唯一的ProcessorSlotChain, 对资源的操作通过ProcessorSlot来完成

## Context源码分析

1. 入口`ContextUtil#enter`

   ```java
   public static Context enter(String name, String origin) {
     // 检验context name是否合理
     if (Constants.CONTEXT_DEFAULT_NAME.equals(name)) {
       throw new ContextNameDefineException(
         "The " + Constants.CONTEXT_DEFAULT_NAME + " can't be permit to defined!");
     }
     // 创建出Context
     return trueEnter(name, origin);
   }
   ```

2. 构建Context：`ContextUtil#trueEnter`

   ```java
   // contextHolder 采用ThreadLocal实现线程安全策略
   private static ThreadLocal<Context> contextHolder = new ThreadLocal<>();
   ```

   

   ```java
   protected static Context trueEnter(String name, String origin) {
     // 1. ThreadLocal方式: 每个线程(每个请求)会创建一个独立的Context
     Context context = contextHolder.get();
     if (context == null) {
       // 2. 使用缓存，保证相同的资源的入口节点是同一个
       Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;
       DefaultNode node = localCacheNameMap.get(name);
       if (node == null) {
         if (localCacheNameMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
           setNullContext();
           return NULL_CONTEXT;
         } else {
           LOCK.lock();
           try {
             node = contextNameNodeMap.get(name);
             if (node == null) {
               // 3 达到上限时，返回空的NULL_CONTEXT
               if (contextNameNodeMap.size() > Constants.MAX_CONTEXT_NAME_SIZE) {
                 setNullContext();
                 return NULL_CONTEXT;
               } else {
                 // 4 构建该资源的入口节点
                 node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);
                 // Add entrance node.
                 // 5 维护资源的节点树
                 Constants.ROOT.addChild(node);
   
                 // 6 更新资源入口节点缓存
                 Map<String, DefaultNode> newMap = new HashMap<>(contextNameNodeMap.size() + 1);
                 newMap.putAll(contextNameNodeMap);
                 newMap.put(name, node);
                 contextNameNodeMap = newMap;
               }
             }
           } finally {
             LOCK.unlock();
           }
         }
       }
       // 7 为每个请求创建一个Context
       context = new Context(node, name);
       context.setOrigin(origin);
       contextHolder.set(context);
     }
   
     return context;
   }
   
   ```

   

