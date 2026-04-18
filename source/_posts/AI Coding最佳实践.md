---
title: AI Coding最佳实践
date: 2026-02-10
updated : 2026-02-10
categories:
- AI Coding
tags: 
  - AI
  - 实战
description: AI 辅助编程的最佳实践总结，涵盖 Prompt 技巧、代码审查与效率提升方法论。
series: AI 辅助编程
series_order: 3
---

## 编写高质量 AI Prompt 的原则

### 核心原则：精确、具体、有上下文

与 AI 编程工具交互的本质是「上下文工程」。AI 的输出质量直接取决于输入信息的质量和完整度。以下是编写高质量 Prompt 的核心原则。

### 原则一：明确角色和目标

告诉 AI 它需要扮演的角色和任务目标，而非模糊的指令。

```
# 差的 Prompt
帮我写一个登录功能

# 好的 Prompt
你是一位 Java 后端高级工程师。请实现一个 JWT 认证的登录接口，要求：
- 技术栈：SpringBoot 3.2 + Spring Security 6
- Token 类型：Access Token（15分钟） + Refresh Token（7天）
- 密码加密：BCrypt
- 返回格式：统一的 ApiResponse<LoginResponse>
- 包含参数校验和错误处理
```

### 原则二：提供充分的上下文

AI 无法猜测你项目中的约定和约束，需要显式告知。

```
# 差的 Prompt
写一个查询用户的接口

# 好的 Prompt
在 UserService 中添加分页查询方法，上下文信息：
- 项目使用 MyBatis-Plus 作为 ORM
- 实体类是 User，包含 id、name、email、status、createdAt 字段
- 查询参数包括：keyword（模糊匹配 name 或 email）、status（精确匹配）、
  startDate/endDate（createdAt 范围查询）
- 使用 LambdaQueryWrapper 构建查询条件
- 返回类型为 IPage<UserVO>，UserVO 是 User 的简化视图
- 排序默认按 createdAt DESC
```

### 原则三：约束边界条件

明确告诉 AI 什么是不能做的，比告诉它能做什么同样重要。

```
# 好的 Prompt 包含约束
实现文件上传功能，约束条件：
- 只允许上传图片（jpg、png、gif），最大 5MB
- 文件名使用 UUID 重命名，保留原始扩展名
- 存储路径：/uploads/{year}/{month}/{day}/{uuid}.{ext}
- 不使用外部 OSS 服务，存储在本地文件系统
- 不需要图片压缩和缩略图生成
```

### 原则四：指定输出格式

明确你期望的输出结构和格式。

```
# 指定输出格式
请按照以下格式输出代码审查结果：
1. 严重程度：CRITICAL / HIGH / MEDIUM / LOW
2. 问题描述
3. 涉及文件和行号
4. 修复建议（包含代码示例）
5. 修复后的预期效果
```

### 原则五：分步骤引导复杂任务

复杂任务不要一次性描述，分步骤引导 AI 完成。

```
# 分步骤 Prompt
请分 3 个步骤完成这个任务：

步骤1：先分析现有的 OrderController.java，列出所有接口和它们的职责
步骤2：识别违反单一职责原则的地方，说明原因
步骤3：提出重构方案，将 OrderController 拆分为多个专职 Controller

每个步骤完成后等待我的确认再继续下一步。
```

## 任务拆分：大任务如何拆解为 AI 可处理的小步骤

### 为什么需要拆分

AI 编程工具的能力边界是有限的。一次性给出过大的任务会导致：

- 输出质量下降：AI 可能遗漏关键细节
- 不可控性增加：难以审查大量生成的代码
- 调试困难：出错时难以定位问题根源
- 上下文溢出：复杂任务的上下文可能超出模型的处理能力

### 拆分策略

#### 按架构层次拆分

```
大任务：实现一个用户注册系统

拆分为：
1. 设计 User 实体和数据库表结构
2. 实现 Repository 层的数据访问
3. 实现 Service 层的注册逻辑（参数校验、密码加密、唯一性检查）
4. 实现 Controller 层的 REST 接口
5. 实现邮件验证功能
6. 编写单元测试
7. 编写集成测试
```

#### 按功能模块拆分

```
大任务：重构支付模块

拆分为：
1. 分析现有支付模块的代码结构和依赖关系
2. 定义支付策略接口（PaymentStrategy）
3. 实现支付宝支付策略
4. 实现微信支付策略
5. 实现银联支付策略
6. 实现 PaymentContext（策略调度器）
7. 迁移现有代码到新架构
8. 运行全量测试验证
```

#### 按风险等级拆分

```
大任务：优化数据库查询性能

拆分为（按风险从低到高）：
1. [低风险] 添加缺失的数据库索引
2. [低风险] 优化 N+1 查询问题
3. [中风险] 引入查询缓存
4. [中风险] 优化复杂 JOIN 查询
5. [高风险] 分库分表改造
```

### 拆分的粒度控制

**原则：每个小任务的预期输出不超过 100 行代码。**

- 过粗：一个任务涉及超过 3 个文件的修改
- 合适：一个任务修改 1-2 个文件，代码量 50-100 行
- 过细：一个任务只是改一个变量名

## 代码生成：如何描述需求、约束上下文、指定技术栈

### 需求描述模板

一个好的代码生成 Prompt 应该包含以下要素：

```
## 任务描述
[一句话概括要实现的功能]

## 技术栈
- 语言/框架版本
- 相关依赖库

## 输入输出
- 输入：参数列表、数据格式
- 输出：返回类型、数据格式

## 业务规则
- 核心逻辑描述
- 边界条件
- 异常场景

## 约束条件
- 性能要求
- 安全要求
- 编码规范

## 参考代码
[如果有类似功能的现有代码，粘贴作为参考]
```

### 实际示例：生成 SpringBoot 接口

```
## 任务描述
实现一个商品搜索接口，支持关键词搜索和多条件筛选

## 技术栈
- SpringBoot 3.2 + Java 17
- Spring Data Elasticsearch
- 已有 ProductDocument 实体（字段：id、name、category、price、brand、stock、createdAt）

## 输入输出
- 输入：SearchRequest { keyword: String, category: String, minPrice: BigDecimal,
         maxPrice: BigDecimal, brands: List<String>, page: int, size: int, sort: String }
- 输出：ApiResponse<Page<ProductVO>>

## 业务规则
- keyword 搜索 name 字段，使用模糊匹配（minimum_should_match: 75%）
- price 范围筛选使用 range query
- brands 使用 terms query
- 排序支持 price_asc、price_desc、created_desc
- 库存为 0 的商品默认不展示
- 搜索结果需要高亮显示 name 中的匹配关键词

## 约束条件
- 搜索响应时间不超过 200ms
- 使用 @Scheduled 定时同步 MySQL 数据到 Elasticsearch
- 索引名称使用别名机制，支持零停机重建索引
```

### 上下文约束技巧

#### 限定技术选型

```
# 明确禁止某些技术选择
不要使用：
- 不要用 Lombok @Data，使用标准 getter/setter
- 不要用 MapStruct，手动编写映射逻辑
- 不要用 Spring Data JPA 的 @Query，使用 Specification

请使用：
- 使用 Builder 模式构建复杂对象
- 使用 Optional 处理可能为空的返回值
- 使用 Stream API 处理集合操作
```

#### 提供现有代码风格

```
# 粘贴现有代码作为风格参考
请参照以下代码风格编写：
```java
// 现有的 Controller 风格
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Product", description = "商品管理接口")
public class ProductController {

    private final ProductService productService;

    @GetMapping("/{id}")
    @Operation(summary = "查询商品详情")
    public ApiResponse<ProductVO> getProduct(@PathVariable Long id) {
        return ApiResponse.success(productService.findById(id));
    }
}
```

## 调试协作：错误报告格式、日志提供方式

### 高效的错误报告格式

向 AI 报告错误时，提供完整的信息可以大幅缩短调试时间。

```
## 错误报告

### 环境信息
- Java 版本：17.0.8
- SpringBoot 版本：3.2.1
- 数据库：PostgreSQL 15.4

### 错误现象
调用 POST /api/orders 接口创建订单时返回 500 错误

### 错误信息
```
org.springframework.dao.DataIntegrityViolationException:
could not execute statement [
    ERROR: null value in column "order_no" violates not-null constraint
]
```

### 相关代码
```java
// OrderService.java line 45-52
public OrderVO createOrder(CreateOrderRequest request) {
    Order order = new Order();
    order.setUserId(request.getUserId());
    order.setItems(request.getItems());
    // 注意：这里没有设置 orderNo
    orderRepository.save(order);
    return convertToVO(order);
}
```

### 预期行为
创建订单时应自动生成唯一的 orderNo（格式：ORD-yyyyMMdd-6位随机数）
```

### 日志提供的最佳实践

#### 提供足够的日志上下文

```
# 差的日志提供方式
报错了，日志如下：
java.lang.NullPointerException

# 好的日志提供方式
以下是完整的错误日志，包含异常堆栈和上下文：
```
2026-02-10 14:32:15.123 INFO  [http-nio-8080-exec-1] c.e.o.controller.OrderController : 收到创建订单请求: CreateOrderRequest(userId=123, items=[ItemRequest(productId=456, quantity=2)])
2026-02-10 14:32:15.156 DEBUG [http-nio-8080-exec-1] c.e.o.service.OrderService : 开始处理订单，用户ID: 123
2026-02-10 14:32:15.178 DEBUG [http-nio-8080-exec-1] c.e.o.service.OrderService : 商品信息查询完成: Product(id=456, name=iPhone 15, price=7999.00, stock=10)
2026-02-10 14:32:15.182 ERROR [http-nio-8080-exec-1] c.e.o.service.OrderService : 订单处理异常
java.lang.NullPointerException: Cannot invoke "com.example.order.domain.Order.getOrderNo()" because the return value of "com.example.order.domain.OrderService.generateOrderNo()" is null
    at com.example.order.service.OrderService.createOrder(OrderService.java:48)
    at com.example.order.controller.OrderController.createOrder(OrderController.java:35)
    ...
```
```

#### 过滤无关日志

对于大量日志，先筛选出关键信息再提供给 AI：

- 使用时间范围缩小日志范围
- 使用 Thread 名称或 Request ID 追踪请求链路
- 优先提供 ERROR 和 WARN 级别的日志
- 附加前后各 5 行的 DEBUG 日志作为上下文

## 代码审查：让 AI 做 Code Review 的正确姿势

### 审查维度设计

让 AI 从多个维度审查代码，而非笼统地要求「帮我看看代码有没有问题」。

```
请从以下维度审查这段代码：

1. 正确性：逻辑是否正确，边界条件是否处理
2. 安全性：是否存在 SQL 注入、XSS、敏感信息泄露风险
3. 性能：是否存在 N+1 查询、不必要的全表扫描、内存泄漏
4. 可维护性：命名是否清晰、函数是否过长、职责是否单一
5. 异常处理：是否覆盖所有异常场景，错误信息是否友好
6. 并发安全：是否有竞态条件、线程安全问题

每个维度使用 CRITICAL/HIGH/MEDIUM/LOW 标注严重程度。
```

### 增量审查策略

不要让 AI 一次性审查整个文件，而是聚焦于变更部分：

```
# 使用 Git Diff 聚焦变更
请审查以下 Git Diff 中的代码变更：

```diff
@@ -45,7 +45,12 @@ public class OrderService {
-    public OrderVO createOrder(CreateOrderRequest request) {
+    public OrderVO createOrder(CreateOrderRequest request, String userId) {
+        if (request.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
+            throw new BusinessException("订单金额必须大于0");
+        }
         Order order = new Order();
-        order.setUserId(request.getUserId());
+        order.setUserId(userId);
         order.setOrderNo(generateOrderNo());
```

重点关注：
1. 新增参数是否需要向后兼容
2. 金额校验逻辑是否完整
3. userId 的来源是否可信
```

### 审查结果的处理流程

1. **CRITICAL 级别**：立即修复，不可延迟
2. **HIGH 级别**：当前迭代内修复
3. **MEDIUM 级别**：记录到技术债务清单，下次迭代处理
4. **LOW 级别**：作为代码风格改进的参考

## 渐进式开发：先骨架后细节的工作流

### 工作流概述

渐进式开发的核心思想是：先让 AI 生成代码的骨架结构，确认方向正确后再逐步填充细节。

### 阶段1：生成接口骨架

```
请为订单管理模块生成接口骨架代码，要求：
- 只包含类定义、方法签名和返回类型
- 方法体用 TODO 注释标记，不实现具体逻辑
- 包含完整的注解定义（@RestController、@GetMapping 等）
- 包含接口文档注解（@Operation、@Tag）
```

### 阶段2：确认骨架并调整

审查骨架代码，确认架构设计和接口划分是否合理。如果需要调整，在骨架阶段修改成本最低。

### 阶段3：填充核心逻辑

```
请实现 OrderService.createOrder 方法的核心逻辑：
- 参数校验（商品是否存在、库存是否充足）
- 创建订单主记录
- 创建订单明细记录
- 扣减库存

暂时不需要实现：支付处理、物流通知、积分计算
```

### 阶段4：补充非功能性需求

```
基于已实现的 createOrder 方法，补充：
- 分布式锁（防止重复下单）
- 事务管理（确保数据一致性）
- 操作日志记录
- 发送领域事件
```

### 阶段5：完善测试

```
为 createOrder 方法编写测试用例，要求：
- 正常场景：成功创建订单
- 异常场景：商品不存在、库存不足、重复提交
- 并发场景：同一商品同时下单
- 数据验证：验证订单号生成规则、金额计算精度
```

## 知识沉淀：Rules 文件、记忆系统、CLAUDE.md 最佳实践

### CLAUDE.md 的最佳实践

CLAUDE.md 是 Claude Code 的项目级配置文件，用于沉淀团队知识和编码规范。

#### 结构化内容组织

```markdown
# CLAUDE.md

## 项目概述
[一段话描述项目的业务背景和技术定位]

## 技术栈
[列出主要的技术组件和版本]

## 项目结构
[描述核心目录结构和各模块的职责]

## 编码规范
[具体的编码约定和规则]

## 禁止操作
[明确列出 AI 不能做的事情]

## 常见陷阱
[项目中已知的坑和需要注意的事项]
```

#### 内容更新时机

- 每次解决了一个非平凡的 Bug 后，在「常见陷阱」中记录
- 每次引入新技术或框架时，在「技术栈」中更新
- 每次修改项目结构后，更新「项目结构」描述
- 每次团队 Code Review 发现新规范时，添加到「编码规范」

### .cursorrules 的最佳实践

```jsonc
// .cursorrules 应该简洁明了
{
  "rules": [
    // 技术栈约束
    "使用 React 18 + TypeScript 5",
    "样式使用 Tailwind CSS",
    "状态管理使用 Zustand",

    // 编码规范
    "组件使用函数式组件 + Hooks",
    "Props 使用 interface 定义，不用 type",
    "事件处理函数命名使用 handle 前缀",

    // 文件组织
    "每个组件一个文件",
    "组件文件名使用 PascalCase",
    "工具函数文件名使用 camelCase",

    // 测试要求
    "使用 Vitest + Testing Library",
    "测试文件放在 __tests__ 目录下",
    "测试命名使用 describe + it 结构"
  ]
}
```

### AGENTS.md 的最佳实践

```markdown
# AGENTS.md

## Agent 职责定义

### planner
- 触发时机：复杂功能需求、架构重构
- 职责：分析需求、识别风险、制定分阶段实施计划
- 输出：包含任务列表、依赖关系、风险评估的实施计划

### tdd-guide
- 触发时机：新功能开发、Bug 修复
- 职责：编写测试用例、驱动实现、验证覆盖率
- 输出：测试代码 + 覆盖率报告

### code-reviewer
- 触发时机：代码编写完成后
- 职责：代码质量审查、安全审查、性能审查
- 输出：按严重程度排序的问题列表 + 修复建议
```

### 知识沉淀的维护策略

1. **定期审查**：每月审查一次 Rules 文件，清理过时内容
2. **增量更新**：每次发现问题后立即更新，不要等到「以后再说」
3. **团队共识**：Rules 文件的修改应经过团队讨论
4. **版本管理**：Rules 文件提交到 Git，通过 PR Review 进行变更管理

## 常见陷阱

### 陷阱一：过度依赖 AI

**表现：**

- 不理解 AI 生成的代码就直接提交
- 遇到 Bug 不自己分析，直接让 AI 修复
- 不学习基础原理，完全依赖 AI 生成

**后果：**

- 系统出现问题时无法排查
- AI 生成错误的代码时无法识别
- 技术能力停滞不前

**应对策略：**

- 强制自己理解每一段 AI 生成的代码
- 提交前进行 Code Review，即使代码是 AI 生成的
- 定期手动编写一些核心代码，保持编程能力

### 陷阱二：盲目信任 AI 输出

**表现：**

- 认为 AI 生成的代码一定是正确的
- 不运行测试就信任 AI 的修改
- 不验证 AI 提供的 API 用法

**后果：**

- 引入隐藏的 Bug
- 使用了已废弃的 API
- 代码在特定场景下表现异常

**应对策略：**

- 所有 AI 生成的代码都要经过测试验证
- 查阅官方文档确认 API 用法
- 特别关注边界条件和异常处理

### 陷阱三：安全风险

**表现：**

- 在 Prompt 中粘贴了包含密钥的配置文件
- AI 生成的代码使用了不安全的加密算法
- 未对 AI 生成的 SQL 进行注入检查

**应对策略：**

```markdown
# 在 CLAUDE.md 中添加安全规则

## 安全规则
- 禁止在 Prompt 中粘贴任何包含密钥、密码、Token 的文件
- 所有 SQL 查询必须使用参数化方式
- 密码存储必须使用 BCrypt，禁止使用 MD5/SHA1
- 敏感日志输出必须脱敏处理
- 外部输入必须进行参数校验和 XSS 过滤
```

### 陷阱四：上下文窗口浪费

**表现：**

- 在对话中粘贴了大量无关代码
- 频繁切换话题导致上下文丢失
- 一次性提供过多文件内容

**应对策略：**

- 只提供与当前任务相关的代码
- 每个独立任务开启新的对话
- 使用 `@file` 引用代替粘贴代码
- 长对话中定期总结关键决策

## AI 辅助测试：生成测试用例、Mock 数据、边界条件

### 生成测试用例

**策略：先描述测试场景，再让 AI 生成代码。**

```
请为 TransferService.transfer 方法编写测试用例，覆盖以下场景：

## 正常场景
1. 同币种转账成功
2. 跨币种转账（自动汇率转换）

## 异常场景
3. 转出账户余额不足
4. 转出账户已冻结
5. 转入账户不存在
6. 转账金额为负数或零
7. 单笔转账金额超过限额
8. 当日累计转账金额超过限额

## 并发场景
9. 同一账户同时转出给多人
10. 同一账户同时被多人转入

## 技术要求
- 使用 JUnit 5 + Mockito
- 使用 @ParameterizedTest 处理金额边界值
- Mock AccountRepository 和 ExchangeRateService
- 使用 BigDecimal 处理金额，精度为 2 位小数
```

### 生成 Mock 数据

```
请生成以下 Mock 数据，用于测试商品搜索功能：

1. 50 条商品数据，包含以下字段：
   - id、name、category、price、brand、stock、status
   - category 分布：电子产品 15、服装 15、食品 10、家居 10
   - price 范围：10-10000，覆盖各种价位
   - stock 包含 0（缺货）的场景

2. 10 条测试用的搜索关键词：
   - 精确匹配关键词
   - 模糊匹配关键词
   - 包含特殊字符的关键词
   - 超长关键词（100+ 字符）
   - 空字符串和 null

格式：JSON 文件
```

### 边界条件测试

```
请分析以下方法并补充边界条件测试：

```java
public BigDecimal calculateDiscount(Order order, User user) {
    BigDecimal discount = BigDecimal.ZERO;

    // 新用户首单优惠
    if (user.getOrderCount() == 0) {
        discount = discount.add(new BigDecimal("50"));
    }

    // 会员等级折扣
    discount = discount.add(order.getTotalAmount()
        .multiply(user.getLevel().getDiscountRate()));

    // 满减优惠
    if (order.getTotalAmount().compareTo(new BigDecimal("500")) >= 0) {
        discount = discount.add(new BigDecimal("30"));
    }

    return discount;
}
```

需要测试的边界条件：
1. orderCount = 0（首次下单）和 orderCount = 1（非首次）
2. totalAmount 刚好等于 500（临界值）
3. totalAmount = 499.99（不满足满减）
4. totalAmount = 0 或 null
5. discountRate 为 0 或负数
6. 最终折扣金额是否可能超过订单总金额
```

## 人机协作的黄金比例

### 何时使用 AI

**适合 AI 处理的任务：**

- 重复性代码编写（CRUD、DTO 转换）
- 已有明确规范的实现（按照设计文档编码）
- 大范围的代码搜索和分析
- 测试用例生成
- 代码格式化和风格统一
- 文档和注释编写
- 已知模式的 Bug 修复

**适合人工处理的任务：**

- 需求分析和用户调研
- 系统架构设计
- 关键业务逻辑的决策
- 复杂的性能优化（需要深入理解瓶颈）
- 安全架构设计
- 技术选型评估
- 代码审查（AI 生成代码的最终把关）

### 协作模式

#### 模式一：AI 生成 + 人工审查

适用：日常功能开发

1. 描述需求，让 AI 生成初版代码
2. 人工审查代码质量和业务正确性
3. 修改不满意的部分
4. 运行测试验证

#### 模式二：人工设计 + AI 实现

适用：复杂功能开发

1. 人工完成架构设计和接口定义
2. 让 AI 按照设计文档实现细节
3. 人工验证实现是否符合设计意图
4. 迭代调整

#### 模式三：AI 分析 + 人工决策

适用：技术决策

1. 让 AI 分析各种方案的优劣
2. 人工基于分析结果做出最终决策
3. 让 AI 按照决策执行实现

## 提升 AI 输出质量的关键技巧

### 技巧一：提供示例代码

AI 根据示例代码的风格和模式生成代码，质量远高于从零生成。

```
请参照以下代码风格，实现 ProductController 的搜索接口：

[粘贴现有的风格规范的 Controller 代码]
```

### 技巧二：明确非功能性需求

不要只描述功能性需求，同时指定性能、安全、可维护性要求。

```
实现商品搜索接口，额外要求：
- 响应时间 < 200ms（P99）
- 支持 1000 QPS 并发
- 搜索结果缓存 5 分钟
- 慢查询日志阈值 100ms
```

### 技巧三：要求 AI 解释代码

让 AI 解释它生成的代码，不仅能帮助理解，还能让 AI 自己发现潜在问题。

```
请解释你刚才生成的代码中：
1. 为什么要使用 ConcurrentHashMap 而不是 HashMap？
2. computeIfAbsent 方法的原子性保证是什么？
3. 如果在高并发场景下，这段代码可能有什么问题？
```

### 技巧四：迭代优化

不要期望 AI 一次生成完美的代码，通过迭代逐步优化。

```
# 第一轮
> 实现一个简单的LRU缓存

# 第二轮（基于第一轮的输出）
> 优化上一版的实现，添加：
> 1. 线程安全支持
> 2. 过期时间机制
> 3. 容量变更通知

# 第三轮
> 添加监控指标：
> 1. 命中率统计
> 2. 驱逐计数
> 3. 平均访问延迟
```

### 技巧五：利用 Plan Mode

在执行复杂任务前，先让 AI 制定计划。这有两个好处：

1. 你可以审查计划的合理性，避免 AI 走错方向
2. AI 在制定计划的过程中会更深入地思考问题

```
请先制定实施计划，不要立即修改代码。

任务：将现有的单体应用拆分为微服务架构。

要求：
1. 分析现有的模块依赖关系
2. 识别可以独立部署的服务边界
3. 评估每个服务的数据一致性需求
4. 制定分阶段的迁移计划
```

## 总结

AI 辅助编程的最佳实践可以归纳为一个核心原则：**把 AI 当作一个能力很强但需要明确指导的初级开发者**。它需要清晰的需求描述、充分的上下文信息、明确的约束条件和及时的反馈纠正。

关键要点：

1. **Prompt 质量**决定输出质量，投入时间编写高质量的 Prompt
2. **任务拆分**是处理复杂需求的关键策略
3. **知识沉淀**通过 Rules 文件实现，是团队效率持续提升的基础
4. **代码审查**不可省略，AI 生成的代码同样需要严格审查
5. **人机协作**找到合适的分工比例，发挥各自优势

本系列的三篇文章从工具概览到实战对比再到最佳实践，形成了完整的 AI 辅助编程知识体系。希望这些内容能帮助你在日常开发中更高效地使用 AI 编程工具。
