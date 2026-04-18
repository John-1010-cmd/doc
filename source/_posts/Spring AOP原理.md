---
title: Spring AOP原理剖析
date: 2024-05-20
updated : 2024-05-20
categories:
- Spring
tags: 
  - Spring
  - 原理
description: 从动态代理到字节码织入，全面解析 Spring AOP 的实现原理与高级用法。
series: Spring 框架原理
series_order: 2
---

## AOP 核心概念

面向切面编程（AOP）将横切关注点（如日志、事务、安全）从业务逻辑中分离出来，通过预编译或运行时动态代理实现功能增强。

核心术语：

| 术语 | 说明 | 示例 |
|------|------|------|
| Aspect（切面） | 横切关注点的模块化 | 日志切面、事务切面 |
| Join Point（连接点） | 程序执行的某个点 | 方法调用、异常抛出 |
| Pointcut（切点） | 匹配连接点的表达式 | `execution(* com.example.service.*.*(..))` |
| Advice（通知） | 在切点执行的动作 | @Before、@After、@Around |
| Target（目标对象） | 被代理的原始对象 | UserServiceImpl |
| Weaving（织入） | 将切面应用到目标对象 | 运行时动态代理 |

## JDK 动态代理 vs CGLIB 代理

Spring 根据目标类是否实现接口来选择代理策略：

**JDK 动态代理**：目标类实现了接口时使用

```java
// JDK 动态代理核心：InvocationHandler
Object proxy = Proxy.newProxyInstance(
    target.getClass().getClassLoader(),
    target.getClass().getInterfaces(),
    (proxyObj, method, args) -> {
        // 前置增强
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);
        // 后置增强
        System.out.println("After: " + method.getName());
        return result;
    }
);
```

限制：只能代理接口方法，无法代理类方法。

**CGLIB 代理**：目标类未实现接口时使用，通过生成子类实现代理

```java
// CGLIB 通过生成子类实现代理
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(TargetClass.class);
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    System.out.println("Before: " + method.getName());
    Object result = proxy.invokeSuper(obj, args);
    System.out.println("After: " + method.getName());
    return result;
});
TargetClass proxy = (TargetClass) enhancer.create();
```

Spring 默认策略（`spring.aop.proxy-target-class`）：
- SpringBoot 2.x 起默认使用 CGLIB（`proxy-target-class=true`）
- 如果目标类实现了接口且 `proxy-target-class=false`，则使用 JDK 动态代理

## ProxyFactoryBean 与代理创建

`ProxyFactory` 是 Spring AOP 创建代理的核心类：

```java
ProxyFactory factory = new ProxyFactory(targetObject);
factory.addInterface(UserService.class);
factory.addAdvice(new MethodBeforeAdvice() {
    @Override
    public void before(Method method, Object[] args, Object target) {
        System.out.println("Before: " + method.getName());
    }
});
UserService proxy = (UserService) factory.getProxy();
```

`ProxyFactory.getProxy()` 的执行流程：
1. 创建 `AdvisedSupport` 对象，保存切面配置
2. 调用 `AopProxyFactory.createAopProxy()`
3. 根据条件选择 `JdkDynamicAopProxy` 或 `ObjenesisCglibAopProxy`
4. 返回代理对象

## 通知类型

### @Before（前置通知）

在目标方法执行前运行：

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint jp) {
    System.out.println("调用: " + jp.getSignature().getName());
}
```

### @AfterReturning（返回通知）

方法正常返回后执行：

```java
@AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
public void logAfterReturning(JoinPoint jp, Object result) {
    System.out.println("返回值: " + result);
}
```

### @AfterThrowing（异常通知）

方法抛出异常时执行：

```java
@AfterThrowing(pointcut = "execution(* com.example.service.*.*(..))", throwing = "ex")
public void logAfterThrowing(JoinPoint jp, Exception ex) {
    System.out.println("异常: " + ex.getMessage());
}
```

### @After（后置通知）

无论方法是否异常都会执行（类似 finally）：

```java
@After("execution(* com.example.service.*.*(..))")
public void logAfter(JoinPoint jp) {
    System.out.println("方法执行完毕: " + jp.getSignature().getName());
}
```

### @Around（环绕通知）

最强大的通知类型，可控制是否执行目标方法：

```java
@Around("execution(* com.example.service.*.*(..))")
public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    try {
        Object result = pjp.proceed();  // 执行目标方法
        return result;
    } finally {
        long cost = System.currentTimeMillis() - start;
        System.out.println(pjp.getSignature().getName() + " 耗时: " + cost + "ms");
    }
}
```

## 切点表达式

### execution（最常用）

匹配方法执行：

```
execution(修饰符? 返回值 包名.类名.方法名(参数) 异常?)
```

```java
// 匹配 service 包下所有类的所有方法
@Pointcut("execution(* com.example.service.*.*(..))")

// 匹配所有 public 方法
@Pointcut("execution(public * *(..))")

// 匹配以 save 开头的方法
@Pointcut("execution(* save*(..))")
```

### @annotation

匹配带有指定注解的方法：

```java
@Pointcut("@annotation(com.example.annotation.Loggable)")
```

### within

匹配指定类型内的所有方法：

```java
@Pointcut("within(com.example.service.*)")
```

### args

匹配参数类型：

```java
@Pointcut("args(java.lang.String, ..)")
```

### 组合切点

```java
@Pointcut("execution(* com.example.service.*.*(..)) && !execution(* com.example.service.Internal*.*(..))")
```

## Advised 接口与拦截器链

代理对象实现了 `Advised` 接口，持有所有通知的列表。方法调用时，通知按拦截器链模式执行：

```
MethodInvocation
    → AfterAdvice
        → AroundAdvice
            → BeforeAdvice
                → 目标方法
            → AfterReturningAdvice
        → AroundAdvice (返回后部分)
    → AfterAdvice (finally)
```

拦截器链的核心实现是 `ReflectiveMethodInvocation`：

```java
public Object proceed() throws Throwable {
    // 拦截器链已执行完毕，调用目标方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }
    // 获取下一个拦截器并执行
    Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
}
```

## @EnableAspectJAutoProxy 与自动代理创建器

`@EnableAspectJAutoProxy` 注解向容器注册 `AnnotationAwareAspectJAutoProxyCreator`：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}
```

`AnnotationAwareAspectJAutoProxyCreator` 是一个 `BeanPostProcessor`，在 Bean 初始化后检查是否需要创建代理：
1. 查找所有 `@Aspect` 注解的 Bean
2. 解析切面中的通知方法
3. 判断当前 Bean 是否匹配任何切点
4. 如果匹配，创建代理对象并替换原始 Bean

## 多个切面的执行顺序

通过 `@Order` 或实现 `Ordered` 接口控制切面执行顺序：

```java
@Aspect
@Component
@Order(1)
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() { /* ... */ }
}

@Aspect
@Component
@Order(2)
public class TransactionAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void txBefore() { /* ... */ }
}
```

执行顺序（Order 值越小越先执行）：
- 前置通知：Order 1 → Order 2
- 目标方法执行
- 后置通知：Order 2 → Order 1

即"洋葱模型"：外层切面先进入后退出，内层切面后进入先退出。

## AOP 与事务的协作原理

`@Transactional` 注解通过 `TransactionInterceptor` 实现：

```java
// 简化的调用链
@Around
public Object invoke(MethodInvocation invocation) throws Throwable {
    // 1. 获取事务属性
    TransactionAttribute attr = this.transactionAttributeSource.getTransactionAttribute(method, targetClass);
    // 2. 开启事务
    TransactionStatus status = transactionManager.getTransaction(attr);
    try {
        // 3. 执行目标方法
        Object result = invocation.proceed();
        // 4. 提交事务
        transactionManager.commit(status);
        return result;
    } catch (Throwable ex) {
        // 5. 回滚事务
        transactionManager.rollback(status);
        throw ex;
    }
}
```

事务切面通常设置最高优先级（`@Order(Ordered.LOWEST_PRECEDENCE)`），确保事务包裹在其他切面之外，这样其他切面的异常也能触发回滚。
