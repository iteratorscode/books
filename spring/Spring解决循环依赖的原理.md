# Spring解决循环依赖的原理

## 情形

两个类分别是`Foo`和`Bar`，其定义如下

```java
public class Foo {
  private Bar bar;
}

public class Bar {
  private Foo foo;
}
```

可以看到，`Foo`类中有一个属性`bar`和`Bar`类中有一个属性`foo`, 我们可以称为`Foo`依赖了`Bar`, 而`Bar`又反过来依赖了`Foo`，则这两个类形成了相互依赖。

**那么Spring是如何解决这种属性相互依赖的呢？**

## 解决相互依赖原理

