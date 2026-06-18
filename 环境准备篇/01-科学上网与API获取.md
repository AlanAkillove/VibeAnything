# 01-科学上网与API获取

> 读完本文你将了解：为什么需要 API 而不是只用网页版、国内有哪些可用的 AI API 平台、怎么注册和获取 API Key、以及怎么用代码调用 AI

---

## 你可能遇到过的问题

你用过 ChatGPT、DeepSeek 或 Kimi 的网页版聊天，觉得 AI 确实有用。但当你想要更进一步时，遇到了这些情况：

- **你想在代码里调用 AI**，但不知道去哪里申请 API
- **你按网上的教程写了代码**，但报错 `401 unauthorized`——没有 API Key
- **你听说要搭梯子才能用 OpenAI**，但不知道怎么搞，也担心不稳定
- **你注册了某些平台**，但发现需要国际信用卡，学生根本办不了

这些问题大部分不是因为"技术难"，而是因为**不知道去哪儿、怎么办**。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| 用 AI 编程只能用 OpenAI | 国内有多个平台提供 OpenAI 兼容的 API，不需要梯子也能用 |
| API Key 就像账号密码，可以用在任何地方 | API Key 绑定的是"谁在调用"，**不能分享**，也不能传到公共代码仓库 |
| 调用 AI API 很贵 | 对学习和个人项目来说，很多平台的免费额度完全够用，费用低到可以忽略 |
| 搞 AI 必须有国际信用卡 | 国内平台（SiliconFlow、DeepSeek、阿里云百炼）都支持支付宝 |

---

## 遇到这些问题怎么办？

| 你遇到的问题 | 现在可以怎么做 |
|-------------|---------------|
| 不知道去哪申请 API Key | 看下面的"推荐路径"，选一个平台注册 |
| 注册了但不知道怎么调用 | 用 OpenAI 兼容的 SDK——代码一样，改个 base_url 就行 |
| 海外平台无法注册（信用卡、手机号）| 先用国内平台（SiliconFlow、DeepSeek），以后再考虑国际平台 |
| 不知道怎么在代码里安全地保存 API Key | 用 `.env` 文件，不要硬编码在代码里 |
| 不确定该用哪个平台的哪个模型 | 选 DeepSeek 的最新 API（便宜、中文好）起步，摸索后再换 |

---

## 基本概念：API Key 是什么？

API Key 是识别调用者身份的密钥。每次调用 LLM API 时，你需要把你的 Key 发过去，服务端验证通过后才返回结果。

它和账号密码的区别：
- 账号密码：用于登录网页控制台
- API Key：用于**程序之间的通信**

一个账号可以生成多个 API Key，用于不同的项目。如果某个 Key 泄露了，可以在控制台吊销它，不影响其他 Key 和你的账号。

---

## 推荐路径

### 路径 A：SiliconFlow（硅基流动）—— 国内首选

这是最推荐给新手的方案。无需梯子、支持支付宝、有免费额度。

**步骤 1：注册账号**

打开 [cloud.siliconflow.cn](https://cloud.siliconflow.cn)，用手机号注册（免费模型需要完成实名认证）。

**步骤 2：获取 API Key**

登录后，进入"API 密钥"页面，点击"新建 API 密钥"，复制生成的密钥（通常在左侧菜单的"API 密钥"页面）。

**步骤 3：用代码测试**

SiliconFlow 提供 OpenAI 兼容的 API，所以你可以直接用 OpenAI 的 Python SDK 来调用，只需要改一个地址：

```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 API Key",
    base_url="https://api.siliconflow.cn/v1"
)

# 如果还没装依赖，先运行：pip install openai python-dotenv

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-72B-Instruct",  # 免费可用的模型
    messages=[{"role": "user", "content": "你好，请用一句话介绍你自己"}]
)

print(response.choices[0].message.content)
```

SiliconFlow 上可以用的模型包括 DeepSeek、Qwen、GLM 等多个系列。选择模型时，在控制台查看可用列表即可。

### 路径 B：DeepSeek 官方平台

如果只想用 DeepSeek 的模型，可以直接在 DeepSeek 官方注册。

**步骤 1：注册**

打开 [platform.deepseek.com](https://platform.deepseek.com)，注册账号。

**步骤 2：获取 API Key**

登录后在左侧菜单找到 API Keys，创建新的 Key。

**步骤 3：调用**

```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 DeepSeek API Key",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-chat"  # 在 DeepSeek 控制台查看当前可用的模型名,
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

注意 DeepSeek 的 API 也是 OpenAI 兼容的——代码几乎一模一样，只是 base_url 和 model 名不同。

### 路径 C：国际平台（OpenAI、Anthropic）

如果需要使用 GPT-4o 或 Claude 等国际模型，需要：

1. 稳定的网络环境（可以访问 api.openai.com 和 api.anthropic.com）
2. 国际支付方式（Visa/Mastercard）
3. 在相应平台注册并绑定支付

以 OpenAI 为例：

```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 OpenAI API Key"
    # base_url 使用默认值 https://api.openai.com/v1
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

**注意：** 国际平台的 API Key 通常不能在国内直接用。如果网络不通，会看到类似 `requests.exceptions.ConnectionError` 的错误。

---

## 安全第一：怎么保护你的 API Key

这是新人最容易踩的坑。

### 正确做法

创建一个 `.env` 文件放在项目根目录：

```
OPENAI_API_KEY=sk-your-key-here
SILICONFLOW_API_KEY=sk-your-key-here
```

在你的 Python 代码中读取：

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()  # 读取 .env 文件

client = OpenAI(
    api_key=os.getenv("SILICONFLOW_API_KEY"),
    base_url="https://api.siliconflow.cn/v1"
)
```

### 常见错误

- **把 API Key 写到代码里并上传到 GitHub**——会有人扫描你的 Key 去调用，月底收到天价账单
- **在 prompt 中把 API Key 发给 AI**——AI 对话内容可能被服务端记录或用于训练，Key 可能被泄露
- **截图分享代码时忘了打码 API Key**——Key 一旦公开就要立即吊销

### 补救措施

如果 Key 泄露了，立刻去平台控制台吊销该 Key，生成一个新的。

---

## 你现在应该做什么？

1. 选一个平台（推荐 SiliconFlow）注册
2. 创建一个 API Key
3. 复制上面的 Python 代码，把 Key 填进去，跑一次看是否成功
4. 如果成功，试试换成不同的模型和 prompt
5. 建立一个 `.env` 文件管理你的 Key

---

## 更进一步

- [SiliconFlow 快速开始](https://docs.siliconflow.cn/cn/userguide/quickstart) — 官方中文文档，包含更多代码示例
- [OpenAI API 快速开始](https://platform.openai.com/docs/quickstart) — 国际平台参考

---

**要点总结**
- API Key 是调用 AI 服务的身份凭证，**不是账号密码**
- 国内首选 **SiliconFlow**（硅基流动）：免梯子、支持支付宝、OpenAI 兼容
- API Key **不要硬编码**在代码里——用 `.env` 文件管理
- 国内平台和国际平台的调用代码几乎一样，只是 `base_url` 不同
- Key 泄露后立刻在平台控制台吊销

