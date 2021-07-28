# Sentinel处理器链

## ProcessorSlotChain

主要作用是将所有的`ProcessorSlot`构成一个链式结构。主要通过`first`和`end`将所有的ProcessorSlot构成一个单链表结构。

```java
// ProcessorSlotChain定义
public abstract class ProcessorSlotChain extends AbstractLinkedProcessorSlot<Object> {

  public abstract void addFirst(AbstractLinkedProcessorSlot<?> protocolProcessor);

  public abstract void addLast(AbstractLinkedProcessorSlot<?> protocolProcessor);
}
// AbstractLinkedProcessorSlot定义
public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {

  // 单链表的下一个节点next
  private AbstractLinkedProcessorSlot<?> next = null;
}

// DefaultProcessorSlotChain是ProcessorSlotChain的默认实现
public class DefaultProcessorSlotChain extends ProcessorSlotChain {

  // 单链表的头节点 first
  AbstractLinkedProcessorSlot<?> first = new AbstractLinkedProcessorSlot<Object>() {

    @Override
    public void entry(Context context, ResourceWrapper resourceWrapper, Object t, int count, boolean prioritized, Object... args)
      throws Throwable {
      super.fireEntry(context, resourceWrapper, t, count, prioritized, args);
    }

    @Override
    public void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
      super.fireExit(context, resourceWrapper, count, args);
    }

  };
  
  AbstractLinkedProcessorSlot<?> end = first;
}
```

## 每个资源如何构建ProcessorSlotChain

通过lookProcessChain的方式来为每个资源构建一个ProcessorSlot处理器链。

> 1. 无论资源resource是在哪一个Context中被使用，相同的resource全局共享同一个ProcessorSlotChain.
> 2. 当ProcessorSlotChain的数量超过最大限制6000时，将返回一个默认的null来表示已经达到最大数量了

### 1. CtSph#lookProcessChain

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
  // 1 从全局缓存中获取资源所关联的ProcessorSlotChain
  ProcessorSlotChain chain = chainMap.get(resourceWrapper);
  if (chain == null) {
    // 2 如果不存在，开始ProcessorSlotChain的创建逻辑
    // DCL锁
    synchronized (LOCK) {
      chain = chainMap.get(resourceWrapper);
      if (chain == null) {
        // Entry size limit.
        // 3 如果达到最大数量限制，返回null
        if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
          return null;
        }

        // 4 使用SPI机制构建一个新的ProcessorSlotChain
        chain = SlotChainProvider.newSlotChain();
        // 5 放入缓存中
        Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
          chainMap.size() + 1);
        newMap.putAll(chainMap);
        newMap.put(resourceWrapper, chain);
        // 6 更新缓存，防止迭代稳定性问题
        chainMap = newMap;
      }
    }
  }
  return chain;
}
```

## SPI机制创建ProcessorSlotChain

1. 加载ProcessorSlotChainBuilder

   ```java
   public static ProcessorSlotChain newSlotChain() {
     if (slotChainBuilder != null) {
       return slotChainBuilder.build();
     }
   
     // Resolve the slot chain builder SPI.
     slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();
   
     if (slotChainBuilder == null) {
       // Should not go through here.
       RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
       slotChainBuilder = new DefaultSlotChainBuilder();
     } else {
       RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: {}",
                      slotChainBuilder.getClass().getCanonicalName());
     }
     return slotChainBuilder.build();
   }
   ```

2. 构建ProcessorSlot链

   ```java
   @Spi(isDefault = true)
   public class DefaultSlotChainBuilder implements SlotChainBuilder {
   
     @Override
     public ProcessorSlotChain build() {
       ProcessorSlotChain chain = new DefaultProcessorSlotChain();
       // 1 SPI机制加载ProcessorSlot实例
       List<ProcessorSlot> sortedSlotList = SpiLoader.of(ProcessorSlot.class).loadInstanceListSorted();
       for (ProcessorSlot slot : sortedSlotList) {
         if (!(slot instanceof AbstractLinkedProcessorSlot)) {
           RecordLog.warn("The ProcessorSlot(" + slot.getClass().getCanonicalName() + ") is not an instance of AbstractLinkedProcessorSlot, can't be added into ProcessorSlotChain");
           continue;
         }
         // 2 构建链式结构
         chain.addLast((AbstractLinkedProcessorSlot<?>) slot);
       }
       return chain;
     }
   }
   ```

   

