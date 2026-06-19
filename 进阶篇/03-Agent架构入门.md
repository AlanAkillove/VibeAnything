# 03-Agent架构入门

> 读完本文你将了解：什么是 AI Agent、Agent 的四大核心组件（LLM + 工具 + 记忆 + 规划）、Agent 的基本架构设计、以及用 OpenAI Function Calling 实现一个简单 Agent 的完整代码
> 注：本文假设你已经理解 Function Calling 的基础（参考本模块第一篇文章），我们把概念向前推进一步。

---

## 你可能遇到过的问题

你写了几个独立的 Function Calling 脚本，让 AI 能查天气、搜网页、算数。但渐渐你发现：

- **每次和 AI 的对话都是无状态的**——它记不住之前查过什么、算过什么，每个问题像重新认识一个新 AI
- **AI 只能一次调一个函数**——有时候一个问题需要「先搜索再根据结果做计算」，你发现单次 function call 搞不定这个流程
- **你的代码越来越乱**——每次要处理多个工具的调用、解析返回值、决定下一步做什么，逻辑散落在各处，改一个地方就会影响别的地方
- **你发现 AI 经常在不该调用工具的时候调用**——用户随便问个常识问题，它也要花几十个 token 去搜一下

这些问题指向同一个方向：**你需要一个 Agent 架构来统一管理 LLM、工具、状态和决策逻辑。**

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| Agent = 更强的 LLM | Agent 是「LLM + 工具 + 记忆 + 规划」的系统架构。给 LLM 加 Function Calling 只是个起点，真正的 Agent 需要管理状态、循环、决策 |
| Agent 就是自动化脚本 | 自动化脚本是固定流程，Agent 是动态决策。LLM 每隔一轮根据现状决定下一步做什么，可能完全不同 |
| Agent 框架必须用 LangChain / CrewAI | LangChain 封装了很多便利功能，但从零手写一个简单 Agent 反而更有助于理解原理。30 行代码就可以跑通核心逻辑 |
| Agent 能完全自主工作 | 目前最先进的 Agent 也需要设定边界、human-in-the-loop 检查和异常处理。完全自治的 Agent 仍是一个研究问题 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| AI 记不住前几次对话的工具调用结果 | 把每次 tool call 和 tool result 都追加到 messages，作为 Agent 的短期记忆 |
| 一个任务需要多个步骤（先搜再算） | 用循环结构：LLM 输出→执行→结果放回→LLM 再决策，直到 LLM 输出最终答案 |
| 管理多个工具调用越来越乱 | 用统一的 Tool 注册机制和 Router，把工具的定义、执行、错误处理收敛到一处 |
| AI 在不该调工具的时候乱调 | 在 system prompt 里明确约束「对于常识性问题直接回答，不要调工具」 |

---

## 核心概念：Agent 的四大组件

一个 AI Agent 可以理解为一个**能自主决策、使用工具、持续运行的 LLM 系统**。它由四个核心组件构成：

### 1. LLM（大脑）
Agent 的推理核心。负责理解用户意图、决定调用什么工具、解读工具返回结果、生成最终回答。

### 2. 工具（手）
Agent 可以调用的外部能力：搜索、计算器、数据库、API、文件系统等。对应 Function Calling 里注册的 functions。

### 3. 记忆（短期 + 长期）
Agent 保持对话上下文的能力。短期记忆就是 messages 列表；长期记忆可以是向量数据库中存储的历史对话摘要或知识。

### 4. 规划（策略）
Agent 决定「先做什么、再做什么」的能力。简单规划是 LLM 根据当前状态决定下一步；复杂规划可以让 Agent 先拆解出子任务、再逐个执行。

```
                    +---------+
                    |  用户    |
                    +----+----+
                         | 问题
    +--------------------v--------------------+
    |              Agent Loop                 |
    |                                        |
    |  +-------+     +-----------+           |
    |  |  LLM  |<--->|  记忆     |           |
    |  |(推理)  |     |(上下文)    |           |
    |  +---+---+     +-----------+           |
    |      |                                 |
    |      | 决定调用工具                     |
    |      v                                 |
    |  +-------+                             |
    |  | Router|---> Tool 1 (搜索)           |
    |  |(调度)  |---> Tool 2 (计算)           |
    |  +-------+---> Tool 3 (数据库)         |
    |      |                                 |
    |      v (结果返回 LLM)                   |
    |  +-------+                             |
    |  | 规划   | (是否需要更多步骤？)         |
    |  +---+---+                             |
    |      | 完成 → 输出最终回答              |
    +----------------------------------------+
                    |
                    v
                最终答案
```

---

## 用 OpenAI Function Calling 实现最小 Agent

我们把上面的架构落地成代码。核心思路是**一个 while 循环**，每一步 LLM 决定是输出最终回答还是调用工具。

```python
from openai import OpenAI
import json

client = OpenAI(
    api_key="sk-你的API Key",
    base_url="https://api.siliconflow.cn/v1"  # 国内直接用，无需特殊网络
)

# ========== 1. 定义工具 ==========

tools = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "搜索网络获取最新信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "执行数学计算",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "数学表达式"}
                },
                "required": ["expression"]
            }
        }
    }
]

# ========== 2. 工具执行器 ==========

tool_registry = {}

def register_tool(name: str, fn):
    tool_registry[name] = fn

def execute_tool(name: str, arguments: dict) -> str:
    if name in tool_registry:
        return tool_registry[name](**arguments)
    return f"错误：未知工具 {name}"

@register_tool
def web_search(query: str) -> str:
    # 这里应该接真实搜索 API
    return f"关于「{query}」的搜索结果：示例结果 1、示例结果 2"

@register_tool
def calculate(expression: str) -> str:
    import math
    allowed = {"abs": abs, "math": math, "float": float, "int": int}
    return str(eval(expression, {"__builtins__": {}}, allowed))

# ========== 3. Agent 主循环 ==========

SYSTEM_PROMPT = """你是一个 AI 助手，可以使用工具来完成任务。
规则：
- 如果问题需要实时信息或精确计算，请使用对应的工具
- 对于常识性问题，直接回答，不要调工具
- 如果工具返回的结果不够回答用户问题，可以多次调用工具
- 完成所有必要的工具调用后，基于已有的信息回应"""

messages = [{"role": "system", "content": SYSTEM_PROMPT}]

def run_agent(user_input: str, max_iterations: int = 10) -> str:
    messages.append({"role": "user", "content": user_input})
    
    for iteration in range(max_iterations):
        response = client.chat.completions.create(
            model="Qwen/Qwen3.7-72B-Instruct",  # 可换成任意 SiliconFlow 上的模型
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )
        
        message = response.choices[0].message
        messages.append(message)
        
        # 如果 LLM 没有调用工具，说明它已经给出了最终回答
        if not message.tool_calls:
            return message.content
        
        # 遍历所有工具调用（一次可能调多个）
        for tool_call in message.tool_calls:
            fn_name = tool_call.function.name
            fn_args = json.loads(tool_call.function.arguments)
            
            print(f"  [Agent] 调用工具: {fn_name}({fn_args})")
            
            # 执行工具
            try:
                result = execute_tool(fn_name, fn_args)
            except Exception as e:
                result = f"执行出错: {e}"
            
            print(f"  [Agent] 工具返回: {result[:100]}...")
            
            # 把工具结果放回消息列表
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result
            })
        
        # 继续循环，LLM 处理工具返回结果
        # 可能还要继续调工具，也可能直接回答
    
    return "已达到最大迭代次数，无法完成请求"

# ========== 4. 使用 Agent ==========

# 示例 1: 多步推理（先搜再算）
result = run_agent("2025 年全球 GDP 排名前三的国家，它们 GDP 的平均值是多少？")
print(f"\n最终回答:\n{result}")

# 示例 2: 直接回答（不调工具）
result2 = run_agent("什么是 Python 的列表推导式？")
print(f"\n最终回答:\n{result2}")
```

---

## 理解 Agent 主循环

上面的 `run_agent` 函数是 Agent 的核心模式。让我们拆解它的每一步：

```
第 1 轮: 用户输入 "搜索 X 然后算 Y"
  → LLM: 需要先调用 web_search("X")
  → 执行 web_search，结果放回 messages
  → 继续循环

第 2 轮: messages 包含搜索结果
  → LLM: 现在有了数据，需要调用 calculate(...) 计算
  → 执行 calculate，结果放回 messages
  → 继续循环

第 3 轮: messages 包含计算结果
  → LLM: 数据齐全，直接输出最终回答
  → 返回最终答案，循环结束
```

核心机制是 **LLM 每次看到更新后的 messages（含工具返回数据），重新决策下一步**。

---

## 给 Agent 加记忆（Memory）

上面的 Agent 在同一个 run 内部记住上下文，但跨对话没有记忆。可以加一个简单的长期记忆层：

```python
class Memory:
    def __init__(self):
        self.short_term = []  # 当前对话的 messages
        self.long_term = []   # 之前对话的摘要
    
    def add_session(self, summary: str):
        self.long_term.append(summary)
    
    def build_context(self) -> str:
        if not self.long_term:
            return ""
        return "历史对话摘要：\n" + "\n".join(self.long_term[-5:])
```

在每个新 session 开始时，把 `memory.build_context()` 塞进 system prompt，Agent 就能「记得」之前聊过什么。

---

## 进阶方向

实现一个基础 Agent 只是开始。生产级的 Agent 需要考虑：

- **错误处理与重试**：工具调用失败时 Agent 应该自动重试或用替代方案
- **Human-in-the-loop**：关键操作（如发邮件、修改数据库）需要先询问用户确认
- **任务拆解**：复杂问题先由 LLM 分解成子任务，sub-agent 各做各的，最后汇总
- **并行工具调用**：同一轮可以同时调用多个无依赖的工具（OpenAI 原生支持）
- **持续运行与观察**：Agent 可以在后台运行，定期检查条件是否满足并触发动作

---

## 要点总结

- Agent = LLM（大脑）+ 工具（手）+ 记忆（上下文）+ 规划（策略）
- 核心架构是一个 while 循环：LLM 输出 → 执行工具 → 结果放回 messages → LLM 再决策，直到输出最终回答
- 用 OpenAI Function Calling + 一个简单的循环，不到 50 行代码就能跑通 Agent 核心逻辑
- system prompt 中的规则约束直接影响 Agent 的决策质量（什么时候调工具、什么时候直接回答）
- 错误处理、Human-in-the-loop、任务拆解是 Agent 进阶的关键方向
