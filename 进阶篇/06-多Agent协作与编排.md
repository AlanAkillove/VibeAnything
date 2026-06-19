# 06-多 Agent 协作与编排

> 读完本文你将了解：什么时候应该用多个 Agent 而不是一个、Subagents 和 Agent Teams 的区别、四种多 Agent 编排模式、A2A（智能体间协议）的工作原理、以及如何设计你的第一个多 Agent 系统

---

## 你可能遇到过的问题

你按照前面几篇教程做了一个 Agent，它现在能调用工具、能记住上下文、还能处理多模态输入。但随着你要做的事情越来越复杂，你遇到了新的瓶颈：

- **一个 Agent 处理所有事情，System Prompt 越来越臃肿**——既要它会写前端、又要懂后端、还要会审查代码安全。你往 System Prompt 塞了三个角色的指令，结果它在简单任务上的表现反而变差了——被无关指令污染了注意力
- **长任务的上下文爆炸**——你让 Agent 分析一个 5000 行的代码库，它探索了 20 个文件、每个文件读了一大段。到第 15 轮循环时，对话历史已经塞了 4 万 token，Agent 开始忘记前面发现的东西了
- **多个独立的子任务没法并行**——你需要在三个不同的目录里同时做三件事（重构 API、写测试、更新文档），但一个 Agent 一次只能做一件事
- **Agent 的决策质量不稳定**——有时候它过于自信地跳过了关键验证步骤，你希望有第二个"观点"来检查和质疑

这些问题的根源：**单体 Agent 的上下文容量、注意力带宽、并发能力都有硬上限。解决方案不是做大一个 Agent，而是把任务分给多个 Agent 协作完成。**

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| 多 Agent 肯定比单 Agent 好 | Anthropic 2026 年明确建议：**从最简单的方案开始**，只有在单 Agent 遇到可验证的瓶颈时才加多 Agent。每加一层编排就加一层延迟和失败概率 |
| 多 Agent = LangChain / CrewAI | 用框架不是必须的。用基础 Agent Loop + 简单的任务分发函数，30 行代码就能实现 Manager-Worker 模式 |
| Subagents 可以无限嵌套 | Claude Code 的 Subagents **深度限制为 1**——Subagent 不能再创建 Subagent。这是有意设计的安全边界 |
| 多 Agent 系统会自动协调好 | 无协调的多 Agent = N 个 Agent 各说各话。需要显式的任务分发、结果汇总、冲突解决机制 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| System Prompt 太臃肿 | 给不同角色创建独立的 Subagent，各自有精准的 System Prompt |
| 长任务上下文爆炸 | 用 Subagents 隔离高 volume 的子任务，主 Agent 只收摘要 |
| 独立子任务无法并行 | 用 Agent Teams 并行派发，共享任务列表，完成后自动汇总 |
| Agent 决策不可靠 | 加一个 Verifier Agent 专门做代码审查/安全检查，双 Agent 制 |
| 不确定该用 Subagents 还是 Teams | 独立、结果导向的任务 → Subagents（省钱）；需要讨论、互相质疑 → Teams |

---

## 核心概念一：Subagents — 隔离上下文的任务工人

### Subagents 是什么

Subagent 是一个**隔离的 AI 实例**——拥有自己独立的上下文窗口、自定义 System Prompt、限定的工具集。它接受一个任务，独立完成，**只把结果摘要返回给主 Agent**。

```
主 Agent（你的对话）
  ├── "帮我设计一个新的 API 模块"
  │   └── [Subagent: API Designer]
  │        ├── 独立上下文窗口
  │        ├── System Prompt: "你是后端 API 设计专家"
  │        ├── 工具: 文件读写, Shell, Grep
  │        └── 返回: API 设计方案摘要 + 代码草案路径
  │
  ├── "同时审查刚才写的代码安全性"
  │   └── [Subagent: Security Reviewer]
  │        ├── 独立上下文窗口
  │        ├── System Prompt: "你是安全审查专家，检查 OWASP Top 10"
  │        └── 返回: 发现 3 个问题，2 个 Critical
  │
  └── 主 Agent 汇总所有 Subagent 结果，输出最终回复
```

### 为什么 Subagents 有效

**1. 上下文隔离**
探索 20 个文件、读了几万 token 的代码？Subagent 在自己的上下文里完成这个工作，只返回 500 字的摘要。主 Agent 的上下文保持干净。

**2. 角色专注**
API 设计 Agent 只需要知道怎么设计好的 API，不需要知道 SQL 注入的检测方法。精确的 System Prompt → 更高质量的输出。

**3. 并行执行**
3 个 Subagent 可以同时运行——重构 API、写测试、更新文档同时进行，而不是串行等待。

### Subagents 的限制（重要）

- **不能嵌套**：Subagent 不能再创建 Subagent（深度 = 1）。这是一个安全边界
- **单向通信**：Subagent 只向主 Agent 汇报，Subagent 之间不直接通信
- **无状态**：Subagent 完成后销毁，不保留记忆

### 用代码实现最小 Subagent 模式

```python
from openai import OpenAI
import asyncio

client = OpenAI(
    api_key="sk-你的API Key",
    base_url="https://api.siliconflow.cn/v1"
)

def subagent(task: str, role: str, tools: list = None) -> str:
    """创建一个隔离的 Subagent 执行单一任务。"""
    system_prompt = f"你是{role}。只做分配给你的任务，完成后输出结果。不要做多余的事。"
    
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": task}
    ]
    
    response = client.chat.completions.create(
        model="deepseek-ai/DeepSeek-V4-Flash",  # 子任务用便宜模型
        messages=messages,
        tools=tools,
        max_tokens=2000
    )
    
    return response.choices[0].message.content

async def run_subagent(task: str, role: str) -> tuple:
    """在线程池中运行同步的 Subagent，实现真正的并行。"""
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, subagent, task, role)
    return role, result

# 主 Agent 并行调用 Subagents
async def main_agent(user_request: str):
    # 第一步：主 Agent 分析请求，拆分子任务
    plan = client.chat.completions.create(
        model="Qwen/Qwen3.7-72B-Instruct",
        messages=[{
            "role": "system",
            "content": "分析用户请求，拆分为可并行的子任务。输出 JSON: [{id, role, task}]"
        }, {
            "role": "user",
            "content": user_request
        }]
    )
    
    import json
    subtasks = json.loads(plan.choices[0].message.content)
    
    # 第二步：真正并行执行所有子任务
    tasks = [
        run_subagent(task=s["task"], role=s["role"])
        for s in subtasks
    ]
    results = dict(await asyncio.gather(*tasks))
    
    # 第三步：主 Agent 汇总结果
    synthesis = client.chat.completions.create(
        model="Qwen/Qwen3.7-72B-Instruct",
        messages=[{
            "role": "user",
            "content": f"汇总以下子任务结果并给出最终回答：\n{json.dumps(results, ensure_ascii=False)}"
        }]
    )
    
    return synthesis.choices[0].message.content

# 使用
result = asyncio.run(main_agent(
    "分析这个项目：1) 检查代码安全  2) 评估 API 设计  3) 审查测试覆盖率"
))
```

---

## 核心概念二：Agent Teams — 互相讨论的协作体

### Subagents vs Agent Teams

| 维度 | Subagents | Agent Teams |
|------|-----------|-------------|
| **通信** | 只向主 Agent 汇报 | 队友之间可以直接发消息 |
| **协调** | 主 Agent 管理一切 | 共享任务列表，自动认领 |
| **讨论** | 不能互相质疑 | 可以直接挑战对方的方案 |
| **成本** | 低（结果摘要） | 高（每个 Team 成员是独立 Agent 实例） |
| **适合** | 独立、结果导向的任务 | 需要讨论、辩论、权衡的任务 |

### 什么时候用 Teams

- **竞争型调试**：两个 Agent 从不同假设出发排查同一个 Bug，互相质疑对方的推理
- **多维度代码审查**：安全 Agent + 性能 Agent + 可维护性 Agent 同时审查
- **跨层功能**：前端 Agent + 后端 Agent + 测试 Agent 各管一层，但要协商接口

### 什么时候不要用 Teams

如果你的任务可以用 Subagents 完成，就用 Subagents。Teams 的通信成本高，只适合真正需要"讨论"的场景。Anthropic 的建议：**"Start with a single agent. Add multi-agent only when you can demonstrate that a single agent hits a verifiable limit."**

---

## 核心概念三：四大编排模式

### 模式 1：Manager-Worker（主从模式）

最常用。一个 Manager Agent 拆解任务、分配给 Worker、汇总结果。

```
Manager ──→ Worker A（安全审查）
         ├─→ Worker B（性能分析）
         └─→ Worker C（文档检查）
              ↓
         Manager 汇总 → 最终报告
```

**适合**：代码审查、研究报告、多维度分析

### 模式 2：Assembly-Line（流水线模式）

各 Agent 串行处理，前一环节的输出是下一环节的输入。

```
需求分析 → 架构设计 → 代码实现 → 测试验证 → 文档生成
  Agent A    Agent B    Agent C    Agent D    Agent E
```

**适合**：内容生产、文档处理、CI/CD 质量门

### 模式 3：Debate Pattern（辩论模式）

两个 Agent 持不同立场辩论，第三个 Agent 裁决。

```
Agent A（主张方）→ 方案 A 的优点、为什么选它
Agent B（质疑方）→ 方案 A 的风险、为什么方案 B 更好
Agent C（裁决方）→ 综合考虑，最终决定：方案 A，但需要 B 建议的额外防护
```

**适合**：架构决策、安全关键系统设计、需要多角度权衡的场景

### 模式 4：Swarm（集群模式）

对等 Agent 自组织，无中央调度者。类似蚂蚁找食物——每个 Agent 独立探索，自然涌现全局最优。

```
Agent 1 探索方向 α  ──┐
Agent 2 探索方向 β  ──┼──→ Reducer 聚合 → 输出
Agent 3 探索方向 γ  ──┘
（各有独立上下文，互相不知道对方存在）
```

**适合**：探索性研究、假设验证、并行信息检索

### 模式选择速查

| 场景 | 推荐模式 |
|------|---------|
| 需要一个人掌控全局、任务边界清晰 | Manager-Worker |
| 流程步骤固定、有明确的先后顺序 | Assembly-Line |
| 影响重大的决策，需要多角度审视 | Debate |
| 不确定答案在哪，需要广泛探索 | Swarm |
| 混合需求 | 嵌套组合（如在 Manager-Worker 中用 Debate 模式做关键决策） |

---

## 核心概念四：A2A — Agent 之间的通信标准

### 什么是 A2A

**A2A = Agent-to-Agent Protocol**，Google 提出并捐赠给 Linux 基金会。解决的是 Agent 之间如何**发现对方、建立通信、委派任务**的问题。

如果 MCP 是 "Agent ↔ 工具的 USB-C"，那 A2A 就是 "Agent ↔ Agent 的 USB-C"。

### A2A 的三步流程

```
1. Discovery（发现）
   Agent A ──→ GET /.well-known/agent-card.json
            ←── {"name": "CodeReviewer", "capabilities": [...], "endpoint": "..."}

2. Authorization（授权）
   Agent A ──→ 验证身份
            ←── 授权 Token

3. Communication（通信）
   Agent A ──→ 发送任务（JSON-RPC over HTTPS）
            ←── 流式返回结果（SSE）
```

### MCP vs A2A — 同层不同向

| 维度 | MCP | A2A |
|------|-----|-----|
| **关系** | Agent → 工具/数据 | Agent → Agent |
| **模式** | 客户端-服务器（主从） | 点对点（对等） |
| **发现** | 工具列表（tools/list） | Agent Card（.well-known） |
| **通信** | JSON-RPC 2.0 | JSON-RPC 2.0 + SSE 流式 |
| **类比** | "递给你一把锤子" | "请另一个专家帮你做事" |

> **简单判断原则**：如果是确定性资源或操作 → MCP。如果需要对话、协商、委派判断 → A2A。

---

## 设计你的第一个多 Agent 系统

### 决策清单

在动手之前，回答以下问题：

1. **单 Agent 真的不够了吗？** 有没有尝试过优化 System Prompt 或加工具？90% 的场景单 Agent + 好设计就够
2. **任务可以拆分成独立子任务吗？** 如果子任务之间有强依赖，多 Agent 可能不如单 Agent
3. **需要讨论还是只需要执行？** 执行 → Subagents。讨论 → Teams
4. **成本预算？** 每个额外的 Agent 都增加 Token 消耗

### 推荐启动方案

```
第 1 周：单 Agent + Skills（模块化能力但不引入多 Agent 复杂度）
第 2 周：Subagents（拆出独立子任务）
第 3 周：Manager-Worker 模式（正式的编排）
第 4 周：按需加入 Debate/Swarm（只在验证了简单方案不够时）
```

**不要在第一天就设计多 Agent 架构。** 让需求推动架构，而不是让架构推动需求。

---

## 要点总结

1. **多 Agent 不是默认选项**——从单 Agent 开始，只在遇到可验证的上下文/注意力/并发瓶颈时才加
2. **Subagents = 隔离的任务工人**：独立上下文 → 只返回摘要 → 保护主 Agent 上下文。不能嵌套、不能互相通信
3. **Agent Teams = 互相讨论的协作体**：直接通信、共享任务列表、适合需要多角度审视的场景。成本更高
4. **四大编排模式**：Manager-Worker（主从）、Assembly-Line（流水线）、Debate（辩论）、Swarm（集群）。选模式比选框架更重要
5. **A2A 让跨框架、跨组织的 Agent 协作成为可能**：Agent Card 发现 → 授权 → JSON-RPC 通信。与 MCP（Agent ↔ 工具）互补
6. **MCP + A2A + Skills = 2026 年 Agent 基础设施三件套**

---

## 更进一步

- **Agent 可靠性**：参考 [07-Agent可靠性与可观测性](./07-Agent可靠性与可观测性.md) — 多 Agent 系统的可靠性如何保障
- **Skills 与 MCP**：参考 [05-Agent Skills与工具协议](./05-Agent%20Skills与工具协议.md) — 工具连接和能力模块化的基础
- **Anthropic 官方指南**：[Building multi-agent systems: when and how to use them](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them) — 为什么"简单优先"是对的
- **Google A2A 文档**：[A2A Protocol](https://opensource.googleblog.com/2026/04/meet-the-a2family.html) — Agent 间通信的完整规范
