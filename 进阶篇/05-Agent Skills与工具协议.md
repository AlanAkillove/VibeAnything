# 05-Agent Skills 与工具协议

> 读完本文你将了解：Agent Skills 是什么、MCP（模型上下文协议）如何标准化工具连接、Skills vs MCP 的分界线在哪里、以及如何写一个简单的 MCP Server 和一个 Skill

---

## 你可能遇到过的问题

你按照上一篇 Agent 架构入门做了个 Agent，但随着功能增多，问题来了：

- **System Prompt 越来越长**——查天气、发邮件、搜索、计算器……所有工具的说明和规则全塞在一起，token 消耗越来越大，Agent 的行为反而越来越不稳定
- **换个项目就要重写一遍**——上次做的搜索工具写得挺好，但代码和业务逻辑混在一起，没法直接拿到新项目用
- **团队成员各写各的工具**——你定义函数的 JSON schema 是 this，同事用的是 that，换模型的时候协议不兼容
- **想接第三方工具但没有标准接口**——比如想接公司的数据库、Slack、飞书，每个平台都有不同的 API 格式，集成成本极高

这些问题指向同一个方向：**Agent 的能力需要一个标准化的模块体系——让工具可复用、可组合、可跨平台互通。**

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| Skills 就是一个写了 System Prompt 的 Markdown 文件 | Skills 是**三层渐进式披露**的能力包：元数据（启动时加载）→ 指令（匹配任务时加载）→ 脚本/模板（执行时访问），远比一个 prompt 文件精密 |
| MCP 就是 Function Calling 的升级版 | MCP 解决的是**标准化连接**问题（Agent ↔ 工具的接口统一），Function Calling 解决的是**调用机制**问题（LLM 如何输出结构化的工具调用）。两者是不同层次，互为补充 |
| Skills 和 MCP 是竞争关系 | Skills 教 Agent **怎么做**（操作手册），MCP 给 Agent **工具**（锤子）。MCP 封装 Skill = "工具的使用说明书" |
| 装了 MCP 就不用管安全了 | MCP 的连接能力比 Function Calling 更强大——Agent 可以操作文件系统、发消息、查数据库。安全不是自动的，需要显式配置权限和守卫 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| System Prompt 太长，token 浪费严重 | 把每个工具的能力说明抽离成独立 Skill。Agent 只在匹配到相关任务时才加载完整指令 |
| 工具代码和业务逻辑耦合 | 把工具封装成 MCP Server（独立进程），Agent 通过标准协议连接——改工具不需要动 Agent 代码 |
| 想集成微信/飞书/Slack 但每个 API 不同 | 找一个现成的 MCP Server，或者自己写一个。MCP Server 只写一次，任何兼容的 Agent 客户端都能用 |
| 不确定该用 Skill 还是 MCP | 操作手册式的知识 → Skill；需要持续连接外部系统 → MCP；两者配合用最常见 |
| 多个 MCP Server 同时连接，Agent 选错工具 | 给每个工具写清晰的 description，控制在 100 字以内。工具太多时考虑按场景拆 Agent |

---

## 核心概念一：Agent Skills — 模块化能力包

### Skills 解决了什么问题？

传统 Agent 的能力定义方式是把所有知识塞进 System Prompt：

```
System Prompt (10000+ tokens):
  - 你是代码审查专家，需要检查: 1) 安全性 ... 2) 性能 ...
  - 你是文档作者，遵循以下格式: 1) 标题规范 ... 2) 代码展示 ...
  - 错误处理策略: 1) 网络超时重试 ... 2) ...
  - 模型路由规则: 简单任务走 Flash，复杂走 Pro ...
  （越写越长...）
```

Skills 把这种单体知识拆成独立的、可组合的文件：

```
.claude/skills/
├── security-review/SKILL.md    ← 代码审查技能
├── doc-writer/SKILL.md         ← 文档撰写技能
├── error-handler/SKILL.md      ← 错误处理技能
└── model-router/SKILL.md       ← 模型路由技能
```

Agent 启动时只加载所有 Skill 的**元数据**（名称 + 一句话描述，~30 字/skill），匹配到相关任务时才加载完整指令。

### 三层渐进式披露

这是 Skills 架构的核心设计理念：

```
Level 1 — 元数据（启动时加载，<1KB）  → "我有这些技能"
  name: security-review
  description: 代码安全审查，检查 SQL 注入、XSS、敏感信息泄露

Level 2 — 指令（任务匹配时加载，~3KB）→ "我知道怎么做"
  完整的审查步骤、检查项、优先级标注、输出格式规范

Level 3 — 资源（执行阶段访问，任意大小）→ "这是工具和模板"
  安全扫描脚本、漏洞数据库、修复方案模板
```

**为什么这个设计好？**
- 10 个 Skill 每个 2000 字，全加载就是 20000 token。用渐进式披露，启动时只消耗 ~200 token 的元数据，实际执行的 Skill 才加载完整指令
- Agent 的上下文质量大幅提升——不被无关指令污染，Model 更专注当前任务

### Skills 标准目录结构

```
my-skill/
├── SKILL.md           ← 必选：YAML frontmatter + Markdown 指令
├── scripts/           ← 可选：Agent 可执行的脚本
│   └── scan.py
├── templates/         ← 可选：输出模板
│   └── report.md
└── docs/              ← 可选：参考文档（Level 3 才加载）
    └── owasp-top10.md
```

**SKILL.md 示例**：

```markdown
---
name: security-review
description: 代码安全审查 — 检查注入漏洞、认证缺陷、敏感信息泄露
user-invocable: true
disable-model-invocation: false
---

## 审查流程

1. 阅读已改动的文件（git diff）
2. 逐文件检查以下风险维度：
   - **注入漏洞**：SQL 拼接、命令注入、路径遍历
   - **认证缺陷**：弱密码策略、越权风险
   - **信息泄露**：日志中的敏感数据、错误信息中的内部路径

## 输出格式

- [Critical] 必须立即修复
- [Warning] 建议修复
- [Info] 可选改进
```

### Skills 的行业采纳

2025 年 10 月 Anthropic 在 Claude Code 中推出 Skills，12 月 18 日作为开放标准发布。48 小时内被 Microsoft 和 OpenAI 采纳。截至 2026 年 6 月：

- **Cursor**、**Codebuddy** 原生支持
- **OpenClaw** 社区拥有 3000+ 社区 Skills
- Skills 已被集成到 Qwen Code CLI、KodaX 等国产工具中
- Gartner 预测：到 2026 年，75% 的 AI 项目将聚焦可组合的 Skills 而非单体 Agent

---

## 核心概念二：MCP — 工具连接的 USB-C

### MCP 解决了什么问题？

在 MCP 之前，每个 AI 客户端连接每个工具需要单独开发：

```
AI 客户端 A ──→ 自定义适配器 ──→ 数据库
AI 客户端 B ──→ 自定义适配器 ──→ 数据库
AI 客户端 C ──→ 自定义适配器 ──→ 数据库
（N 个客户端 × M 个工具 = N×M 个适配器要写）
```

有了 MCP 之后：

```
任意 AI 客户端 ──→ [MCP 协议] ──→ MCP Server ──→ 数据库
                                   └─→ MCP Server ──→ Slack
                                   └─→ MCP Server ──→ 文件系统
（N 个客户端 + M 个工具 = N+M 个连接）
```

**MCP = Model Context Protocol（模型上下文协议）**。Anthropic 于 2024 年 11 月开源，2025 年 12 月捐赠给 Linux 基金会 Agentic AI Foundation。2026 年已成为 AI 工具集成的事实标准：

- **9700 万+** 月 SDK 下载量
- **10000+** 活跃 MCP Server
- 被 OpenAI、Google、Microsoft 全面采纳

### MCP 的三大原语

| 原语 | 方向 | 作用 | 示例 |
|------|------|------|------|
| **Tools** | Agent → 工具 | 让 Agent 执行操作 | 查询数据库、发送消息、创建文件 |
| **Resources** | 工具 → Agent | 暴露只读数据 | 读取文档内容、获取 API 响应 |
| **Prompts** | 工具 → Agent | 提供可复用模板 | "帮我写周报" 的标准 Prompt 模板 |

### 写一个最简单的 MCP Server

以下示例创建一个数学计算 MCP Server，任何兼容 MCP 的 Agent 客户端都能调用：

```python
# math_server.py — 一个最简单的 MCP Server
# pip install mcp

from mcp.server import Server
from mcp.server.stdio import stdio_server

server = Server("math-server")

@server.tool()
def add(a: float, b: float) -> float:
    """加法计算器：计算两个数的和"""
    return a + b

@server.tool()
def multiply(a: float, b: float) -> float:
    """乘法计算器：计算两个数的积"""
    return a * b

async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

Agent 通过 MCP 连接后，`add` 和 `multiply` 就像 Agent 自身的能力一样可用。你不需要在 Agent 代码里硬编码工具的 JSON schema——MCP Server 自动提供。

### MCP 的配置（以 Claude Code 为例）

```json
{
  "mcpServers": {
    "math": {
      "command": "python",
      "args": ["math_server.py"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-filesystem", "/path/to/allowed/dir"]
    }
  }
}
```

Agent 启动时自动连接所有配置的 MCP Server，协议自动发现 Server 提供的能力（tools/list、resources/list），Agent 就具备了操作文件系统和数学计算的能力——零代码集成。

---

## 核心概念三：Skills vs MCP — 精确分界线

这是 2026 年最容易被混淆的一对概念：

| 维度 | Skills | MCP |
|------|--------|-----|
| **解决的问题** | 教 Agent **怎么做事** | 给 Agent **做事的能力** |
| **内容** | 指令 + 脚本 + 模板（知识） | 工具 + 数据源（连接） |
| **加载方式** | 按需渐进式加载 | 持续连接，随叫随到 |
| **执行位置** | Agent 进程中执行脚本 | 独立进程（MCP Server） |
| **适用场景** | 代码审查流程、文档模板、错误处理策略 | 数据库查询、文件系统操作、外部 API 调用 |
| **类比** | 操作手册 / 使用说明书 | 锤子 / 螺丝刀 |

### 场景判断速查

| 你的需求 | 选什么 | 原因 |
|---------|--------|------|
| 需要一套标准化的代码审查流程 | **Skill** | 这是知识和流程，不需要外部连接 |
| 需要查询公司数据库 | **MCP** | 需要持续连接外部系统 |
| 需要批量处理图片 | **Skill**（含 Python 脚本） | 自包含的一次性操作 |
| 需要在 Slack 上发消息 | **MCP** | 需要与外部服务保持连接 |
| 需要教 Agent 怎么写周报 | **Skill** | 模板 + 指令即可 |
| 需要让 Agent 读写项目文件 | **MCP** | 文件系统是持续变化的数据源 |

### 最常见的组合：MCP 封装 Skill

```
Skill（使用说明书）    +    MCP Server（工具箱）
├── 什么时候用这个工具      ├── 数据库查询接口
├── 正确的参数怎么写        ├── 错误处理
├── 常见踩坑和注意事项      ├── 连接池管理
└── 输出格式规范            └── 权限控制
```

**MCP 是"递给你一把锤子"，Skill 是"教你怎么用这把锤子"。** 两者配合才是最有效的 Agent 能力扩展方式。

---

## 动手：为你的 Agent 添加 Skills 和 MCP

### 第一步：装一个现成的 MCP Server

用 npx 零安装运行文件系统 MCP Server：

```bash
npx -y @anthropic/mcp-filesystem /path/to/your/project
```

这会启动一个 MCP Server，Agent 连接后就能浏览、读取、写入你指定的目录。

### 第二步：在 Claude Code 或兼容工具中配置

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-filesystem", "."]
    }
  }
}
```

### 第三步：写第一个 Skill

在 `.claude/skills/` 下创建 `SKILL.md`：

```markdown
---
name: code-explainer
description: 用中文解释代码逻辑，适合阅读陌生代码库时使用
---

## 步骤

1. 读取目标文件或函数
2. 用以下结构输出解释：
   - **一句话总结**：这段代码做了什么
   - **核心逻辑**：按执行流程解释（不要逐行翻译）
   - **关键设计选择**：为什么这么写（如果有非显而易见的写法）
3. 用中文回答，术语保留英文
```

配置好后，当你对 Agent 说"帮我理解一下这个文件"，Agent 会自动匹配这个 Skill 并加载指令。

### 第四步：验证

```bash
# 在 Claude Code 中查看已加载的 Skills
/skills

# 查看已连接的 MCP Server
/mcp
```

---

## 要点总结

1. **Skills = Agent 的模块化知识库**：把单体 System Prompt 拆成文件级、可复用、按需加载的能力包。三层渐进式披露（元数据→指令→资源）在大幅节省 token 的同时保持能力完整
2. **MCP = Agent 到工具的标准化协议**：解决 N×M 集成复杂度为 N+M。三大原语：Tools（执行操作）、Resources（暴露数据）、Prompts（复用模板）。2026 年已是行业共识标准，9700 万+ 月下载量
3. **Skills 教怎么做，MCP 给工具**：两者互补而非竞争。手册式知识用 Skill，外部系统连接用 MCP，复杂场景两者配合
4. **写一个 MCP Server 只需要 20 行代码**：用 Python `mcp` 库，`@server.tool()` 装饰器定义工具，Agent 自动发现
5. **从今天就可以开始**：装一个现成的 MCP Server → 写一个 Skill → 感受模块化 Agent 开发的效率提升

---

## 更进一步

- **多 Agent 协作**：参考 [06-多Agent协作与编排](./06-多Agent协作与编排.md) — 当单个 Agent 不够用时，如何组织多个 Agent 协作
- **Agent 入门基础**：参考 [03-Agent架构入门](./03-Agent架构入门.md) — 如果你还没读过，先理解 Agent 的四大组件
- **MCP 官方文档**：[modelcontextprotocol.io](https://modelcontextprotocol.io) — 完整的协议规范和 SDK 文档
- **Skills 开放标准**：Claude Code Skills 文档 — 了解 Skills 的完整规范和行业采纳情况
