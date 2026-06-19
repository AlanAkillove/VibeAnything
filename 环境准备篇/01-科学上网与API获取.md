# 01-科学上网与API获取

> 读完本文你将了解：为什么要用 API 而不是只用网页版、国内有哪些免梯子可用的 AI API 平台、怎么注册和获取 API Key、以及怎么用代码快速调用大模型
>
> 注：**国内主流平台均无需科学上网**，仅国际平台（OpenAI、Anthropic）需要特殊网络环境。

---

## 你可能遇到过的问题

你用过 DeepSeek、豆包、Kimi 或 ChatGPT 的网页版聊天，觉得 AI 确实有用。但当你想要更进一步，用代码做项目、搭工具时，遇到了这些情况：

- **你想在代码里调用 AI**，但不知道去哪里申请 API，也不知道选哪个模型
- **你按网上的旧教程写了代码**，但报错 `401 unauthorized`——没有 API Key，或者模型名已经过时
- **你听说要用 OpenAI 必须搭梯子**，但不知道怎么搞，也担心不稳定、有风险
- **你注册海外平台**，发现需要国际信用卡，学生根本办不了
- **开源模型只能本地跑吗？** 自己电脑显卡不够，有没有云端托管的开源模型 API

这些问题大部分不是因为"技术难"，而是因为**不知道去哪儿、用什么、怎么起步**。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| 用 AI 编程只能用 OpenAI | 国内有多个平台提供 OpenAI 兼容的 API，不需要梯子，支持支付宝，模型选择非常丰富 |
| API Key 就像账号密码，可以用在任何地方 | API Key 绑定你的账户计费，**绝对不能分享**，更不能传到公共代码仓库 |
| 调用 AI API 很贵 | 对学习和个人项目来说，很多平台有免费额度，国产开源模型 API 单价低至百万 Token 几分钱，成本可以忽略 |
| 搞 AI 必须有国际信用卡 | 国内平台（SiliconFlow、DeepSeek、阿里云百炼）都支持支付宝/微信，学生零门槛使用 |
| 开源模型只能本地部署 | 主流开源模型（Qwen、DeepSeek、GLM、Llama）都有云端托管 API，不用本地显卡也能调用 |

---

## 新手常见问题快速解答

| 你遇到的问题 | 解决方案 |
|-------------|---------|
| 不知道去哪申请 API Key | 看下面的「推荐路径」，优先选国内平台注册 |
| 注册了但不知道怎么调用 | 所有主流平台都兼容 OpenAI SDK，代码几乎一样，只改 `base_url` 和模型名即可 |
| 海外平台无法注册（信用卡、手机号）| 先用国内平台起步，功能完全满足学习和开发需求，后续有需要再考虑国际平台 |
| 不知道怎么在代码里安全保存 API Key | 用 `.env` 文件管理，绝对不要硬编码在代码里 |
| 不确定该选哪个模型起步 | 新手推荐 DeepSeek V4 Flash / Qwen3.7，性价比高、中文好、能力够用 |

---

## 基本概念：API Key 是什么？

API Key 是调用 AI 服务的身份凭证。每次调用 LLM API 时，你需要把你的 Key 随请求一起发送，服务端验证通过后才会返回结果，并按调用量扣费。

它和账号密码的区别：
- 账号密码：用于登录网页控制台，管理账户、查看账单、生成密钥
- API Key：专门用于**程序与服务端之间的通信**，不用于登录网页

一个账号可以生成多个 API Key，分配给不同的项目使用。如果某个 Key 意外泄露，可以在控制台单独吊销它，不会影响其他 Key 和账户本身的安全。

---

## 推荐起步路径

### 路径 A：SiliconFlow（硅基流动）—— 国内新手首选

这是最推荐给新手的方案：无需梯子、支持支付宝、有免费额度、模型最全、完全兼容 OpenAI 接口格式，一份代码可以切换所有主流开源模型。

**步骤 1：注册账号**

打开 [cloud.siliconflow.cn](https://cloud.siliconflow.cn)，用手机号注册即可，国内用户零门槛。部分免费模型需要完成实名认证，按平台提示操作即可。

**步骤 2：获取 API Key**

登录后，在左侧菜单找到「API 密钥」页面，点击「新建 API 密钥」，复制生成的密钥妥善保存。

**步骤 3：用代码测试调用**

SiliconFlow 完全兼容 OpenAI 的 Python SDK，只需要修改 `base_url` 即可调用。

先安装依赖：
```bash
pip install openai python-dotenv
```

基础调用示例：
```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 SiliconFlow API Key",
    base_url="https://api.siliconflow.cn/v1"
)

response = client.chat.completions.create(
    model="Qwen/Qwen3.7-72B-Instruct",  # 可选：deepseek-ai/DeepSeek-V4、GLM-5.2 等
    messages=[{"role": "user", "content": "你好，请用一句话介绍你自己"}]
)

print(response.choices[0].message.content)
```

如果使用 DeepSeek V4（支持 Think 推理模式）这类推理模型，还可以在流式输出中获取模型的思考过程：
```python
response = client.chat.completions.create(
    model="deepseek-ai/DeepSeek-V4",
    messages=[{"role": "user", "content": "用逐步推理的方式计算 123+456*789"}],
    stream=True
)

for chunk in response:
    if not chunk.choices:
        continue
    delta = chunk.choices[0].delta
    if delta.reasoning_content:  # 输出模型的内部思考过程
        print(delta.reasoning_content, end="", flush=True)
    if delta.content:  # 输出最终答案
        print(delta.content, end="", flush=True)
```

平台覆盖主流开源模型：DeepSeek V4、Qwen3.7 全系列、GLM-5.2、Llama 4、Step 等上百款模型，可在「模型广场」查看完整列表和实时价格。

### 路径 B：DeepSeek 官方平台

如果你主要使用 DeepSeek 系列模型，可以直接注册官方平台，体验最新版本的完整能力与提示缓存优惠。

**步骤 1：注册账号**

打开 [platform.deepseek.com](https://platform.deepseek.com)，使用邮箱或手机号注册，支持支付宝充值。

**步骤 2：获取 API Key**

登录后在左侧菜单找到「API Keys」，创建新的密钥并复制保存。

**步骤 3：调用示例**

DeepSeek API 完全兼容 OpenAI 格式，仅需修改接口地址和模型名：
```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 DeepSeek API Key",
    base_url="https://api.deepseek.com"
)

response = client.chat.completions.create(
    model="deepseek-v4-flash",  # 高性价比首选，同系列还有 deepseek-v4-pro 旗舰版
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

> 提示：V4 全系列标配 1M 上下文，且支持提示缓存机制，重复上下文场景下输入成本可降低 90% 以上，具体价格以官方控制台为准。

### 路径 C：国际平台（OpenAI、Anthropic）

如果需要使用 GPT、Claude 等海外旗舰模型，需要满足：
1. 可稳定访问境外 API 的网络环境
2. 支持境外支付的信用卡（Visa / Mastercard）
3. 在对应平台注册并绑定支付方式

以 OpenAI 为例，调用代码如下：
```python
from openai import OpenAI

client = OpenAI(
    api_key="你的 OpenAI API Key"
    # 默认 base_url 为 https://api.openai.com/v1，无需手动修改
)

response = client.chat.completions.create(
    model="gpt-5.5",  # 2026 年最新旗舰通用模型
    messages=[{"role": "user", "content": "你好"}]
)

print(response.choices[0].message.content)
```

**注意：** 国际平台 API 在国内无法直接访问，网络不通时会出现 `ConnectionError` 报错。学生群体不推荐优先折腾国际平台，国产模型已能满足绝大多数学习和开发需求。

---

## 安全第一：怎么保护你的 API Key

这是新人最容易踩的坑，一旦密钥泄露可能被盗刷产生高额费用，请务必遵守以下规范。

### 正确做法：用 .env 文件管理密钥

在项目根目录创建 `.env` 文件，写入你的密钥：
```env
# .env 文件
SILICONFLOW_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxxxxxx
```

在代码中通过 `python-dotenv` 读取，永远不要把密钥直接写在代码里：
```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()  # 自动读取 .env 文件中的环境变量

client = OpenAI(
    api_key=os.getenv("SILICONFLOW_API_KEY"),
    base_url="https://api.siliconflow.cn/v1"
)
```

同时在项目根目录添加 `.gitignore` 文件，排除敏感文件，**绝对避免误提交到代码仓库**：
```gitignore
# .gitignore 文件
.env
.vscode
__pycache__
```

### 绝对禁止的行为
- ❌ 把 API Key 硬编码写在代码里，并上传到 GitHub / Gitee 等公开仓库
- ❌ 在截图、博客、社群聊天中暴露完整 API Key
- ❌ 把自己的 Key 分享给他人使用
- ❌ 在对话 prompt 中把 API Key 发给 AI

### 泄露后的补救措施
如果发现 Key 泄露，立刻去对应平台的控制台**吊销该密钥**，生成新的密钥替换使用，一般几分钟内即可完成止损。

---

## 你现在应该做什么？

1. 选一个国内平台（推荐 SiliconFlow）完成注册
2. 创建第一个 API Key
3. 复制上面的 Python 代码，配置好密钥，运行一次验证是否成功
4. 运行成功后，尝试切换不同模型（比如 DeepSeek V4、GLM-5.2），感受效果差异
5. 配置 `.env` 和 `.gitignore`，养成安全管理密钥的习惯

---

## 更进一步

- [SiliconFlow 官方快速开始文档](https://docs.siliconflow.cn/cn/userguide/quickstart) — 中文官方教程，包含更多语言的调用示例
- [DeepSeek 官方平台文档](https://platform.deepseek.com/docs) — DeepSeek 模型的完整 API 说明
- [OpenAI API 快速开始](https://platform.openai.com/docs/quickstart) — 国际平台参考（境外站点，国内访问可能受限）

---

**要点总结**
- API Key 是调用 AI 服务的身份凭证，**不等于账号密码**，泄露会直接产生费用风险
- 国内新手首选 **SiliconFlow**：免梯子、支持支付宝、模型全、完全兼容 OpenAI 接口
- API Key **绝对不能硬编码**在代码里，必须用 `.env` 文件管理，并配置 `.gitignore` 避免提交
- 国内平台与国际平台的调用代码几乎完全一致，仅 `base_url` 和模型名不同，切换成本极低
- 密钥泄露后第一时间去平台控制台吊销，生成新密钥替换