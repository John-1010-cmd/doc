---
title: AI Agent开发指南
date: 2025-06-15
updated : 2025-06-15
categories:
- AI
tags: 
  - AI
  - 实战
description: AI Agent 开发核心原理与实战，从 Function Calling 到多 Agent 协作。
series: AI 技术实践
series_order: 4
---

## Agent 概念：感知、推理、行动循环

### 什么是 AI Agent

AI Agent 是一个能够**自主感知环境、进行推理决策、执行行动**的智能体。与简单的 Prompt-Response 模式不同，Agent 具备以下核心特征：

- **自主性**：能够独立做出决策，而非仅仅响应指令
- **目标导向**：围绕明确的目标组织行为
- **工具使用**：能够调用外部工具扩展自身能力
- **环境感知**：能够获取并理解环境信息
- **学习适应**：能够从经验中改进行为

### Agent 的核心循环

```
        ┌─────────┐
        │  目标/任务 │
        └────┬────┘
             │
             ▼
     ┌──────────────┐
     │   感知 (Perceive) │◄────── 观察环境状态
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │   推理 (Reason)  │◄────── 分析、规划、决策
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │   行动 (Act)     │◄────── 调用工具、执行操作
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │   观察 (Observe) │◄────── 获取行动结果
     └──────┬───────┘
            │
            └──── 是否完成？── 否 ──► 回到"感知"
                │
                是
                ▼
           任务完成
```

这个循环（Perceive-Reason-Act-Observe）是所有 Agent 框架的基础模式。Agent 通过不断循环这个流程，逐步逼近目标。

### Agent vs 传统 LLM 应用

| 维度 | 传统 LLM 应用 | AI Agent |
|------|-------------|---------|
| 交互模式 | 单轮请求-响应 | 多轮循环 |
| 工具使用 | 无或有限 | 核心能力 |
| 决策方式 | 直接生成 | 推理后决策 |
| 状态管理 | 无状态 | 有状态（记忆） |
| 错误处理 | 返回错误 | 自主重试和恢复 |
| 复杂任务 | 一步完成 | 分步规划和执行 |

## Function Calling 机制

Function Calling 是 LLM 与外部世界交互的核心接口，让模型能够调用预定义的工具。

### JSON Schema 定义

每个工具使用 JSON Schema 描述其接口：

```json
{
  "name": "search_database",
  "description": "在指定的数据库表中搜索数据",
  "parameters": {
    "type": "object",
    "properties": {
      "table": {
        "type": "string",
        "description": "要查询的数据库表名",
        "enum": ["users", "orders", "products"]
      },
      "conditions": {
        "type": "object",
        "description": "查询条件，键为字段名，值为匹配值",
        "properties": {
          "status": {"type": "string"},
          "created_after": {"type": "string", "format": "date"}
        }
      },
      "limit": {
        "type": "integer",
        "description": "返回结果数量上限",
        "default": 10,
        "maximum": 100
      }
    },
    "required": ["table"]
  }
}
```

### 工具注册与调用

```python
import openai
import json

#​ 定义工具集
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位"
                    }
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "搜索互联网获取最新信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜索查询"
                    }
                },
                "required": ["query"]
            }
        }
    }
]


def run_agent(user_message, max_turns=5):
    """运行 Agent 循环"""
    messages = [{"role": "user", "content": user_message}]

    for turn in range(max_turns):
        # 调用 LLM（带工具定义）
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=messages,
            tools=tools,
            tool_choice="auto",
        )

        message = response.choices[0].message
        messages.append(message)

        # 如果没有工具调用，说明 Agent 认为可以直接回答
        if not message.tool_calls:
            return message.content

        # 执行每个工具调用
        for tool_call in message.tool_calls:
            function_name = tool_call.function.name
            function_args = json.loads(tool_call.function.arguments)

            # 调用实际函数
            result = execute_function(function_name, function_args)

            # 将结果添加到消息历史
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result, ensure_ascii=False),
            })

    return "达到最大交互轮数，任务未完成。"


def execute_function(name, args):
    """执行具体的工具函数"""
    functions = {
        "get_weather": lambda a: {"temperature": 25, "condition": "晴天"},
        "search_web": lambda a: {"results": [f"关于'{a['query']}'的搜索结果"]},
    }
    return functions[name](args)


#​ 使用示例
result = run_agent("今天北京天气怎么样？适合户外活动吗？")
print(result)
```

### Function Calling 的关键设计原则

1. **描述清晰**：description 要详细说明函数的功能、适用场景和注意事项
2. **参数验证**：利用 JSON Schema 的类型、枚举、范围等约束
3. **单一职责**：每个函数只做一件事，保持原子性
4. **幂等性**：相同输入应产生相同输出（对 GET 类操作）

## ReAct 模式：Reasoning + Acting

ReAct 是 Agent 最常用的推理-行动框架，将思考过程和工具调用交替进行。

### ReAct 的核心流程

```
User: 查找 Python 3.12 的新特性，并写一篇总结

Agent 思考: 我需要先搜索 Python 3.12 的新特性
Agent 行动: search("Python 3.12 new features")
观察: [搜索结果：Python 3.12 新增了类型参数语法、改进的 f-string、性能提升...]

Agent 思考: 我获取到了基本信息，让我搜索更多细节
Agent 行动: search("Python 3.12 type parameter syntax details")
观察: [搜索结果：PEP 695 引入了类型参数语法...]

Agent 思考: 现在我有足够的信息来写总结了
Agent 回答: Python 3.12 的主要新特性包括：...
```

### ReAct 的代码实现

```python
class ReActAgent:
    """ReAct 模式的 Agent 实现"""

    def __init__(self, llm, tools, max_iterations=10):
        self.llm = llm
        self.tools = {tool.name: tool for tool in tools}
        self.max_iterations = max_iterations

    def run(self, task):
        """执行任务"""
        prompt = f"""你是一个智能助手。使用以下格式完成任务：

Question: 需要回答的问题
Thought: 你对下一步行动的思考
Action: 要使用的工具名称
Action Input: 工具的输入参数（JSON 格式）
Observation: 工具的返回结果
... (Thought/Action/Action Input/Observation 可以重复多次)
Thought: 我现在知道最终答案了
Final Answer: 对原始问题的最终回答

可用工具：{list(self.tools.keys())}

Question: {task}
"""
        conversation = [{"role": "user", "content": prompt}]

        for _ in range(self.max_iterations):
            response = self.llm.invoke(conversation)
            conversation.append({"role": "assistant", "content": response})

            # 解析 Action
            action, action_input = self._parse_action(response)

            if action is None:
                # 没有 Action，检查是否有 Final Answer
                final_answer = self._parse_final_answer(response)
                if final_answer:
                    return final_answer
                continue

            # 执行工具
            if action in self.tools:
                observation = self.tools[action].execute(action_input)
            else:
                observation = f"错误：未知工具 '{action}'"

            # 将观察结果添加到对话
            conversation.append({
                "role": "user",
                "content": f"Observation: {observation}",
            })

        return "达到最大迭代次数，任务未完成。"

    def _parse_action(self, text):
        """从响应中解析 Action 和 Action Input"""
        import re
        action_match = re.search(r'Action:\s*(\w+)', text)
        input_match = re.search(r'Action Input:\s*(.+)', text)
        if action_match and input_match:
            return action_match.group(1), input_match.group(1).strip()
        return None, None

    def _parse_final_answer(self, text):
        """从响应中解析 Final Answer"""
        import re
        match = re.search(r'Final Answer:\s*(.+)', text, re.DOTALL)
        return match.group(1).strip() if match else None
```

## Plan-and-Execute 模式

Plan-and-Execute 模式将任务分解为**规划**和**执行**两个独立阶段。

### 工作原理

```
用户任务: "分析竞品并生成市场报告"

┌─────────────┐
│   规划阶段    │
│ Planner LLM  │
└──────┬──────┘
       │ 生成计划
       ▼
┌──────────────────────────────────┐
│ Step 1: 搜索竞品列表              │
│ Step 2: 获取每个竞品的详细信息      │
│ Step 3: 对比分析功能差异           │
│ Step 4: 分析市场定位和定价策略      │
│ Step 5: 生成综合报告              │
└──────┬───────────────────────────┘
       │ 逐步执行
       ▼
┌─────────────┐
│   执行阶段    │
│ Executor Agent│ × 5 步
└──────┬──────┘
       │ 汇总
       ▼
┌─────────────┐
│   输出报告    │
└─────────────┘
```

### 代码实现

```python
class PlanAndExecuteAgent:
    """Plan-and-Execute 模式"""

    def __init__(self, planner_llm, executor_agent):
        self.planner = planner_llm
        self.executor = executor_agent

    def run(self, task):
        """执行任务"""
        # 阶段 1：生成计划
        plan = self._create_plan(task)

        # 阶段 2：逐步执行
        results = []
        for i, step in enumerate(plan):
            print(f"执行步骤 {i+1}/{len(plan)}: {step}")

            # 将之前的结果作为上下文
            context = self._build_context(task, plan, results, i)
            step_result = self.executor.execute(step, context)
            results.append(step_result)

            # 动态调整：根据执行结果更新计划
            if self._need_replan(results):
                plan = self._replan(task, plan, results)

        # 阶段 3：汇总输出
        return self._synthesize(task, plan, results)

    def _create_plan(self, task):
        """使用 Planner 生成执行计划"""
        prompt = f"""为以下任务创建一个详细的执行计划。

任务：{task}

要求：
1. 将任务分解为具体的、可执行的步骤
2. 每个步骤应该是原子性的
3. 步骤之间有明确的依赖关系
4. 输出为 JSON 数组格式

输出格式：
["步骤1", "步骤2", "步骤3", ...]
"""
        response = self.planner.invoke(prompt)
        import json
        return json.loads(response)

    def _need_replan(self, results):
        """判断是否需要重新规划"""
        # 如果最后一步执行失败，可能需要调整计划
        if results and isinstance(results[-1], dict) and results[-1].get("error"):
            return True
        return False

    def _replan(self, task, original_plan, results):
        """根据执行结果重新规划"""
        prompt = f"""原始任务：{task}
原始计划：{original_plan}
已执行步骤的结果：{results}

请根据实际执行情况，调整剩余的计划步骤。
输出调整后的完整计划（JSON 数组）。"""
        response = self.planner.invoke(prompt)
        import json
        return json.loads(response)
```

### ReAct vs Plan-and-Execute

| 维度 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| 决策方式 | 每步即时决策 | 先规划后执行 |
| 适应性 | 高（可随时调整） | 中（需重新规划） |
| 效率 | 可能走弯路 | 计划优化后更高效 |
| 可解释性 | 逐步推理可见 | 计划结构清晰 |
| 适用场景 | 开放性探索任务 | 结构化复杂任务 |

## 记忆系统

记忆系统让 Agent 能够在交互过程中保持和利用历史信息。

### 短期记忆

短期记忆存储当前对话的上下文：

```python
class ShortTermMemory:
    """短期记忆：当前对话上下文"""

    def __init__(self, max_messages=20):
        self.messages = []
        self.max_messages = max_messages

    def add(self, role, content):
        """添加消息"""
        self.messages.append({"role": role, "content": content})
        # 超出限制时移除最早的消息（保留 System Prompt）
        if len(self.messages) > self.max_messages:
            self.messages = [self.messages[0]] + self.messages[-(self.max_messages-1):]

    def get_context(self):
        """获取当前上下文"""
        return self.messages

    def clear(self):
        """清空记忆"""
        self.messages = []
```

### 长期记忆

长期记忆持久化存储跨会话的知识和经验：

```python
import chromadb
from sentence_transformers import SentenceTransformer


class LongTermMemory:
    """长期记忆：基于向量数据库的持久化存储"""

    def __init__(self):
        self.encoder = SentenceTransformer("BAAI/bge-small-zh-v1.5")
        self.client = chromadb.PersistentClient(path="./agent_memory")
        self.collection = self.client.get_or_create_collection("memories")

    def store(self, content, memory_type="experience", metadata=None):
        """存储记忆"""
        embedding = self.encoder.encode([content]).tolist()
        mem_id = f"mem_{self.collection.count() + 1}"

        self.collection.add(
            ids=[mem_id],
            embeddings=embedding,
            documents=[content],
            metadatas=[{"type": memory_type, **(metadata or {})}],
        )

    def recall(self, query, top_k=5):
        """回忆相关记忆"""
        query_embedding = self.encoder.encode([query]).tolist()
        results = self.collection.query(
            query_embeddings=query_embedding,
            n_results=top_k,
        )
        return results["documents"][0]

    def consolidate(self):
        """记忆整合：合并相似记忆，遗忘过时信息"""
        # 获取所有记忆
        all_memories = self.collection.get(include=["documents", "metadatas"])

        # 合并相似记忆
        # （实际实现中可使用 LLM 来判断和合并）
        pass
```

### 工作记忆

工作记忆是 Agent 当前任务执行过程中的临时信息：

```python
class WorkingMemory:
    """工作记忆：当前任务的临时状态"""

    def __init__(self):
        self.state = {}
        self.task_history = []

    def set(self, key, value):
        """设置状态"""
        self.state[key] = value

    def get(self, key, default=None):
        """获取状态"""
        return self.state.get(key, default)

    def add_action(self, action, result):
        """记录动作历史"""
        self.task_history.append({
            "action": action,
            "result": result,
            "timestamp": __import__("time").time(),
        })

    def get_summary(self):
        """获取工作记忆摘要"""
        return {
            "current_state": self.state,
            "actions_taken": len(self.task_history),
            "last_action": self.task_history[-1] if self.task_history else None,
        }
```

## 工具设计原则

### 原子性

每个工具应该只完成一个明确的功能：

```python
#​ 好的设计：原子性工具
class SearchTool:
    """搜索文档"""
    def execute(self, query, top_k=5):
        return search_index(query, top_k)

class SummarizeTool:
    """总结文档"""
    def execute(self, document):
        return summarize(document)

class TranslateTool:
    """翻译文本"""
    def execute(self, text, target_language):
        return translate(text, target_language)

#​ 差的设计：一个工具做太多事
class SwissArmyKnifeTool:
    """搜索、总结、翻译一体化"""
    def execute(self, query, summarize=True, translate_to=None, top_k=5):
        # 职责不清晰，难以测试和复用
        pass
```

### 可组合性

工具之间可以灵活组合，形成复杂工作流：

```python
class ToolRegistry:
    """工具注册中心"""

    def __init__(self):
        self.tools = {}

    def register(self, name, tool, description, parameters_schema):
        self.tools[name] = {
            "tool": tool,
            "description": description,
            "parameters": parameters_schema,
        }

    def execute(self, name, **kwargs):
        if name not in self.tools:
            raise ValueError(f"未知工具: {name}")
        return self.tools[name]["tool"].execute(**kwargs)

    def get_tools_description(self):
        """生成工具列表描述（供 LLM 使用）"""
        descriptions = []
        for name, info in self.tools.items():
            descriptions.append(
                f"- {name}: {info['description']}\n"
                f"  参数: {info['parameters']}"
            )
        return "\n".join(descriptions)


#​ 注册工具
registry = ToolRegistry()
registry.register("search", SearchTool(), "搜索文档", {"query": "str", "top_k": "int"})
registry.register("summarize", SummarizeTool(), "总结文档", {"document": "str"})
registry.register("translate", TranslateTool(), "翻译文本", {"text": "str", "target_language": "str"})
```

### 容错性

工具调用可能出现各种异常，需要健壮的错误处理：

```python
class RobustToolWrapper:
    """带容错的工具包装器"""

    def __init__(self, tool, max_retries=3, timeout=30):
        self.tool = tool
        self.max_retries = max_retries
        self.timeout = timeout

    def execute(self, **kwargs):
        for attempt in range(self.max_retries):
            try:
                # 参数验证
                validated_args = self._validate_args(kwargs)

                # 执行（带超时）
                result = self._execute_with_timeout(validated_args)

                # 结果验证
                return self._validate_result(result)

            except ValidationError as e:
                return {"error": f"参数验证失败: {e}"}
            except TimeoutError:
                if attempt < self.max_retries - 1:
                    continue
                return {"error": "执行超时"}
            except Exception as e:
                if attempt < self.max_retries - 1:
                    continue
                return {"error": f"执行失败: {str(e)}"}

    def _execute_with_timeout(self, args):
        """带超时的执行"""
        import signal
        # 实现超时逻辑
        return self.tool.execute(**args)

    def _validate_args(self, kwargs):
        """参数验证"""
        return kwargs

    def _validate_result(self, result):
        """结果验证"""
        return result
```

## Multi-Agent 协作

### 分工模式

将复杂任务分配给不同专业领域的 Agent：

```python
class AgentTeam:
    """多 Agent 团队"""

    def __init__(self):
        self.agents = {}

    def add_agent(self, name, role, system_prompt, tools):
        self.agents[name] = {
            "role": role,
            "system_prompt": system_prompt,
            "tools": tools,
        }

    def delegate(self, task):
        """根据任务类型分配给合适的 Agent"""
        # 分析任务，选择最合适的 Agent
        assigned_agent = self._select_agent(task)

        # 执行任务
        result = self._run_agent(assigned_agent, task)

        # 如果需要协作，调度其他 Agent
        if result.get("needs_review"):
            reviewer = self.agents.get("reviewer")
            review_result = self._run_agent(reviewer, result["content"])
            return review_result

        return result

    def _select_agent(self, task):
        """选择最合适的 Agent"""
        # 可以用 LLM 来判断任务应该分配给哪个 Agent
        for name, agent in self.agents.items():
            if agent["role"] in task.lower():
                return name
        return "general"  # 默认 Agent


#​ 创建团队
team = AgentTeam()
team.add_agent("researcher", "research", "你是研究专家，负责信息收集和分析。", [search_tool, web_tool])
team.add_agent("writer", "writing", "你是写作专家，负责内容创作和编辑。", [write_tool, edit_tool])
team.add_agent("reviewer", "review", "你是审查专家，负责质量检查和事实核查。", [fact_check_tool])
```

### 通信机制

Agent 之间的通信需要一个共享的消息通道：

```python
class AgentMessageBus:
    """Agent 间通信的消息总线"""

    def __init__(self):
        self.channels = {}  # channel_name -> [messages]

    def publish(self, channel, sender, message):
        """发布消息到指定频道"""
        if channel not in self.channels:
            self.channels[channel] = []
        self.channels[channel].append({
            "sender": sender,
            "content": message,
            "timestamp": __import__("time").time(),
        })

    def subscribe(self, channel, agent_name):
        """订阅频道"""
        # 返回该频道的所有历史消息
        return self.channels.get(channel, [])

    def broadcast(self, sender, message):
        """广播消息到所有频道"""
        for channel in self.channels:
            self.publish(channel, sender, message)
```

### 冲突解决

多 Agent 协作时可能出现意见分歧，需要冲突解决机制：

```python
class ConflictResolver:
    """冲突解决器"""

    @staticmethod
    def resolve_by_vote(proposals):
        """投票决策"""
        from collections import Counter
        votes = Counter(p["decision"] for p in proposals)
        winner = votes.most_common(1)[0][0]
        return winner

    @staticmethod
    def resolve_by_authority(proposals, authority_order):
        """权威优先决策"""
        for authority in authority_order:
            for p in proposals:
                if p["agent"] == authority and p.get("decision"):
                    return p["decision"]

    @staticmethod
    def resolve_by_llm(proposals, task):
        """使用 LLM 仲裁"""
        prompt = f"""任务：{task}

不同 Agent 的提案：
{proposals}

请综合各方的意见，给出最佳决策。"""
        # 调用 LLM 做最终决策
        pass
```

## Agent 框架对比

### LangChain Agent

```python
from langchain.agents import create_openai_tools_agent, AgentExecutor
from langchain.tools import tool

@tool
def search_database(query: str) -> str:
    """搜索数据库"""
    return f"搜索结果：{query}"

agent = create_openai_tools_agent(llm, [search_database], prompt)
executor = AgentExecutor(agent=agent, tools=[search_database], verbose=True)
result = executor.invoke({"input": "查找用户信息"})
```

- 优势：生态丰富、社区活跃、工具链完善
- 劣势：抽象层多、调试复杂、版本迭代快导致 API 不稳定

### AutoGen

```python
import autogen

#​ 配置 Agent
assistant = autogen.AssistantAgent(
    name="assistant",
    llm_config={"model": "gpt-4"},
)

user_proxy = autogen.UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    code_execution_config={"work_dir": "coding"},
)

#​ 多 Agent 对话
user_proxy.initiate_chat(
    assistant,
    message="帮我写一个排序算法的性能对比测试",
)
```

- 优势：原生多 Agent 对话、代码执行、人类参与
- 劣势：配置复杂、主要面向代码生成场景

### CrewAI

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="研究员",
    goal="收集和分析信息",
    backstory="你是一位经验丰富的市场研究员",
    tools=[search_tool],
)

writer = Agent(
    role="写作者",
    goal="撰写高质量报告",
    backstory="你是一位专业的技术写作专家",
)

research_task = Task(description="研究 AI 市场趋势", agent=researcher)
write_task = Task(description="撰写市场分析报告", agent=writer)

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task])
result = crew.kickoff()
```

- 优势：角色定义直观、任务编排清晰、易于理解
- 劣势：定制性有限、社区生态较小

### 框架选型建议

| 框架 | 适用场景 | 学习曲线 | 定制度 |
|------|---------|---------|--------|
| LangChain Agent | 通用场景，需丰富工具 | 高 | 高 |
| AutoGen | 多 Agent 对话、代码生成 | 中 | 中 |
| CrewAI | 角色扮演、团队协作 | 低 | 中 |

## Agent 安全与可控性

### 护栏（Guardrails）

在 Agent 执行链路中设置安全检查点：

```python
class Guardrail:
    """Agent 安全护栏"""

    def __init__(self):
        self.rules = []

    def add_rule(self, name, check_fn, action="block"):
        """添加安全规则"""
        self.rules.append({
            "name": name,
            "check": check_fn,
            "action": action,  # "block" | "warn" | "log"
        })

    def validate(self, action, context):
        """验证 Agent 的行动是否安全"""
        violations = []
        for rule in self.rules:
            if not rule["check"](action, context):
                violations.append(rule)

        return {
            "is_safe": len(violations) == 0,
            "violations": violations,
        }


#​ 使用示例
guardrail = Guardrail()

#​ 规则 1：禁止删除操作
guardrail.add_rule(
    "no_delete",
    lambda action, ctx: "delete" not in action.get("tool", "").lower(),
    action="block",
)

#​ 规则 2：敏感数据检测
guardrail.add_rule(
    "no_sensitive_data",
    lambda action, ctx: not contains_sensitive_data(str(action)),
    action="block",
)

#​ 在 Agent 执行前检查
def safe_execute(agent, task):
    result = agent.plan(task)
    for action in result.actions:
        validation = guardrail.validate(action, context={"task": task})
        if not validation["is_safe"]:
            for v in validation["violations"]:
                print(f"安全拦截: {v['name']}")
            continue
        action.execute()
```

### 权限控制

```python
class PermissionManager:
    """Agent 权限管理"""

    LEVELS = {
        "read_only": ["search", "read", "list"],
        "standard": ["search", "read", "write", "summarize"],
        "admin": ["search", "read", "write", "delete", "execute", "admin"],
    }

    def __init__(self, agent_id, level="standard"):
        self.agent_id = agent_id
        self.level = level
        self.allowed_tools = self.LEVELS[level]

    def check_permission(self, tool_name):
        """检查 Agent 是否有权限使用指定工具"""
        return tool_name in self.allowed_tools

    def request_elevation(self, reason):
        """请求权限提升"""
        # 需要人工审批
        pass
```

### 审计日志

```python
import json
import time


class AuditLogger:
    """Agent 行为审计日志"""

    def __init__(self, log_file="agent_audit.jsonl"):
        self.log_file = log_file

    def log_action(self, agent_id, action, result, metadata=None):
        """记录 Agent 的每次行动"""
        entry = {
            "timestamp": time.time(),
            "agent_id": agent_id,
            "action": action,
            "result": str(result)[:500],  # 截断过长的结果
            "metadata": metadata,
        }
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(entry, ensure_ascii=False) + "\n")

    def get_agent_history(self, agent_id, limit=100):
        """获取指定 Agent 的行为历史"""
        history = []
        with open(self.log_file, "r", encoding="utf-8") as f:
            for line in f:
                entry = json.loads(line)
                if entry["agent_id"] == agent_id:
                    history.append(entry)
        return history[-limit:]
```

## 实战案例：构建代码审查 Agent

### 需求分析

构建一个能够自动审查代码并给出改进建议的 Agent，需要：

1. 读取代码变更（diff）
2. 分析代码质量（风格、安全、性能）
3. 检索相关的最佳实践
4. 生成结构化的审查报告

### 完整实现

```python
"""
代码审查 Agent - 完整实现
"""
import json
from dataclasses import dataclass


@dataclass
class CodeChange:
    """代码变更"""
    file_path: str
    diff: str
    language: str


@dataclass
class ReviewComment:
    """审查意见"""
    severity: str       # "critical" | "warning" | "info"
    file_path: str
    line_number: int | None
    category: str       # "security" | "performance" | "style" | "bug" | "maintainability"
    message: str
    suggestion: str


class CodeReviewAgent:
    """代码审查 Agent"""

    def __init__(self, llm_client, knowledge_base=None):
        self.llm = llm_client
        self.knowledge_base = knowledge_base

    def review(self, changes: list[CodeChange]) -> dict:
        """审查代码变更"""
        all_comments = []

        for change in changes:
            # 1. 基础代码分析
            comments = self._analyze_code(change)
            all_comments.extend(comments)

            # 2. 安全检查
            security_comments = self._security_check(change)
            all_comments.extend(security_comments)

            # 3. 查找相关最佳实践
            if self.knowledge_base:
                best_practices = self._lookup_best_practices(change)
                for practice in best_practices:
                    all_comments.append(ReviewComment(
                        severity="info",
                        file_path=change.file_path,
                        line_number=None,
                        category="best_practice",
                        message=practice,
                        suggestion="",
                    ))

        # 4. 生成审查摘要
        summary = self._generate_summary(all_comments, changes)

        return {
            "comments": all_comments,
            "summary": summary,
            "stats": {
                "total_comments": len(all_comments),
                "critical": sum(1 for c in all_comments if c.severity == "critical"),
                "warnings": sum(1 for c in all_comments if c.severity == "warning"),
                "info": sum(1 for c in all_comments if c.severity == "info"),
            },
        }

    def _analyze_code(self, change: CodeChange) -> list[ReviewComment]:
        """使用 LLM 分析代码质量"""
        prompt = f"""审查以下 {change.language} 代码变更，从以下维度分析：

1. 代码风格和可读性
2. 潜在的 Bug
3. 性能问题
4. 可维护性

代码变更：
```diff
{change.diff}
```

以 JSON 数组格式输出，每个问题格式：
{% raw %}
{{
  "severity": "critical|warning|info",
  "line_number": null,
  "category": "style|bug|performance|maintainability",
  "message": "问题描述",
  "suggestion": "修复建议"
}}
{% endraw %}
"""

        response = self.llm.invoke(prompt)
        try:
            issues = json.loads(response)
            return [
                ReviewComment(
                    severity=issue["severity"],
                    file_path=change.file_path,
                    line_number=issue.get("line_number"),
                    category=issue["category"],
                    message=issue["message"],
                    suggestion=issue["suggestion"],
                )
                for issue in issues
            ]
        except json.JSONDecodeError:
            return []

    def _security_check(self, change: CodeChange) -> list[ReviewComment]:
        """安全检查"""
        security_prompt = f"""检查以下代码变更中的安全问题：

- SQL 注入
- XSS 跨站脚本
- 命令注入
- 硬编码密钥/密码
- 不安全的随机数
- 路径遍历

代码变更：
```diff
{change.diff}
```

以 JSON 数组格式输出安全问题。如果没有安全问题，输出空数组 []。"""

        response = self.llm.invoke(security_prompt)
        try:
            issues = json.loads(response)
            return [
                ReviewComment(
                    severity=issue.get("severity", "critical"),
                    file_path=change.file_path,
                    line_number=issue.get("line_number"),
                    category="security",
                    message=issue["message"],
                    suggestion=issue["suggestion"],
                )
                for issue in issues
            ]
        except json.JSONDecodeError:
            return []

    def _lookup_best_practices(self, change: CodeChange):
        """从知识库查找最佳实践"""
        if not self.knowledge_base:
            return []

        results = self.knowledge_base.search(
            f"{change.language} best practices",
            top_k=3,
        )
        return [r.content for r in results]

    def _generate_summary(self, comments, changes):
        """生成审查摘要"""
        if not comments:
            return "代码审查通过，未发现问题。"

        critical = sum(1 for c in comments if c.severity == "critical")
        warnings = sum(1 for c in comments if c.severity == "warning")

        if critical > 0:
            return f"发现 {critical} 个严重问题，建议修复后再合并。"
        elif warnings > 0:
            return f"发现 {warnings} 个警告，建议关注。"
        else:
            return "仅有少量建议性意见，可以合并。"


#​ 使用示例
if __name__ == "__main__":
    agent = CodeReviewAgent(llm_client=None, knowledge_base=None)

    changes = [CodeChange(
        file_path="src/auth/login.py",
        diff="""- password = request.form['password']
+ password = request.form.get('password', '')
+ query = f"SELECT * FROM users WHERE password='{password}'" """,
        language="Python",
    )]

    result = agent.review(changes)
    print(json.dumps(result["summary"], ensure_ascii=False))
```

## 总结

AI Agent 是 LLM 从"被动工具"进化为"主动助手"的关键技术。从基础的 Function Calling 到高级的 Multi-Agent 协作，Agent 的能力边界在不断扩展。实践中需要关注几个核心维度：选择合适的推理模式（ReAct vs Plan-and-Execute）、设计好用的工具集、建立健壮的记忆系统、确保安全可控。框架选择上，建议从轻量实现开始，根据复杂度需求逐步引入 LangChain、AutoGen 或 CrewAI 等框架。
