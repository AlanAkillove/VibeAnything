# 01-Function Calling

> 读完本文你将了解：Function Calling 是什么、LLM 怎么调用外部函数、如何用 OpenAI SDK 实现 Tool Use、以及搜索/计算/数据库查询三个实际场景
> 注：本文假设你已有基本的 Python 和 OpenAI API 使用经验，代码示例基于 openai>=1.0 SDK。

---

## 你可能遇到过的问题

你用 LLM 做了一些有意思的 demo，但慢慢发现一些瓶颈：

- **AI 只能凭训练数据回答问题，无法查询实时信息**——问它今天天气、最新股价、某个 GitHub repo 现在的 star 数，它只能说「截至我的训练数据…」
- **AI 的数学计算经常翻车**——让它算个复杂点的表达式，有时候准有时候离谱，你不敢把关键数据交给它处理
- **AI 没法和你自己的系统交互**——你想让 LLM 查数据库里的用户订单、调用内部 API 发通知，但它只能生成文本，不会真的去执行
- **你试着用 prompt 让 AI 输出 JSON 格式来模拟函数调用**——结果格式经常跑偏，解析起来很痛苦

这些问题的本质是同一个：**LLM 默认只是一个文本生成模型，它没有能力「执行动作」或「获取外部信息」。Function Calling 就是解决这个问题的桥梁。**

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| Function Calling 就是让 AI 直接运行函数 | Function Calling 是 AI 输出一个函数调用声明（包含参数），由你的代码去实际执行。AI 本身不执行任何代码，它只「提议调用」 |
| Function Calling 需要特殊模型支持 | 主流模型（DeepSeek V4、Qwen3.7、GLM-5.2 等）都原生支持 Function Calling，通过 SiliconFlow 等国内平台即可使用 |
| 有了 Function Calling，AI 就无所不能 | Function Calling 只是「调用接口」，真正的能力和可靠性取决于你注册给 AI 的函数设计得怎么样 |
| 函数可以返回任意数据，AI 总会正确处理返回结果 | AI 处理函数返回值的质量取决于返回格式设计和 prompt 引导，返回太复杂或模糊的数据 AI 一样会懵 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| 想让 AI 查实时天气 | 定义一个 get_weather(location, date) 函数，接入天气 API，注册给 OpenAI 的 tools 参数 |
| 让 AI 算复杂表达式怕翻车 | 定义一个 calculate(expression) 函数，内部用 Python 的 eval() 或 sympy，让 AI 把计算委托给它 |
| 想通过对话查询数据库 | 定义一个 query_database(sql) 函数，让 AI 生成 SQL 语句并执行返回结果 |
| AI 输出的 JSON 格式总不对 | 用 function calling 代替手动 JSON prompt，SDK 会保证结构化输出 |

---

## 核心概念：Function Calling 究竟是什么

Function Calling（在 OpenAI 的 API 中也叫 Tool Use）是一种让 LLM **输出结构化函数调用声明**的能力。

理解它的关键在于：**LLM 收到你的请求后，不是直接生成「最终回答」，而是先判断是否需要调用某个外部函数，然后输出一个结构化的 function call 对象。你的应用层代码拿到这个对象后，执行对应的函数，再把结果返回给 LLM，让它基于函数的输出生成最终回答。**

```
你: "北京今天几度？"
  → LLM: function_call("get_weather", {location: "北京", date: "2026-06-19"})
  → 你的代码: 调用天气 API → 返回 "北京今天 28°C，晴"
  → LLM: "北京今天 28°C，天气晴朗☀️"
```

这里 LLM 的角色类似「调度员」：它理解你的意图，决定什么时候该使用什么工具，但工具的实际执行由你的代码完成。

### OpenAI SDK 的最小实现

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-你的API Key",
    base_url="https://api.siliconflow.cn/v1"  # 国内直接用，无需特殊网络
)

# 1. 定义一个函数描述（也就是 tool）
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定地点的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "城市名称，如 北京、上海"
                    },
                    "date": {
                        "type": "string",
                        "description": "日期，格式 YYYY-MM-DD"
                    }
                },
                "required": ["location", "date"]
            }
        }
    }
]

# 2. 实际执行函数（你的代码）
def get_weather(location: str, date: str) -> str:
    # 这里是你的天气 API 调用逻辑
    return f"{location} 在 {date} 的天气是 28°C，晴天"

# 3. 发送消息给 LLM
messages = [{"role": "user", "content": "北京今天几度？"}]
response = client.chat.completions.create(
    model="Qwen/Qwen3.7-72B-Instruct",  # 可换成任意 SiliconFlow 上的模型
    messages=messages,
    tools=tools,
    tool_choice="auto"
)
# 4. 检查 LLM 是否调用了函数
message = response.choices[0].message

if message.tool_calls:
    # 取出函数调用信息
    tool_call = message.tool_calls[0]
    function_name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)

    # 5. 执行函数
    if function_name == "get_weather":
        result = get_weather(**arguments)

    # 6. 把结果送回 LLM
    messages.append(message)
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": result
    })

    # 7. 让 LLM 生成最终回答
    final_response = client.chat.completions.create(
        model="Qwen/Qwen3.7-72B-Instruct",  # 可换成任意 SiliconFlow 上的模型
        messages=messages,
        tools=tools
    )
    print(final_response.choices[0].message.content)
```

这个流程的核心是 **6 个步骤循环**：定义工具 → 注册给 LLM → LLM 决定调用 → 你的代码执行 → 结果返回 LLM → LLM 生成回答。

### 注意几点

- `tools` 参数是一个 list，可以注册多个函数，LLM 会自动决定调用哪个（或哪个都不调用，直接回答）
- `tool_choice` 控制行为模式：`"auto"` 让 AI 自己判断；`"required"` 强制至少调一个；`{"type": "function", "function": {"name": "xxx"}}` 强制调用某个指定函数
- 函数参数用 JSON Schema 描述，结构越清晰 AI 调用越准确
- 如果你的函数返回很长的数据，可以只返回关键摘要，减少 token 消耗

---

## 实际场景一：AI + 搜索 API

让 AI 搜索实时信息是最常见的使用场景。

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "搜索互联网上的最新信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"},
                    "num_results": {"type": "integer", "description": "返回结果数量"}
                },
                "required": ["query"]
            }
        }
    }
]

def web_search(query: str, num_results: int = 5) -> list[dict]:
    # 接入 SerpAPI / Bing Search / Google Custom Search
    # 这里简化示例
    results = [
        {"title": f"关于 {query} 的结果 1", "url": "https://example.com/1"},
        {"title": f"关于 {query} 的结果 2", "url": "https://example.com/2"},
    ]
    return results
```

实战中要注意：搜索结果往往包含大量文本，建议在 function 内部做摘要再返回，避免 token 爆炸。

---

## 实际场景二：AI + 计算器

LLM 的数学能力不靠谱，把精确计算委托给专用工具。

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "执行数学计算，支持 + - * / ** sqrt 等运算",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {
                        "type": "string",
                        "description": "数学表达式，如 2**10 + 3.14 * 5"
                    }
                },
                "required": ["expression"]
            }
        }
    }
]

def calculate(expression: str) -> str:
    import math
    # 安全起见，限制可用的函数和操作
    allowed_names = {"abs": abs, "math": math, "float": float, "int": int}
    try:
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"
```

这里用了 eval() 但限制了 __builtins__，不是完全安全的沙箱。生产环境建议用 sympy 或 numexpr 这类专门库来安全求值。

---

## 实际场景三：AI + 数据库查询

让 AI 根据自然语言自动生成 SQL 并查询数据库。

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_sales_database",
            "description": "查询销售数据库。表名为 orders，字段包括 id, customer_name, product, amount, order_date",
            "parameters": {
                "type": "object",
                "properties": {
                    "sql": {
                        "type": "string",
                        "description": "要执行的 SQL 语句，必须是一个 SELECT 查询"
                    }
                },
                "required": ["sql"]
            }
        }
    }
]

def query_sales_database(sql: str) -> list[dict]:
    import sqlite3
    conn = sqlite3.connect("sales.db")
    cursor = conn.cursor()
    # 安全检查：只允许 SELECT
    if not sql.strip().upper().startswith("SELECT"):
        return [{"error": "只允许 SELECT 查询"}]
    cursor.execute(sql)
    columns = [desc[0] for desc in cursor.description]
    rows = cursor.fetchall()
    conn.close()
    return [dict(zip(columns, row)) for row in rows]
```

**安全提示**：生产环境中让 AI 生成 SQL 需要严格限制权限，最好用只读账号，并限定只能访问特定表。也可以在函数内部做 SQL 解析和校验，或者让 AI 先生成自然语言描述，再由固定逻辑映射到预定义的查询。

---

## 要点总结

- Function Calling 的本质是「LLM 输出结构化函数调用声明，你的代码执行它」——AI 不执行代码，它只提议调用
- 注册函数的 tools 参数用 JSON Schema 描述，描述越清晰 AI 调用越准确
- tool_choice 控制行为：auto（默认，AI 自己决定）、required（强制调用）、指定函数名（强制调某个）
- 函数返回结果后要追加到 messages（role 为 tool），让 LLM 基于结果生成回答
- 三个经典场景：搜索（获取实时信息）、计算（弥补 LLM 数学短板）、数据库（自然语言查询）
- 函数设计的关键是「返回精简但有信息量的结果」——返回太多原始数据只会浪费 token
