# 03-成本意识与Token管理

> 读完本文你将了解：AI API 调用的成本怎么算，不同模型的价格对比，怎么控制 Token 消耗，缓存策略、模型选择、批量处理等实用的省钱技巧

---

## 你可能遇到过的问题

- **一个月下来发现 AI API 账单比服务器费用还高**
- **开发阶段随便调 API 没感觉，上了线天天被用户调用，成本飞涨**
- **同样的功能，换了便宜点的模型效果差很多，不知道怎么选**
- **写 Prompt 时没注意 Token 数，最后发现一个 Prompt 几十万个 Token**
- **你不知道每个用户的 API 调用到底花了多少钱，没法衡量 ROI**

AI 开发有一个独特的成本结构——代码不花钱，但每次调用都花钱。如果不主动管理 Token 消耗，你的「免费 API Key」用完免费额度后，账单会让你措手不及。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| API 调用很便宜，不用在意 | 开发阶段用量小不觉得，生产环境一天几万次调用，一个月几百甚至上千美元 |
| 选最贵的模型效果最好 | 很多场景下小模型就能解决，大模型是浪费 |
| Token 就是字数，很好估算 | 中文 Token 和英文 Token 换算不一样，System Prompt 每次调用都算进去 |
| Prompt 写长了精度更高 | Prompt 越长精度不一定线性提升，但成本线性增长 |
| 缓存没用，反正每次回答不一样 | 很多应用场景是重复查询，缓存命中率很高 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| 不知道 API 花了多少钱 | 去 OpenAI Dashboard 看 Usage 页面，设定预算告警 |
| 觉得 Prompt 太长了 | 检查 System Prompt 的长度，去掉不必要的历史会话 |
| 想省钱但不想牺牲质量 | 先用便宜的模型做原型，只在关键步骤用大模型 |
| 用户量大了成本暴涨 | 实现 Response 缓存 + 批量处理 |
| 中文场景 Token 算不准 | 用 tiktoken 库实测 Token 数，不要靠估算 |

---

## Token 的成本模型

AI API 的计费方式很简单：**按 Token 计费，输入和输出分开计价**。

### Token 到底是什么

一个 Token 在英文里大约对应 0.75 个词，中文大约对应 1-2 个汉字。这段文字大约 100 个 Token。更好用的方法是直接用 OpenAI 的 Tokenizer 工具或 tiktoken 库来算。

### 成本公式

```
每次调用的成本 = (Input Tokens × Input Price) + (Output Tokens × Output Price)
```

注意两个关键点：
1. **输入和输出价格不同**。对大多数模型，输出的价格比输入贵 3-4 倍（因为生成比理解更消耗算力）
2. **System Prompt 每次调用都算在 Input Token 里**。如果你的 System Prompt 是 2000 Token，每次调用都扣这 2000

---

## 主流模型价格对比（截至 2025 年中）

以下价格基于官方公布的最新定价，单位是每 1M Token（即每百万 Token）的价格：

| 模型 | 输入价格 / 1M Token | 输出价格 / 1M Token | 适用场景 |
|------|-------------------|-------------------|---------|
| GPT-4o | $2.50 | $10.00 | 复杂推理、代码生成、多模态 |
| GPT-4o Mini | $0.15 | $0.60 | 日常问答、分类、摘要 |
| Claude 3.5 Sonnet | $3.00 | $15.00 | 大型代码任务、长文档处理 |
| Claude 3.5 Haiku | $0.25 | $1.25 | 简单任务、快速响应 |
| Claude 4.5 Sonnet | $15.00 | $75.00 | 前沿推理、复杂研究 |
| DeepSeek-V3 | $0.27 | $1.10 | 编程、通用任务 |
| DeepSeek-V4 Think | $0.55 | $2.19 | 深度推理（思考链） |

> 注意：API 价格经常调降，以上价格仅供趋势参考，实际以各平台官网最新公告为准。从趋势看，模型价格在快速下降，成本逐年降低。

### 价格差异意味着什么

假设你每天处理 100 万个输入 Token、生成 20 万个输出 Token：

| 用哪个模型 | 每天成本 | 每月成本 |
|-----------|---------|---------|
| GPT-4o | $2.50 + $2.00 = $4.50 | ~$135 |
| GPT-4o Mini | $0.15 + $0.12 = $0.27 | ~$8 |
| Claude 3.5 Haiku | $0.25 + $0.25 = $0.50 | ~$15 |

对于很多场景，Mini 或 Haiku 的效果完全够用，而价格差了 10-20 倍。

---

## 控制 Token 消耗的七大策略

### 策略一：选对模型等级

模型分层使用是最省钱的做法：

| 任务类型 | 推荐的模型等级 |
|---------|--------------|
| 简单分类、关键词抽取 | Mini / Haiku |
| 内容摘要、翻译 | Mini / Haiku |
| 代码生成 | 中等模型（4o, Sonnet） |
| 复杂推理、多步骤分析 | 顶级模型（o1, DeepSeek V4 Think） |
| 结构化输出 | 中等模型，配合 JSON mode |

**核心原则**：能用小模型解决的就不要用大模型。大模型只用在必要的地方。

### 策略二：精简 System Prompt

很多人习惯在 System Prompt 里写长篇大论的背景信息，但这些信息每次调用都会被计算 Token。

```bash
# 实际案例：一个摘要应用
# 原来 1500 Token 的 System Prompt
# 去掉不必要的背景说明和示例后 → 300 Token
# 省了 80% 的输入成本
```

**精简技巧**：
- 去掉示例（Example）——改成在 User Prompt 里给
- 去掉不必要的格式说明
- 把背景信息精简到最核心的部分

### 策略三：控制输出长度

输出比输入贵 3-4 倍，所以控制输出长度是省钱的关键。

```python
# ❌ 不限制输出长度
response = client.chat.completions.create(
    model="deepseek-ai/DeepSeek-V4-Flash",  # 可换成任意 SiliconFlow 上的模型
    messages=messages,
    # 没有 max_tokens → AI 可能回复很长
)

# ✅ 限制输出长度
response = client.chat.completions.create(
    model="deepseek-ai/DeepSeek-V4-Flash",  # 可换成任意 SiliconFlow 上的模型
    messages=messages,
    max_tokens=500,  # 明确控制最长的回复 Token 数
)
```

### 策略四：实现 Response 缓存

对于重复的查询，缓存可以省去 90-100% 的 API 调用。

**哪些场景适合缓存**：
- 知识库问答（同样的问法重复出现）
- 分类任务（相同的输入会重复）
- 配置验证（相同的规则多次判断）

```python
import hashlib
import json
from functools import lru_cache

# 简单的内存缓存
@lru_cache(maxsize=1000)
def cached_llm_call(prompt_hash: str):
    # 实际的 API 调用
    pass

def get_llm_response(prompt: str):
    prompt_hash = hashlib.md5(prompt.encode()).hexdigest()
    return cached_llm_call(prompt_hash)
```

更完善的方案是使用 Redis 等外部缓存，加上 TTL 过期策略。

### 策略五：批量处理

把多个任务合并到一个 API 请求里处理，可以减少调用次数。

\`\`\`python
# ❌ 逐个处理（多次 API 调用）
for comment in comments:
    result = classify_sentiment(comment)

# ✅ 批量处理（一次 API 调用处理多个）
batch_prompt = f"分析以下评论的情感倾向（正面/负面/中性）：\n"
for i, comment in enumerate(comments):
    batch_prompt += f"{i+1}. {comment}\n"
batch_prompt += "以 JSON 数组形式输出结果"
\`\`\`

批量处理的注意事项：
- 一次不要放太多，5-10 条为宜，否则输出太长容易出错
- 要求结构化输出（JSON），方便解析
- 注意批处理的 Token 总和不能超过模型的上限

### 策略六：用 Token 计数做预算

在代码里预估算每次调用的 Token 数，超过阈值时做降级处理：

```python
import tiktoken

def estimate_tokens(text: str, model: str = "deepseek-ai/DeepSeek-V4-Flash") -> int:
    encoding = tiktoken.encoding_for_model("gpt-4o")  # tiktoken 用于估算，模型名不影响计数
    return len(encoding.encode(text))

# 在调用前做预算
system_prompt = "你的角色是..."
user_input = "..."

input_tokens = estimate_tokens(system_prompt + user_input)
estimated_cost = (input_tokens / 1_000_000) * INPUT_PRICE + 0.005  # 预估输出

if estimated_cost > 0.01:  # 单次超过 1 分钱，考虑用小模型
    model = "deepseek-ai/DeepSeek-V4-Flash"  # 可换成任意 SiliconFlow 上的模型
```

### 策略七：设置预算告警

这是最后的防线。在 OpenAI Dashboard 上设置：

1. **月度预算**：设置上限，超了自动暂停 API 调用
2. **用量告警**：用到 50%/80%/90% 时发邮件或 webhook 通知
3. **额度限制**：给 API Key 设额度，防被盗或滥用

---

## 模型选择的实际策略

### 分层模型架构

```
                     ┌─ 简单任务 → DeepSeek V4 Flash（极低成本）
用户请求 → 路由判断 ─┼─ 中等任务 → Qwen3.7-72B-Instruct（均衡）
                     └─ 复杂任务 → DeepSeek V4 Pro（深度推理）
```

实现一个路由函数，根据任务复杂度选模型：

```python
def select_model(task_type: str) -> str:
    """根据任务类型选择最经济的模型（国内可用模型示例）"""
    model_map = {
        "classification": "deepseek-ai/DeepSeek-V4-Flash",      # 轻量任务，极低成本
        "extraction": "deepseek-ai/DeepSeek-V4-Flash",           # 轻量任务
        "summarization": "Qwen/Qwen3.7-72B-Instruct",            # 中等任务
        "code_generation": "Qwen/Qwen3.7-72B-Instruct",          # 代码生成
        "complex_reasoning": "deepseek-ai/DeepSeek-V4",          # 复杂推理
        "creative_writing": "deepseek-ai/DeepSeek-V4",           # 创意写作
        # 以上模型均可在 SiliconFlow 上使用，可自由切换
    }
    return model_map.get(task_type, "deepseek-ai/DeepSeek-V4-Flash")
```

### 混合模型策略

在同一个工作流里，不同阶段用不同模型：

```
输入 → 小模型做分类 → 大模型做核心推理 → 小模型做格式化输出
```

这样既保证了核心结果的质量，又把大部分 Token 消耗集中在便宜的模型上。

---

## 成本监控

有没有省钱，数据说了算。建议记录每次 API 调用的成本数据：

```python
# 记录日志
log_entry = {
    "timestamp": datetime.now().isoformat(),
    "model": model,
    "input_tokens": input_tokens,
    "output_tokens": output_tokens,
    "cost": cost,
    "user_id": user_id,
    "task_type": task_type,
}

# 定期分析
# - 哪个用户消耗最多
# - 哪种任务成本最高
# - 哪个模型性价比最优
```

把这些数据可视化，你就能看到真正的成本分布，找到最有优化空间的方向。

---

## 你现在应该做什么？

1. 去你的 API 控制台查一下上个月的花费，搞清楚钱花在哪了
2. 检查你的 System Prompt 长度，去掉不必要的部分
3. 给每个 API 调用加上 `max_tokens`，限制输出长度
4. 评估你的应用场景是否可以用 Mini / Haiku 替代当前的大模型

---

**要点总结**

1. Token 成本由输入和输出两部分构成——输出比输入贵 3-4 倍，System Prompt 每次调用都全额计入输入
2. 不同模型价格差异可达数十倍——日常任务用 Mini/Haiku 级模型完全够用，只在关键步骤用大模型
3. 七个省钱策略按优先级：选对模型等级 > 精简 System Prompt > 控制输出长度 > 实现缓存 > 批量处理 > Token 预算 > 设置告警
4. 分层模型架构：简单任务走小模型、中等任务走标准模型、复杂任务走大模型——按任务复杂度智能路由
5. 提示缓存是长上下文场景最核心的省钱手段——重复固定内容可降至原价 1 折，但只有放在开头的重复前缀才能命中

---

## 更进一步

- [OpenAI API Pricing](https://openai.com/api/pricing/) — 实时价格表
- [tiktoken GitHub](https://github.com/openai/tiktoken) — OpenAI 官方 Token 计数库
- **上一篇文章**：[02-从原型到生产](./02-从原型到生产.md)
- **下一篇文章**：[04-AI开发的验证闭环](./04-AI开发的验证闭环.md) — 每步验证避免浪费 Token
- **相关阅读**：[02-Token与上下文窗口](../基石篇/02-Token与上下文窗口.md) — Token 和上下文窗口的完整基础
- **相关阅读**：[03-API获取与成本](../中国大陆生态篇/03-API获取与成本.md) — 国内 API 获取和定价对比
- **相关阅读**：[03-价格与选型](../行业篇/03-价格与选型.md) — 更详细的模型价格与选型策略
