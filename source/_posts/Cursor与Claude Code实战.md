---
title: Cursor与Claude Code实战
date: 2026-02-05
updated : 2026-02-05
categories:
- AI Coding
tags: 
  - AI
  - 实战
description: Cursor 与 Claude Code 深度实战对比，掌握 Agent 模式下的高效编程工作流。
series: AI 辅助编程
series_order: 2
---

## Cursor 核心功能解析

### Composer

Composer 是 Cursor 最强大的功能之一，允许开发者通过自然语言描述来同时修改多个文件。

**使用方式：**

按下 `Ctrl+I`（或 `Cmd+I`）打开 Composer 面板，用自然语言描述你想要的改动。Cursor 会分析项目上下文，自动定位需要修改的文件，并生成差异化的修改建议。

**核心优势：**

- 支持多文件同时编辑，保持跨文件的逻辑一致性
- 提供差异化的预览视图，方便审查每一处改动
- 支持 Normal 和 Agent 两种模式切换

**Composer Normal vs Agent Mode：**

| 特性 | Normal Mode | Agent Mode |
|------|:-----------:|:----------:|
| 多文件编辑 | 是 | 是 |
| 自动搜索代码库 | 否 | 是 |
| 自动运行命令 | 否 | 是 |
| 自动修复错误 | 否 | 是 |
| 适合场景 | 明确的局部修改 | 复杂的全局重构 |

### Chat

Cursor Chat 提供了 IDE 内的 AI 对话能力，比 Copilot Chat 更强大。

**上下文引用语法：**

- `@file`：引用特定文件的内容
- `@folder`：引用整个目录结构
- `@web`：联网搜索最新信息
- `@docs`：引用项目文档
- `@git`：引用 Git 提交历史
- `@codebase`：让 AI 搜索整个代码库

**使用技巧：**

```
# 好的 Chat Prompt 示例
@codebase 找到所有使用 RedisTemplate 的地方，分析是否有连接泄漏的风险。
重点关注没有调用 connection.close() 的代码路径。

# 差的 Chat Prompt 示例
帮我看看 Redis 有没有问题
```

### Inline Edit

Inline Edit 是 Cursor 最常用的快捷操作，选中代码后按 `Cmd+K` 即可进行精确编辑。

**适用场景：**

- 快速重构选中的函数或代码块
- 添加错误处理逻辑
- 优化性能热点代码
- 修改命名和代码风格

### Agent Mode

Cursor 的 Agent Mode 是其最新的能力突破，让 AI 具备了自主完成复杂任务的能力。

**工作流程：**

1. 开发者描述任务目标
2. Agent 自动搜索相关代码文件
3. 分析依赖关系和影响范围
4. 制定修改计划并逐步执行
5. 运行测试验证修改正确性
6. 自动修复发现的问题

**配置 Agent 行为：**

```jsonc
// .cursorrules
// 控制Agent的行为边界

// 代码风格规则
- 使用 Java 17 特性，优先使用 record 和 sealed class
- 遵循领域驱动设计（DDD）分层架构
- Repository 层使用 Spring Data JPA
- Service 层使用事务注解，只读操作标记为 readOnly

// 安全规则
- 禁止在日志中输出敏感信息
- 所有外部输入必须进行参数校验
- SQL 查询必须使用参数化方式
- API 接口必须有权限校验

// 测试规则
- Service 层单元测试覆盖率不低于 80%
- Controller 层集成测试覆盖核心接口
- 使用 Testcontainers 进行数据库测试
```

## Claude Code 核心功能解析

### CLI 工具

Claude Code 以命令行工具的形式运行，适合习惯终端操作的开发者。

**安装与配置：**

```bash
# 安装 Claude Code
npm install -g @anthropic-ai/claude-code

# 在项目根目录启动
cd your-project
claude

# 常用启动参数
claude --model claude-sonnet-4-20250514  # 指定模型
claude --dangerously-skip-permissions    # 跳过权限确认（不推荐）
claude --verbose                         # 显示详细输出
```

**核心工具集：**

Claude Code 内置了丰富的工具，每个工具都有明确的职责：

| 工具 | 用途 |
|------|------|
| Read | 读取文件内容 |
| Write | 创建或覆盖文件 |
| Edit | 精确替换文件中的特定字符串 |
| Bash | 执行 Shell 命令 |
| Glob | 文件模式匹配搜索 |
| Grep | 内容搜索（基于 ripgrep） |
| TodoWrite | 任务跟踪和进度管理 |
| NotebookEdit | Jupyter Notebook 编辑 |

### Plan Mode

Plan Mode 是 Claude Code 处理复杂任务的核心能力。开启后，Claude 会先制定详细的执行计划，经用户确认后再逐步执行。

**启用方式：**

在对话中输入 `Shift+Tab` 切换到 Plan Mode，或在对话开头明确要求：

```
请先制定计划再执行：重构 OrderService，将订单状态管理抽取为独立的 State Machine
```

**Plan Mode 的优势：**

- 避免盲目修改导致的错误
- 让开发者有机会审查和调整方案
- 复杂任务可以分阶段执行
- 每个阶段的结果可以独立验证

**典型 Plan 输出：**

```
## 重构计划

### 阶段1：分析现有代码
- 读取 OrderService.java，识别所有状态转换逻辑
- 搜索所有调用 OrderService 状态相关方法的地方

### 阶段2：设计状态机
- 创建 OrderState 枚举
- 创建 OrderStateMachine 类
- 定义合法的状态转换规则

### 阶段3：实现重构
- 将状态转换逻辑从 OrderService 移至 OrderStateMachine
- 更新 OrderService 调用新的状态机
- 添加状态转换的事件通知

### 阶段4：验证
- 运行现有测试确保无回归
- 补充状态机相关的单元测试
- 检查并发安全性

确认执行？(y/n)
```

### Agent SDK

Claude Code 的 Agent SDK 允许开发者构建自定义的多 Agent 协作系统。

**核心概念：**

- **SubAgent**：独立的 AI Agent 实例，可以并行执行任务
- **Tool**：Agent 可以调用的工具
- **Prompt**：控制 Agent 行为的指令
- **Context**：Agent 可以访问的信息

**使用场景：**

1. **并行代码审查**：同时从安全、性能、可维护性三个维度审查代码
2. **多模块开发**：多个 Agent 并行开发独立的功能模块
3. **自动化测试**：一个 Agent 写代码，另一个 Agent 写测试

### Hooks 系统

Hooks 允许在 Claude Code 的工具调用前后插入自定义逻辑。

**Hook 类型：**

| Hook 类型 | 触发时机 | 典型用途 |
|-----------|---------|---------|
| PreToolUse | 工具执行前 | 参数验证、权限检查 |
| PostToolUse | 工具执行后 | 自动格式化、日志记录 |
| Stop | 会话结束时 | 最终验证、生成报告 |

**配置示例：**

```jsonc
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "npx prettier --write $FILE_PATH"
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "echo '请确认执行此命令' && exit 0"
      }
    ]
  }
}
```

## 工作流对比：可视化交互 vs 命令行驱动

### Cursor 的可视化工作流

Cursor 的交互完全在 IDE 内完成，开发者通过图形界面与 AI 协作。

**优势：**

- 所见即所得的编辑体验
- 差异化预览直观清晰
- 鼠标选区操作自然流畅
- 学习曲线低，新手友好

**劣势：**

- 受限于 IDE 环境的边界
- 复杂任务的可控性不如命令行
- 对 CI/CD 集成的支持有限

### Claude Code 的命令行工作流

Claude Code 通过终端交互，以文本形式展示所有信息。

**优势：**

- 与 Git、Docker、Kubernetes 等命令行工具无缝集成
- 可以编写脚本自动化重复任务
- 精确的权限控制，每个操作都可以审计
- 适合在远程服务器和 CI 环境中使用

**劣势：**

- 纯文本交互，不如图形界面直观
- 需要一定的命令行使用经验
- 代码预览不如 IDE 内的差异视图方便

### 如何选择

| 维度 | Cursor | Claude Code |
|------|:------:|:-----------:|
| 学习难度 | 低 | 中 |
| 日常编码 | 极佳 | 良好 |
| 复杂重构 | 良好 | 极佳 |
| 代码搜索 | 良好 | 极佳 |
| 自动化能力 | 有限 | 强大 |
| 远程开发 | 不支持 | 支持 |
| CI/CD 集成 | 不支持 | 支持 |
| 团队标准化 | 良好 | 极佳 |

## MCP（Model Context Protocol）协议

### 什么是 MCP

MCP 是 Anthropic 提出的开放协议，旨在标准化 AI 模型与外部工具的交互方式。通过 MCP，AI 可以连接到数据库、API、文件系统等外部资源。

### MCP 的架构

```
AI 应用（如 Claude Code / Cursor）
        |
    MCP Client
        |
    MCP Server（工具提供方）
        |
    外部资源（数据库 / API / 文件系统）
```

### MCP Server 类型

#### 内置 MCP Server

Claude Code 和 Cursor 都内置了一些 MCP Server：

- **文件系统 Server**：读写文件、搜索代码
- **终端 Server**：执行命令行操作
- **Git Server**：版本控制操作

#### 自定义 MCP Server

开发者可以根据项目需求创建自定义 MCP Server：

```typescript
// 示例：自定义数据库查询 MCP Server
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({
  name: "database-query",
  version: "1.0.0",
});

server.tool(
  "query_database",
  { sql: z.string().describe("SQL 查询语句") },
  async ({ sql }) => {
    // 只允许 SELECT 查询，防止数据修改
    if (!sql.trim().toUpperCase().startsWith("SELECT")) {
      return { content: [{ type: "text", text: "只允许 SELECT 查询" }] };
    }
    const result = await executeQuery(sql);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
);
```

### 在 Claude Code 中配置 MCP

```jsonc
// .claude/settings.json
{
  "mcpServers": {
    "database": {
      "command": "node",
      "args": ["./mcp-servers/database-query.js"]
    },
    "jira": {
      "command": "node",
      "args": ["./mcp-servers/jira-integration.js"],
      "env": {
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    }
  }
}
```

### 在 Cursor 中配置 MCP

Cursor 同样支持 MCP 协议，配置方式类似：

```jsonc
// .cursor/mcp.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/project"]
    }
  }
}
```

## 实战场景1：用 Cursor 从零搭建 SpringBoot 项目

### 步骤1：项目初始化

在 Cursor Chat 中描述项目需求：

```
@web 请帮我创建一个 SpringBoot 3.2 项目，要求：
- Java 17 + Gradle
- 包含 Spring Web、Spring Data JPA、PostgreSQL Driver、Lombok
- 采用 DDD 分层架构：controller、service、repository、domain
- 配置文件使用 YAML 格式
- 包含 Docker Compose 配置（PostgreSQL + Redis）
```

Cursor 会自动生成完整的项目骨架，包括 `build.gradle`、配置文件、Docker Compose 和基础包结构。

### 步骤2：领域模型设计

使用 Composer 设计领域模型：

```
创建一个电商订单系统的领域模型，包括：
- Order：订单聚合根，包含订单项、收货地址、支付信息
- OrderItem：订单项，关联商品和数量
- OrderStatus：订单状态枚举（PENDING、PAID、SHIPPED、COMPLETED、CANCELLED）
- 使用 JPA 注解映射数据库
- 使用 Lombok 减少样板代码
```

### 步骤3：接口开发

使用 Agent Mode 开发完整的 CRUD 接口：

```
@folder src/main/java/com/example/order 为 Order 实体开发完整的 REST API：
1. POST /api/orders - 创建订单
2. GET /api/orders/{id} - 查询订单详情
3. GET /api/orders - 分页查询订单列表
4. PUT /api/orders/{id}/status - 更新订单状态
5. DELETE /api/orders/{id} - 取消订单

要求：
- 使用统一的 ApiResponse 包装返回结果
- 参数校验使用 Jakarta Validation
- 异常处理使用 @ControllerAdvice
- 分页参数使用 Pageable
```

### 步骤4：测试与调试

让 Cursor 生成测试代码并运行：

```
为 OrderService 编写单元测试，要求：
- 使用 JUnit 5 + Mockito
- 覆盖创建订单、查询订单、更新状态的核心逻辑
- 包含异常场景测试（订单不存在、状态转换不合法）
- 使用 @DisplayName 标注测试用例
```

## 实战场景2：用 Claude Code 重构遗留代码

### 步骤1：代码分析

```bash
$ claude
> 分析 src/legacy/ 目录下的代码结构，识别以下问题：
> 1. 重复代码片段
> 2. 过长的函数（超过 50 行）
> 3. 循环依赖
> 4. 硬编码的魔法值
> 5. 缺少错误处理的地方
```

Claude Code 会使用 Grep 和 Glob 工具搜索整个代码库，生成详细的代码质量报告。

### 步骤2：制定重构计划

```
> 开启 Plan Mode。基于上面的分析结果，制定一个分阶段的重构计划：
> - 阶段1：提取重复代码为公共工具方法
> - 阶段2：拆分过长函数
> - 阶段3：引入常量替换魔法值
> - 阶段4：补充错误处理
> - 阶段5：运行全量测试验证
```

### 步骤3：逐步执行

确认计划后，Claude Code 会逐步执行每个阶段：

```
正在执行阶段1：提取重复代码...

发现以下重复模式：
- DateUtil.format() 在 12 个文件中重复实现
- ResponseBuilder 在 8 个 Controller 中重复定义

创建公共工具类：
- common/util/DateUtil.java
- common/response/ResponseBuilder.java

更新引用：修改 20 个文件的 import 语句

运行测试... 全部通过 ✓
```

### 步骤4：验证与提交

```
> 运行全量测试，确保重构没有引入回归

> 为本次重构生成提交信息，遵循 Conventional Commits 格式
```

## 上下文管理

### .cursorrules 配置

`.cursorrules` 文件放在项目根目录，为 Cursor 提供项目级的行为规则。

```jsonc
// .cursorrules
{
  "rules": [
    "项目使用 Next.js 14 + App Router",
    "样式使用 Tailwind CSS，不使用 CSS Modules",
    "状态管理使用 Zustand，不使用 Redux",
    "API 调用使用 Server Actions",
    "组件命名使用 PascalCase",
    "工具函数命名使用 camelCase",
    "文件命名使用 kebab-case"
  ]
}
```

### CLAUDE.md 配置

`CLAUDE.md` 文件为 Claude Code 提供项目级的上下文信息。

```markdown
# CLAUDE.md

## 项目概述
这是一个电商平台后端系统，使用 SpringBoot 3.2 + Java 17。

## 技术栈
- 框架：SpringBoot 3.2
- 数据库：PostgreSQL 15
- 缓存：Redis 7
- 消息队列：RabbitMQ
- 构建工具：Gradle

## 代码规范
- 使用领域驱动设计（DDD）分层架构
- 统一使用 ApiResponse<T> 包装返回结果
- 异常使用自定义 BusinessException
- 日志使用 SLF4J + Logback

## 禁止操作
- 不要直接操作 EntityManager
- 不要在 Controller 层写业务逻辑
- 不要在循环中调用数据库查询
- 不要忽略异常
```

### AGENTS.md 配置

`AGENTS.md` 用于定义多 Agent 协作的分工策略。

```markdown
# AGENTS.md

## Agent 职责分工

### planner Agent
- 负责制定实现计划
- 分析依赖关系和风险评估
- 输出分阶段的实施方案

### tdd-guide Agent
- 负责测试驱动开发流程
- 编写测试用例
- 验证实现覆盖率

### code-reviewer Agent
- 代码质量审查
- 安全漏洞检测
- 性能问题识别
```

## 多文件编辑能力对比

### Cursor 的多文件编辑

Cursor 通过 Composer 实现多文件编辑，核心机制是：

1. 分析用户意图，确定涉及的文件列表
2. 为每个文件生成差异化的修改建议
3. 在统一的 Composer 面板中展示所有改动
4. 用户可以逐个 Accept 或 Reject 每处改动

**优势：** 可视化的差异对比，操作直觉化。
**劣势：** 当涉及文件超过 10 个时，审查所有改动的效率会下降。

### Claude Code 的多文件编辑

Claude Code 通过 Read + Edit 工具的组合实现多文件编辑：

1. 先读取所有相关文件的内容
2. 分析文件间的依赖关系
3. 使用 Edit 工具逐文件进行精确替换
4. 每次修改都会明确标注 old_string 和 new_string

**优势：** 修改精确可控，每次替换都可追溯。
**劣势：** 无法一次性预览所有修改的差异视图。

## 调试与错误修复流程

### Cursor 调试流程

1. 遇到错误时，将错误信息粘贴到 Cursor Chat
2. 使用 `@terminal` 引用终端输出
3. Cursor 分析错误原因并提供修复建议
4. 使用 Inline Edit 快速应用修复

### Claude Code 调试流程

1. 遇到错误时，直接将错误信息告诉 Claude Code
2. Claude Code 使用 Grep 搜索相关代码
3. 使用 Read 工具读取相关文件
4. 使用 Edit 工具精确修复问题
5. 使用 Bash 运行测试验证修复

```
> 运行测试时报错：NullPointerException in OrderService.processOrder line 42

Claude Code 执行步骤：
1. Read OrderService.java
2. Grep "processOrder" 查找所有调用者
3. 分析第42行的变量，发现 orderId 可能为 null
4. Edit: 在调用 processOrder 前添加 null 检查
5. Bash: mvn test -pl order-service
6. 测试通过 ✓
```

## 团队协作与最佳实践

### 统一配置管理

团队应统一管理 AI 编程工具的配置文件：

```gitignore
# .gitignore
.claude/settings.local.json    # 个人配置不提交
.cursor/mcp.local.json         # 本地 MCP 配置不提交
```

```
# 应提交到版本控制的文件
.cursorrules          # 项目级 Cursor 规则
CLAUDE.md             # 项目级 Claude Code 规则
AGENTS.md             # Agent 协作配置
.claude/settings.json # 共享的 Claude Code 配置
```

### Code Review 流程中整合 AI

1. 提交 PR 前让 AI 做一次预审查
2. 使用 Claude Code 分析 PR 的改动范围和潜在风险
3. 将 AI 审查结果作为 PR Comment 附带提交
4. 人工审查时重点关注 AI 标记的风险点

### 知识传递

将 AI 生成的代码规范和最佳实践沉淀到项目的 Rules 文件中：

- 每次解决复杂 Bug 后，更新 CLAUDE.md 中的相关注意事项
- 发现新的编码规范时，更新 .cursorrules
- 重构完成后，更新 AGENTS.md 中的职责定义

## 总结

Cursor 和 Claude Code 各有擅长的领域。Cursor 在日常编码和快速原型开发方面体验更优，其可视化交互让代码编辑更直观。Claude Code 在复杂重构、项目级搜索、自动化和团队标准化方面更具优势，其命令行特性和强大的工具集适合处理系统性工程任务。

最佳实践是两者结合使用：用 Cursor 做日常开发和快速迭代，用 Claude Code 处理复杂重构和自动化任务。两者的 Rules 文件可以互相参考，形成统一的团队开发规范。
