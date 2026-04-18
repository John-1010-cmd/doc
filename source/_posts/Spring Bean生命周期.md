---
title: Spring Bean生命周期
date: 2024-06-08
updated : 2024-06-08
categories:
- Spring
tags: 
  - Spring
  - 原理
description: 完整梳理 Spring Bean 从创建到销毁的全生命周期，掌握各扩展点的调用时机。
series: Spring 框架原理
series_order: 3
---

## Bean 完整生命周期流程

Spring Bean 从创建到销毁经历以下阶段：

```
BeanDefinition 注册
    ↓
实例化（Instantiation）
    ↓
属性赋值（Populate）
    ↓
Aware 回调
    ↓
BeanPostProcessor#postProcessBeforeInitialization
    ↓
初始化（Initialization）
    ├── @PostConstruct
    ├── InitializingBean#afterPropertiesSet
    └── init-method
    ↓
BeanPostProcessor#postProcessAfterInitialization
    ↓
Bean 就绪，可使用
    ↓
销毁（Destruction）
    ├── @PreDestroy
    ├── DisposableBean#destroy
    └── destroy-method
```

## 实例化阶段

Bean 的实例化由 `AbstractAutowireCapableBeanFactory.createBeanInstance()` 完成：

```java
// 1. 如果有 Supplier，使用 Supplier 创建
if (mbd.getSupplier() != null) {
    return obtainFromSupplier(mbd.getSupplier(), beanName);
}

// 2. 如果有 factoryMethodName，使用工厂方法
if (mbd.getFactoryMethodName() != null) {
    return instantiateUsingFactoryMethod(beanName, mbd, args);
}

// 3. 解析构造函数，通过反射实例化
Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
return autowireConstructor(beanName, mbd, ctors, args);
```

### InstantiationAwareBeanPostProcessor

在实例化前后可以拦截：

```java
@Component
public class MyInstantiationProcessor implements InstantiationAwareBeanPostProcessor {

    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
        // 实例化之前，返回非 null 可跳过 Spring 的实例化
        return null;
    }

    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) {
        // 实例化之后，属性注入之前
        // 返回 false 可跳过属性注入
        return true;
    }

    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        // 修改或添加属性值（如 @Autowired 的处理）
        return pvs;
    }
}
```

## 属性赋值阶段

`populateBean()` 方法完成依赖注入：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // 1. 调用 InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation
    // 2. 调用 InstantiationAwareBeanPostProcessor#postProcessProperties（处理 @Autowired）
    // 3. 应用属性值（PropertyValues）
}
```

`@Autowired` 的处理由 `AutowiredAnnotationBeanPostProcessor.postProcessProperties()` 完成：
1. 扫描字段和方法上的 `@Autowired`、`@Value` 注解
2. 通过 `beanFactory.resolveDependency()` 查找依赖 Bean
3. 通过反射注入

## 初始化阶段

初始化阶段是扩展点最集中的阶段：

### Aware 回调

```java
// 在 AbstractAutowireCapableBeanFactory.initializeBean() 中
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(this);
        }
    }
}
```

### 各 Aware 接口的调用顺序

Spring 通过不同的 `BeanPostProcessor` 处理不同的 Aware 接口：

| 顺序 | Aware 接口 | 处理者 |
|------|-----------|--------|
| 1 | BeanNameAware | invokeAwareMethods() |
| 2 | BeanClassLoaderAware | invokeAwareMethods() |
| 3 | BeanFactoryAware | invokeAwareMethods() |
| 4 | EnvironmentAware | ApplicationContextAwareProcessor |
| 5 | EmbeddedValueResolverAware | ApplicationContextAwareProcessor |
| 6 | ResourceLoaderAware | ApplicationContextAwareProcessor |
| 7 | ApplicationEventPublisherAware | ApplicationContextAwareProcessor |
| 8 | MessageSourceAware | ApplicationContextAwareProcessor |
| 9 | ApplicationContextAware | ApplicationContextAwareProcessor |

### BeanPostProcessor

`BeanPostProcessor` 是 Spring 最强大的扩展机制：

```java
public interface BeanPostProcessor {
    // 初始化前调用
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }
    // 初始化后调用（AOP 代理在此创建）
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

常见的 BeanPostProcessor：
- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`
- `CommonAnnotationBeanPostProcessor`：处理 `@PostConstruct`、`@PreDestroy`
- `AnnotationAwareAspectJAutoProxyCreator`：创建 AOP 代理
- `AsyncAnnotationBeanPostProcessor`：处理 `@Async`

### @PostConstruct

由 `CommonAnnotationBeanPostProcessor`（JSR-250）处理：

```java
@Component
public class MyService {
    @PostConstruct
    public void init() {
        // 依赖注入完成后执行初始化逻辑
        System.out.println("Bean 初始化完成");
    }
}
```

### InitializingBean

```java
@Component
public class MyService implements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        // 所有属性设置完成后调用
    }
}
```

### init-method

```java
@Component(initMethod = "customInit")
public class MyService {
    public void customInit() {
        // 自定义初始化方法
    }
}
```

三种初始化方式的执行顺序：`@PostConstruct` → `InitializingBean.afterPropertiesSet()` → `init-method`

## BeanPostProcessor 与 BeanFactoryPostProcessor

两者在生命周期中的位置不同：

```
容器启动
    ↓
BeanFactoryPostProcessor（修改 BeanDefinition）
    ├── PropertySourcesPlaceholderConfigurer（处理 ${} 占位符）
    ├── ConfigurationClassPostProcessor（处理 @Configuration）
    └── CustomBeanFactoryPostProcessor
    ↓
Bean 实例化
    ↓
BeanPostProcessor（修改 Bean 实例）
    ├── AutowiredAnnotationBeanPostProcessor
    ├── CommonAnnotationBeanPostProcessor
    └── AsyncAnnotationBeanPostProcessor
```

关键区别：
- `BeanFactoryPostProcessor` 在 Bean 实例化前执行，操作的是 BeanDefinition
- `BeanPostProcessor` 在 Bean 实例化后执行，操作的是 Bean 实例

## 常用扩展点实战

### 自定义 BeanPostProcessor

```java
@Component
public class TimeCostBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(TimeCost.class)) {
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    long start = System.currentTimeMillis();
                    Object result = method.invoke(bean, args);
                    System.out.println(beanName + "." + method.getName() +
                        " 耗时: " + (System.currentTimeMillis() - start) + "ms");
                    return result;
                }
            );
        }
        return bean;
    }
}
```

### 自定义 BeanFactoryPostProcessor

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        BeanDefinition bd = factory.getBeanDefinition("myService");
        bd.setScope(ConfigurableBeanFactory.SCOPE_PROTOTYPE);
    }
}
```

## 销毁阶段

当容器关闭时，单例 Bean 按注册的逆序销毁：

```java
@Component
public class MyService implements DisposableBean {
    @PreDestroy
    public void preDestroy() {
        // 释放资源（数据库连接、文件句柄等）
    }

    @Override
    public void destroy() {
        // DisposableBean 的销毁方法
    }
}
```

销毁顺序：`@PreDestroy` → `DisposableBean.destroy()` → `destroy-method`

## Prototype vs Singleton 的生命周期差异

| 阶段 | Singleton | Prototype |
|------|-----------|-----------|
| 实例化 | 容器启动时（非懒加载） | 每次获取时 |
| 属性注入 | 容器负责 | 容器负责 |
| 初始化回调 | 容器负责 | 容器负责 |
| 销毁回调 | 容器关闭时 | **不负责**，由客户端管理 |

Prototype Bean 的特殊注意：
- Spring 不管理 Prototype Bean 的完整生命周期
- 如果 Singleton Bean 注入了 Prototype Bean，Prototype Bean 不会每次创建新的（除非使用 `@Lookup` 或 `ObjectFactory`）

```java
@Component
public class SingletonBean {
    // 每次调用都获取新的 Prototype Bean
    @Lookup
    public PrototypeBean getPrototypeBean() {
        return null; // Spring 会覆盖此方法
    }
}
```

## 生命周期中的常见陷阱

### 1. BeanPostProcessor 注册时机

BeanPostProcessor 本身也是 Bean，但它的注册必须在其他 Bean 创建之前完成。如果在 BeanPostProcessor 中注入了普通 Bean，可能导致该 Bean 无法被其他 BeanPostProcessor 处理。

### 2. FactoryBean 的生命周期

`FactoryBean` 是一种特殊的 Bean，它本身是 Singleton，但 `getObject()` 返回的对象由 FactoryBean 控制：

```java
public class MyFactoryBean implements FactoryBean<MyService> {
    @Override
    public MyService getObject() {
        return new MyService();
    }
    @Override
    public Class<?> getObjectType() {
        return MyService.class;
    }
}
```

获取 FactoryBean 本身需要加 `&` 前缀：`context.getBean("&myFactoryBean")`。

### 3. @Async 与生命周期

在 `@PostConstruct` 中调用 `@Async` 方法不会异步执行，因为代理对象可能尚未创建完成。
