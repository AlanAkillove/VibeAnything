# 04-Prompt 模板库

> 可直接复用的 Prompt 模板，覆盖日常使用频率最高的 6 个场景。每个模板配有「差 vs 好」对比，帮你理解每个模板背后的设计思路。

---

## 1. 代码生成

### 差的 Prompt
`
写一个 Python 函数读取 CSV 文件
`

### 好的 Prompt
`
Task: 写一个 Python 函数读取 CSV 文件并返回格式化结果
Language: Python 3.12+
Requirements:
- 使用 pandas 读取输入 CSV 文件路径
- 自动检测编码（优先 UTF-8，失败后尝试 GBK）
- 返回 dict list，key 为列名，value 为对应值
- 对数值列自动转换类型（不要全部是 str）
- 异常处理：文件不存在、编码错误、空文件
- 日志输出：打印文件行数和列名
Output: 仅输出代码，不要额外解释
`

### 为什么好
差的 Prompt 太模糊，模型不知道要什么编码、什么异常处理、什么输出格式。一个好的代码生成 prompt = 任务描述 + 语言版本 + 具体约束 + 输出格式。AI 生成的程序在第一次就能正确使用的概率大幅提升。

---

## 2. Code Review

### 差的 Prompt
`
帮我 review 这段代码
`

### 好的 Prompt
`
Code Review Request

Context: 这是一个用户注册接口的后端 handler，运行在 FastAPI 上。
文件: /app/routes/auth.py
重点关注:
1. 安全性——是否有 SQL 注入、输入验证漏洞
2. 性能——是否有不必要的数据库查询或 N+1 问题
3. 错误处理——边界情况和异常是否覆盖完整
4. 可维护性——命名、结构、函数长度是否合理

Review Format:
- [Critical]: 必须修复的问题
- [Warning]: 建议修复的问题
- [Nitpick]: 风格性问题，不强制
- [Praise]: 做得好的地方

Code:
`python
@app.post("/register")
async def register(username: str, password: str):
    user = db.query(f"SELECT * FROM users WHERE username='{username}'")
    if user:
        return {"error": "user exists"}
    db.execute(f"INSERT INTO users VALUES ('{username}', '{password}')")
    return {"ok": True}
`
`

### 为什么好
差的 Prompt 没有给 model 任何 context，review 结果往往流于表面。好的 review prompt 包含了上下文、关注点、格式要求，输出的 review 更有针对性、更有结构、更可操作。

---

## 3. Debugging

### 差的 Prompt
`
我的代码报错了，帮我看看什么问题
`

### 好的 Prompt
`
Debug Request

环境:
- Python 3.11
- Windows 11
- openai 1.30.0

错误信息:
Traceback (most recent call last):
  File "test.py", line 12, in <module>
    response = client.chat.completions.create(
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
openai.APIConnectionError: Connection error.

相关代码:
`python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxx",
    base_url="https://api.siliconflow.cn/v1"
)

response = client.chat.completions.create(
    model="Qwen/Qwen3.7-72B-Instruct",
    messages=[{"role": "user", "content": "hello"}]
)
`

我已经尝试过:
1. 确认 API Key 正确且在有效期内
2. 网络可以正常访问 siliconflow.cn
3. 重启了电脑
`

### 为什么好
差的 Prompt 省略了关键的上下文信息（环境、错误堆栈、代码、已尝试的方法），AI 只能猜问题。好的 debug prompt = 运行环境 + 完整错误信息 + 相关代码 + 已尝试的排查步骤。AI 能排除你已经试过的方向，直接给出更深入的诊断。

---

## 4. 学习辅助

### 差的 Prompt
`
给我讲讲什么是 Transformer
`

### 好的 Prompt
`
学习请求

当前水平: 我了解基本 Python 编程和神经网络概念（知道什么是 forward pass、什么是 loss function），但从未读过 Transformer 论文

目标: 理解 Transformer 的核心思想，不需要理解每个数学公式

请用这种方式讲解:
1. 先用一句话类比解释 Transformer 解决的核心问题
2. 然后用一个具体例子（比如翻译"I love you" -> "我爱你"）走一遍流程
3. 解释 Attention 机制时，不要讲公式，用"每个词看所有其他词"的方式描述
4. 最后用 3 句话总结核心概念
5. 每段讲完问我"这部分清楚吗？"，如果有不明白的地方我会追问
`

### 为什么好
差的 Prompt 让 AI 猜你的水平、猜你的目的，结果往往是两种极端——太浅或太深。好的学习 prompt = 明确当前水平 + 目标 + 讲解风格 + 交互机制。AI 能精确匹配你的理解水平，而不是用一个泛泛的回答打发你。

---

## 5. 论文辅助

### 差的 Prompt
`
帮我读一下这篇论文，总结一下
`

### 好的 Prompt
`
论文阅读辅助

论文: [论文标题 / 粘贴 PDF 或 arXiv 链接]

我的背景:
- 熟悉 Python 和 PyTorch
- 了解 Attention 和 Transformer 基本概念
- 没做过 LLM 训练，对 RLHF 只有概念性理解

帮我从以下维度解读:
1. 核心贡献: 这篇论文解决了什么问题？用一句话概括
2. 方法概述: 核心思路是什么？不要细节公式
3. 关键实验: 主要实验结果是什么？跟什么 baseline 比？
4. 跟我当前工作的关联: 我现在在做一个 RAG 问答系统，这篇论文的哪些部分可以直接借鉴？
5. 阅读建议: 如果想进一步深入，应该重点读哪几个 section？参考文献里哪几篇是必读的？

优先回答 1-3，4-5 在回答完基础信息后再补充
`

### 为什么好
差的 Prompt 得到的摘要太泛，看完等于没看。好的论文 prompt = 明确你的背景 + 要读的论文 + 具体想知道什么 + 关联你自己的项目。AI 能从「给你翻译论文」变成「帮你理解论文和你的关联」。

---

## 6. 安全审查

### 差的 Prompt
`
检查这段代码安不安全
`

### 好的 Prompt
`
Security Review Request

审查目标: 这段代码是否适合在生产环境中处理用户输入

重点关注:
1. Prompt Injection —— 用户输入是否可能绕过系统指令
2. SQL Injection —— 是否有字符串拼接 SQL
3. XSS —— 输出是否有可能被注入 HTML/JavaScript
4. 敏感信息泄露 —— 是否可能暴露 API Key、路径、内部错误信息
5. 权限问题 —— 是否存在越权风险

Code:
`python
@app.post("/chat")
async def chat(user_input: str, session_id: str):
    prompt = f"You are a helpful assistant. User says: {user_input}"
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": prompt}
        ]
    )
    print(session_id)  # log
    return {"response": response.choices[0].message.content}
`

Output 格式:
- [Severity: Critical/High/Medium/Low]
- Issue description + 代码行号
- Fix suggestion
`

### 为什么好
差的 Prompt 给 AI 太多猜测空间。好的安全审查 prompt = 明确审查维度 + 具体的代码 + 结构化的输出格式。AI 能系统地覆盖每个安全风险维度，而不是随机发现几个问题。

---

## 常用 Prompt 设计原则总结

| 原则 | 说明 |
|------|------|
| **明确角色** | 告诉 AI 你是谁、让它扮演什么角色 |
| **提供上下文** | 不要假设 AI 知道你的项目背景、技术水平、目标 |
| **给出约束** | 语言版本、框架、格式、风格偏好 |
| **指定输出格式** | 结构化输出（列表、表格、分级）比自由文本更可控 |
| **提供反面示例** | 告诉 AI 不要做什么，比只告诉它要做什么更有效 |
| **单次单任务** | 一个 prompt 只做一件事，比多任务混在一起结果更好 |
| **迭代修正** | 第一次不完美正常，补充 context 继续调整即可 |
