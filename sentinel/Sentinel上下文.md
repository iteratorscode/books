# Sentinel上下文

## Context

`Context`是对资源操作的上下文呢，每个资源操作必须属于一个Context。如果代码中没有指定Context，则会创建一个name为sentinel_default_context的默认上下文。

**一个Context的生命周期中可以包含多个资源操作。**Context的生命周期中的最后一个资源在exit()时会清理该context，这也就意味着这个Context生命周期结束了。

### Context的特点

- 每个Context对应着一个唯一的EntranceNode入口节点。（一般每个请求对应着一个Context上下文。）
- 一个Context中可以包含对多个资源的操作
- 每个对资源的操作对应着一个唯一的DefaultNode节点
- 相同的的资源（资源名称相同）具有一个唯一的ProcessorSlotChain ----> 也就是说，DefaultNode节点是Context纬度的，而ProcessorSlotChain是资源纬度的，从而，**在不同上下文Context中的相同资源的DefaultNode对应着同一个ProcessorSlotChain。**

