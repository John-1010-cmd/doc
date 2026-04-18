---
title: Spring IoC容器原理
date: 2024-05-05
updated : 2024-05-05
categories:
- Spring
tags: 
  - Spring
  - 原理
description: 深入理解 Spring IoC 容器的设计哲学与实现原理，从 BeanFactory 到 ApplicationContext 的完整架构。
series: Spring 框架原理
series_order: 1
---

## IoC 与 DI 的概念

**控制反转（Inversion of Control）** 是一种设计原则，将对象的创建和依赖管理从程序代码中转移到外部容器。传统开发中，对象自己通过 `new` 创建依赖；IoC 模式下，容器负责创建和注入依赖。

**依赖注入（Dependency Injection）** 是 IoC 的一种实现方式，容器在运行时将依赖关系注入到对象中，而非对象自己去获取。

IoC 的核心价值：
- 解耦：组件之间不再硬编码依赖关系
- 可测试：依赖可通过 Mock 注入，便于单元测试
- 可扩展：通过配置切换实现，无需修改代码

## BeanFactory vs ApplicationContext

`BeanFactory` 是 Spring 最基础的容器接口，提供 Bean 的创建和获取能力：

```java
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));
MyService service = factory.getBean(MyService.class);
```

`ApplicationContext` 是 BeanFactory 的子接口，在基础容器之上增加了：
- 国际化支持（MessageSource）
- 事件发布（ApplicationEventPublisher）
- 资源加载（ResourceLoader）
- AOP 集成
- Bean 后置处理器的自动注册
- Environment 与 Profile 支持

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyService service = ctx.getBean(MyService.class);
```

实际开发中，ApplicationContext 是首选容器，BeanFactory 主要用于资源受限场景。

## BeanDefinition 的注册与解析

Spring 将每个 Bean 的配置信息封装为 `BeanDefinition` 对象，包含：
- Bean 的全限定类名
- 作用域（singleton / prototype）
- 依赖关系
- 初始化和销毁方法
- 构造函数参数和属性值

注册过程：

```java
// 注解方式注册
AnnotatedBeanDefinitionReader reader = new AnnotatedBeanDefinitionReader(registry);
reader.register(MyService.class);

// 编程方式注册
GenericBeanDefinition bd = new GenericBeanDefinition();
bd.setBeanClass(MyService.class);
bd.setScope(BeanDefinition.SCOPE_SINGLETON);
registry.registerBeanDefinition("myService", bd);
```

配置类通过 `@Configuration` + `@Bean` 解析为 BeanDefinition 的过程由 `ConfigurationClassPostProcessor` 完成。

## 容器启动流程：refresh() 方法

`AbstractApplicationContext.refresh()` 是容器启动的核心方法，共 12 个步骤：

```java
public void refresh() {
    // 1. 准备刷新，记录启动时间，设置状态标志
    prepareRefresh();
    // 2. 创建并获取 BeanFactory（子类实现）
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
    // 3. 对 BeanFactory 进行功能填充
    prepareBeanFactory(beanFactory);
    // 4. 子类后置处理（可注册特殊的 BeanPostProcessor）
    postProcessBeanFactory(beanFactory);
    // 5. 调用 BeanFactoryPostProcessor（修改 BeanDefinition）
    invokeBeanFactoryPostProcessors(beanFactory);
    // 6. 注册 BeanPostProcessor（拦截 Bean 的创建过程）
    registerBeanPostProcessors(beanFactory);
    // 7. 初始化消息源（国际化）
    initMessageSource();
    // 8. 初始化事件广播器
    initApplicationEventMulticaster();
    // 9. 子类特殊初始化（如 ThemeSource）
    onRefresh();
    // 10. 注册监听器
    registerListeners();
    // 11. 实例化所有非懒加载的单例 Bean
    finishBeanFactoryInitialization(beanFactory);
    // 12. 完成刷新，发布 ContextRefreshedEvent
    finishRefresh();
}
```

第 5 步和第 11 步是最关键的：前者修改 Bean 定义（如处理 `@Configuration`、`@ComponentScan`），后者真正创建所有单例 Bean。

## Bean 的实例化策略

Spring 支持三种实例化方式：

**构造器实例化**（最常见）：

```java
// 反射调用无参构造器
BeanUtils.instantiateClass(constructor);
```

**工厂方法实例化**：

```java
@Configuration
public class Config {
    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

**Supplier 实例化**（Spring 5.0+）：

```java
context.registerBean(MyService.class, () -> new MyServiceImpl());
```

实例化入口在 `AbstractAutowireCapableBeanFactory.createBean()`：

```java
// 实例化前，给 BeanPostProcessor 一个返回代理对象的机会
Object bean = resolveBeforeInstantiation(beanName, mbd);
if (bean != null) return bean;

// 真正创建 Bean
bean = doCreateBean(beanName, mbd, args);
```

## 依赖注入方式

**构造器注入**（推荐）：

```java
@Service
public class OrderService {
    private final UserRepository userRepo;
    private final PaymentService paymentService;

    public OrderService(UserRepository userRepo, PaymentService paymentService) {
        this.userRepo = userRepo;
        this.paymentService = paymentService;
    }
}
```

优点：依赖不可变、依赖不为空、完全初始化的状态。

**Setter 注入**：

```java
@Service
public class OrderService {
    private UserRepository userRepo;

    @Autowired
    public void setUserRepo(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
```

适用于可选依赖。

**字段注入**（不推荐）：

```java
@Service
public class OrderService {
    @Autowired
    private UserRepository userRepo;
}
```

缺点：无法声明为 final、隐藏依赖关系、难以单元测试。

## @Autowired 注入原理

`AutowiredAnnotationBeanPostProcessor` 处理 `@Autowired` 注解的注入过程：

1. **查找候选者**：根据类型在容器中查找所有匹配的 Bean
2. **筛选**：通过 `@Qualifier` 缩小范围
3. **优先级**：`@Primary` > `@Priority` > 按名称匹配
4. **注入**：通过反射设置字段或调用方法

```java
// 源码核心逻辑在 doResolveDependency()
// 先按类型查找，再按名称匹配
Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
```

当存在多个同类型 Bean 时，解析策略：
- `@Primary` 标注的 Bean 优先
- `@Qualifier` 指定 Bean 名称
- 字段名或参数名匹配 Bean 名称

## 三级缓存与循环依赖

Spring 通过三级缓存解决 Setter 注入的循环依赖问题：

```java
// 一级缓存：存放完全初始化好的 Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();
// 二级缓存：存放早期暴露的 Bean（已实例化但未初始化）
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>();
// 三级缓存：存放 Bean 的工厂对象（用于生成早期引用）
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>();
```

处理流程（以 A 依赖 B、B 依赖 A 为例）：
1. 创建 A，实例化后将其工厂放入三级缓存
2. 为 A 注入 B，发现 B 不存在，开始创建 B
3. 创建 B，实例化后将其工厂放入三级缓存
4. 为 B 注入 A，从三级缓存获取 A 的早期引用
5. B 完成初始化，放入一级缓存
6. A 完成 B 的注入，完成初始化，放入一级缓存

详细分析见 [Spring循环依赖](/2024/06/22/Spring循环依赖/)。

## 事件机制

Spring 事件机制基于观察者模式：

```java
// 定义事件
public class OrderCreatedEvent extends ApplicationEvent {
    private final Order order;
    public OrderCreatedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// 发布事件
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder(Order order) {
        publisher.publishEvent(new OrderCreatedEvent(this, order));
    }
}

// 监听事件
@Component
public class NotificationListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 发送通知
    }
}
```

事件发布通过 `ApplicationEventMulticaster` 广播给所有匹配的监听器，支持异步执行（`@Async`）和条件过滤（`condition`）。

## Aware 接口回调机制

Aware 接口让 Bean 获取容器的基础设施对象：

| 接口 | 注入内容 | 调用时机 |
|------|---------|---------|
| BeanNameAware | Bean 的名称 | 属性注入前 |
| BeanClassLoaderAware | 类加载器 | 属性注入前 |
| BeanFactoryAware | BeanFactory 实例 | 属性注入前 |
| ApplicationContextAware | ApplicationContext | 初始化阶段 |
| EnvironmentAware | Environment | 初始化阶段 |
| ResourceLoaderAware | ResourceLoader | 初始化阶段 |

```java
@Component
public class MyBean implements ApplicationContextAware {
    private ApplicationContext ctx;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.ctx = ctx;
    }
}
```

Aware 回调在 BeanPostProcessor 之前执行，通过 `AbstractApplicationContext.prepareBeanFactory()` 注册的 `ApplicationContextAwareProcessor` 完成。
