---
title: Spring循环依赖解决方案
date: 2024-06-22
updated : 2024-06-22
categories:
- Spring
tags: 
  - Spring
  - 原理
description: "深入解析 Spring 三级缓存解决循环依赖的原理，以及 @Async 等场景下的失效分析。"
series: Spring 框架原理
series_order: 4
---

## 什么是循环依赖

循环依赖是指两个或多个 Bean 之间存在相互引用的关系：

```java
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA;
}
```

Spring 支持处理的循环依赖类型：

| 注入方式 | 是否支持 |
|---------|---------|
| Setter 注入（单例） | 支持 |
| 字段注入（单例） | 支持 |
| 构造器注入 | 不支持 |
| Prototype 作用域 | 不支持 |

## 三级缓存

Spring 通过三级缓存解决单例 Bean 的 Setter/字段注入循环依赖：

```java
public class DefaultSingletonBeanRegistry {

    // 一级缓存：完全初始化好的 Bean
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 二级缓存：早期暴露的 Bean（已实例化，未初始化）
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

    // 三级缓存：Bean 的工厂对象（用于生成早期引用）
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
}
```

### 一级缓存（singletonObjects）

存放完全走完生命周期的 Bean，即完成了实例化、属性注入、初始化的单例对象。这是 `getBean()` 首先查找的缓存。

### 二级缓存（earlySingletonObjects）

存放提前暴露的 Bean 引用，已实例化但尚未完成初始化。当一个 Bean 被其他 Bean 提前引用时，从三级缓存升级到二级缓存。

### 三级缓存（singletonFactories）

存放 `ObjectFactory<?>` 工厂对象。当调用 `getObject()` 时，工厂决定返回原始 Bean 还是代理 Bean。这个设计是为了解决 AOP 代理场景下的循环依赖。

## 完整解决流程

以 ServiceA 依赖 ServiceB、ServiceB 依赖 ServiceA 为例：

```
1. 创建 ServiceA
   ├── 实例化 ServiceA（调用构造器）
   ├── 将 ServiceA 的 ObjectFactory 放入三级缓存
   │   singletonFactories.put("serviceA", () -> getEarlyBeanReference("serviceA", serviceA))
   ├── 属性注入：发现需要 ServiceB
   │
2. 创建 ServiceB
   ├── 实例化 ServiceB（调用构造器）
   ├── 将 ServiceB 的 ObjectFactory 放入三级缓存
   ├── 属性注入：发现需要 ServiceA
   │   ├── 从一级缓存查找 → 没有
   │   ├── 从二级缓存查找 → 没有
   │   ├── 从三级缓存查找 → 找到 ObjectFactory
   │   ├── 调用 ObjectFactory.getObject() 获取 ServiceA 早期引用
   │   ├── 将 ServiceA 早期引用升级到二级缓存
   │   └── 删除三级缓存中的 ServiceA
   ├── ServiceB 完成属性注入
   ├── ServiceB 完成初始化
   ├── ServiceB 放入一级缓存
   └── 返回 ServiceB
   │
3. ServiceA 获得 ServiceB，完成属性注入
   ├── ServiceA 完成初始化
   ├── ServiceA 放入一级缓存
   └── 删除二级缓存中的 ServiceA
```

关键代码：

```java
// AbstractAutowireCapableBeanFactory.doCreateBean()
// 实例化后，提前暴露工厂
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}

// DefaultSingletonBeanRegistry.getSingleton()
// 获取 Bean 时，按顺序查找三级缓存
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);  // 一级
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);  // 二级
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();  // 三级
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

## 为什么构造器注入无法解决

构造器注入时，Bean 还没完成实例化，就无法提前暴露到三级缓存中：

```java
@Service
public class ServiceA {
    private final ServiceB serviceB;

    public ServiceA(ServiceB serviceB) {  // 构造器注入
        this.serviceB = serviceB;
    }
}
```

执行流程：
1. 尝试创建 ServiceA，调用构造器
2. 构造器需要 ServiceB，此时 ServiceA 还没有实例化完成
3. 无法将 ServiceA 暴露到三级缓存（因为还没实例化）
4. 尝试创建 ServiceB，构造器需要 ServiceA
5. ServiceA 正在创建中且没有早期引用 → 抛出 `BeanCurrentlyInCreationException`

## 为什么需要第三级缓存

第三级缓存存放的是 `ObjectFactory` 而非直接的对象引用，原因是处理 AOP 代理：

```java
// AbstractAutowireCapableBeanFactory.getEarlyBeanReference()
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
        exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
    }
    return exposedObject;
}
```

如果只有二级缓存（没有工厂）：
- 没有 AOP 时：直接暴露原始 Bean，二级缓存够用
- 有 AOP 时：需要在实例化后就创建代理对象并暴露

问题在于，Spring 的设计原则是代理对象在初始化后（`postProcessAfterInitialization`）才创建。如果为了解决循环依赖而在实例化后就创建代理，所有 Bean 都需要提前判断是否需要代理，即使没有循环依赖也会产生额外开销。

`ObjectFactory` 的延迟执行解决了这个问题：
- 没有循环依赖时：`ObjectFactory.getObject()` 不会被调用，代理正常在初始化后创建
- 有循环依赖时：`ObjectFactory.getObject()` 被调用，提前创建代理并暴露

## @Async 导致循环依赖失败的原理

`@Async` 注解通过 `AsyncAnnotationBeanPostProcessor` 在 `postProcessAfterInitialization` 中创建代理。但与 AOP 不同，它没有实现 `getEarlyBeanReference()` 方法：

```java
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;

    @Async
    public void asyncMethod() { /* ... */ }
}
```

问题场景：
1. ServiceA 实例化后，三级缓存存储 ObjectFactory
2. ServiceB 注入 ServiceA，从三级缓存获取 ServiceA 早期引用（原始对象）
3. ServiceA 初始化后，`AsyncAnnotationBeanPostProcessor` 创建了代理对象
4. 此时一级缓存中是代理对象，但 ServiceB 持有的是原始对象 → 不一致

Spring 检测到这个不一致，抛出 `BeanCurrentlyInCreationException`。

### 解决方案

**方案1：用 @Lazy 延迟注入**

```java
@Service
public class ServiceB {
    @Lazy
    @Autowired
    private ServiceA serviceA;
}
```

`@Lazy` 注入的是一个代理对象，只有在实际调用时才创建真实的 Bean。

**方案2：将 @Async 方法提取到独立的 Bean**

```java
@Service
public class AsyncService {
    @Async
    public void asyncMethod() { /* ... */ }
}

@Service
public class ServiceA {
    @Autowired
    private AsyncService asyncService;
}
```

**方案3：使用 ApplicationContext.getBean() 手动获取**

```java
@Service
public class ServiceA implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }

    public void doSomething() {
        ctx.getBean(ServiceB.class).someMethod();
    }
}
```

## @Lazy 打破循环依赖的原理

`@Lazy` 注解由 `ContextAnnotationAutowireCandidateResolver` 处理，在依赖注入时生成一个代理对象：

```java
// 当检测到 @Lazy 时，构建懒加载代理
public class LazyResolutionDescriptor extends DependencyDescriptor {
    @Override
    public Object resolveShortcut(BeanFactory beanFactory) {
        // 返回一个代理对象，首次调用时才真正获取 Bean
        return buildLazyResourceProxy(this, beanFactory);
    }
}
```

代理对象不会触发目标 Bean 的创建，等到实际调用方法时才从容器获取。这样打破了循环等待链。

## 多例 Bean 的循环依赖

Prototype 作用域的 Bean 不使用三级缓存：

```java
@Scope("prototype")
@Service
public class PrototypeA {
    @Autowired
    private PrototypeB prototypeB;
}

@Scope("prototype")
@Service
public class PrototypeB {
    @Autowired
    private PrototypeA prototypeA;
}
```

每次获取 Prototype Bean 都会创建新实例，Spring 无法缓存和提前暴露。遇到循环依赖直接抛出异常。

解决方案：重构代码消除循环依赖，或使用 `@Lazy`。

## SpringBoot 2.6+ 的变更

从 SpringBoot 2.6 开始，`spring.main.allow-circular-references` 默认为 `false`，即默认禁止循环依赖：

```yaml
#​ 如果确实需要循环依赖，可以手动开启
spring:
  main:
    allow-circular-references: true
```

这个变更的目的是推动开发者重构代码，消除循环依赖这一设计缺陷。最佳实践是通过提取公共逻辑到第三个 Bean 来解耦。
