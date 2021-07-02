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

首先执行Foo的创建

```java
// AbstractApplicationContext#getBean
@Override
public Object getBean(String name) throws BeansException {
  assertBeanFactoryActive();
  return getBeanFactory().getBean(name);
} 
// 
```

进一步执行 AbstractBeanFactory#doGetBean，其中首先执行如下代码

```java
// Eagerly check singleton cache for manually registered singletons.
Object sharedInstance = getSingleton(beanName);
// 即
@Override
@Nullable
public Object getSingleton(String beanName) {
  return getSingleton(beanName, true);
}
```

进一步地，分别从`singletonObjects` `earlySingletonObjects` `singletonFactories`依次查询Bean对象是否存在。代码如下

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 1 从singletonObjects获取对象
  Object singletonObject = this.singletonObjects.get(beanName);
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    // 2 从earlySingletonObjects获取对象
    singletonObject = this.earlySingletonObjects.get(beanName);
    if (singletonObject == null && allowEarlyReference) {
      synchronized (this.singletonObjects) {
        // Consistent creation of early reference within full singleton lock
        singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
          singletonObject = this.earlySingletonObjects.get(beanName);
          if (singletonObject == null) {
            // 3 从singletonFactories获取对象
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
              singletonObject = singletonFactory.getObject();
              this.earlySingletonObjects.put(beanName, singletonObject);
              this.singletonFactories.remove(beanName);
            }
          }
        }
      }
    }
  }
  return singletonObject;
}
```

此时，Foo对象从这三个地方是获取不到任何信息的，所以，需要进一步执行如下信息：

```java
// 创建Bean实例对象.
if (mbd.isSingleton()) {
  sharedInstance = getSingleton(beanName, () -> {
    try {
      return createBean(beanName, mbd, args);
    }
    catch (BeansException ex) {
      // Explicitly remove instance from singleton cache: It might have been put there
      // eagerly by the creation process, to allow for circular reference resolution.
      // Also remove any beans that received a temporary reference to the bean.
      destroySingleton(beanName);
      throw ex;
    }
  });
  beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

此处，`DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)` 接受两个参数，一个beanName, 一个对象工厂（通过Lambda方式写的匿名对象）。观察`getSingleton`原理如下

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  synchronized (this.singletonObjects) {
    // 1 首先从singletonObjects获取对象
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null) {
      // 1.1 将对象添加到正在创建的Bean的缓存singletonsCurrentlyInCreation中
      beforeSingletonCreation(beanName);
      boolean newSingleton = false;
      boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
      if (recordSuppressedExceptions) {
        this.suppressedExceptions = new LinkedHashSet<>();
      }
      try {
        // 2 使用工厂创建对象
        singletonObject = singletonFactory.getObject();
        newSingleton = true;
      }
      catch (IllegalStateException ex) {
        // Has the singleton object implicitly appeared in the meantime ->
        // if yes, proceed with it since the exception indicates that state.
        singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
          throw ex;
        }
      }
      catch (BeanCreationException ex) {
        if (recordSuppressedExceptions) {
          for (Exception suppressedException : this.suppressedExceptions) {
            ex.addRelatedCause(suppressedException);
          }
        }
        throw ex;
      }
      finally {
        if (recordSuppressedExceptions) {
          this.suppressedExceptions = null;
        }
        // 3 从正在创建的Bean的缓存singletonsCurrentlyInCreation中移除该对象
        afterSingletonCreation(beanName);
      }
      if (newSingleton) {
        // 4 将创建的对像加入到singletonObjects中
        addSingleton(beanName, singletonObject);
      }
    }
    return singletonObject;
  }
}
```

使用工厂创建对象的过程中，在对象实例构建完成后，对可以提前暴露的对象会进行如下处理

```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                  isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
  // 如果可以提前暴露，进行如下处理
  addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
// 将提前暴露对象的工厂放入singletonFactories中
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
  synchronized (this.singletonObjects) {
    if (!this.singletonObjects.containsKey(beanName)) {
      // 将提前暴露对象的工厂放入singletonFactories中
      this.singletonFactories.put(beanName, singletonFactory);
      this.earlySingletonObjects.remove(beanName);
      this.registeredSingletons.add(beanName);
    }
  }
}
```

接下来会执行属性装配和对象初始化操作。在属性装配过程中，从容器中获取依赖的Bean对象，依赖的Bean不存在时会执行该依赖对象的创建操作。重复上述流程。

## 总结

Foo和Bar相互依赖的解决过程

1. 执行Foo对象的获取操作
2. 分别从`singletonObjects` `earlySingletonObjects` `singletonFactories`依次查询Bean对象是否存在。首次查询不到，执行第3步
3. 执行Foo对象的实例化操作得到对象foo
4. 将foo对象放入提前暴露的工厂singletonFactories中
5. 执行foo的属性装配`populateBean`过程，发现依赖Bar对象bar，那么继续执行Bar对象的获取
6. 从容器中获取Bar对象，分别分别从`singletonObjects` `earlySingletonObjects` `singletonFactories`依次查询Bean对象是否存在。首次查询不到，执行第7步
7. 执行Bar对象的实例化操作得到对象bar
8. 将bar对象放入提前暴露的工厂singletonFactories中
9. 执行bar对象的属性装配过程，发现依赖Foo对象的foo
10. 从容器中获取foo，执行`getBean`方法
11. 分别从`singletonObjects` `earlySingletonObjects` `singletonFactories`依次查询foo对象是否存在，此时可以获取到提前暴露的foo对象，那么bar对象的属性装配就可以完成
12. bar对象获取成功，继续返回执行foo对象的属性装配
13. 执行foo的初始化，完成foo对象的构建

**可以看出，比如先完成Foo和Bar对象的实例化，然后在属性装配时使用3即缓存就可以解决属性循环依赖。如果是构造方法循环依赖，因为无法实例化成功，所以3级缓存无法解决构造方法循环依赖。**

