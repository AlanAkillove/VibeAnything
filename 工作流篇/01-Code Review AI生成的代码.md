# 01-Code Review AI生成的代码

> 读完本文你将了解：AI 生成的代码最常出什么问题，从安全性、性能、可维护性、边界条件四个维度怎么审查，一份可直接复用的 Code Review Checklist，以及让 AI 帮你做 Review 的技巧

---

## 你可能遇到过的问题

- **AI 写了一百多行代码，你扫了一遍看起来都对，跑起来第一分钟就崩了**
- **AI 生成的代码在测试环境跑得好好的，上线就被用户爆出了安全漏洞**
- **你让 AI 加一个功能，它改了三处代码，结果原来能用的功能坏了**
- **Review 时不知道重点在哪，只能凭感觉看，漏掉了真正致命的问题**
- **AI 生成的代码又长又绕，人读起来很吃力，但你不敢改，怕改坏了**

这些问题指向同一个核心事实：AI 生成的代码看起来「对」，但离「好」还差得很远。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| AI 写代码很少出错，Review 可以放松 | AI 的代码错误率在 10-30%，复杂逻辑场景更高 |
| 能跑就行，Review 只是走形式 | AI 生成的代码最擅长「表面正确」，最不擅长「边界条件」和「安全防护」 |
| AI 会考虑到所有边缘情况 | AI 倾向于处理典型路径，异常路径经常被忽略 |
| 代码能编译通过就没问题 | 编译通过只说明语法对，逻辑错误、性能隐患编译检查不出来 |
| Review AI 代码和人代码一样 | AI 代码的常见错误模式和人不同，Review 重点也不一样 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| 不知道 Review 从哪入手 | 按下文四个维度逐项检查：安全性、性能、可维护性、边界条件 |
| AI 代码太长不想读 | 让 AI 分段加注释，然后逐段 Review |
| 怕漏掉问题 | 用下文附带的 Checklist，逐条打勾 |
| 想用 AI 辅助 Review | 把 AI 代码喂给另一个 AI，指定「从安全角度找漏洞」|

---

## AI 代码的核心特征：表面正确，深处有坑

AI 生成的代码有一个显著模式：它擅长把「看起来正确」的代码写出来，但在真正的工程细节上经常翻车。这是因为大模型训练数据里，最常见的代码是「教学示例」和「开源项目」，而教学代码很少考虑安全防护、生产环境的高并发、跨边界数据处理。

理解这一点后，Review AI 代码就不能只看「逻辑对不对」，而要刻意去找那些 AI 天然不擅长的地方。

---

## 第一维度：安全性审查

安全是 AI 代码最薄弱的一环。AI 不会主动考虑恶意输入，你需要重点检查：

### SQL 注入与数据污染
\`\`\`python
# ❌ AI 常见写法 - 直接用用户输入拼接
query = f"SELECT * FROM users WHERE name = '{user_input}'"

# ✅ 正确写法 - 参数化查询
query = "SELECT * FROM users WHERE name = ?"
cursor.execute(query, (user_input,))
\`\`\`

AI 在简单示例里很少写出参数化查询，因为示例数据都是「可信的」。而生产环境里用户输入永远不可信。

### 路径遍历与文件操作
\`\`\`python
# ❌ AI 常见写法 - 未做路径白名单校验
with open(f"/data/{user_filename}", "r") as f:

# ✅ 正确写法 - 白名单或路径验证
allowed_files = {"report.pdf", "summary.csv"}
if user_filename not in allowed_files:
    raise PermissionError("File not allowed")
\`\`\`

### 密钥与凭证硬编码
AI 训练数据里包含大量教学代码，其中经常出现 \`API_KEY = "your-api-key"\` 这样的占位符。AI 会模仿这种模式，把密钥硬编码写在代码里。

\`\`\`python
# ❌ 审查时要注意
api_key = "sk-..."  # AI 可能从这里漏出真实密钥

# ✅ 应改为
api_key = os.environ.get("API_KEY")
\`\`\`

### 权限与访问控制
AI 生成的 API 端点默认不做鉴权的比例很高。Review 时要确认每个暴露的接口都有权限检查。

> **安全审查三问**：输入是否做了验证？输出是否过滤了敏感信息？每步操作有没有权限检查？

---

## 第二维度：性能审查

AI 默认生成「能跑就行」的代码，不会主动优化性能。你需要额外关注：

### 循环重复调用
\`\`\`python
# ❌ 循环内重复查询数据库
for user_id in user_ids:
    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    data.append(user)

# ✅ 批量查询一次
users = db.query(f"SELECT * FROM users WHERE id IN ({','.join(user_ids)})")
\`\`\`

### 不必要的重复计算
\`\`\`python
# ❌ 每次计算相同结果
result = []
for item in items:
    x = complex_calculation(item)  # 每次循环都算
    # ...

# ✅ 缓存计算结果
cache = {}
for item in items:
    if item.id not in cache:
        cache[item.id] = complex_calculation(item)
\`\`\`

### 内存使用不合理
AI 经常一次性把所有数据加载到内存，不考虑流式处理。

\`\`\`python
# ❌ 全部加载到内存
with open("large_file.csv") as f:
    all_data = f.readlines()  # 文件 10GB？内存直接爆

# ✅ 流式逐行处理
with open("large_file.csv") as f:
    for line in f:
        process(line)
\`\`\`

### 性能审查要点清单
- [ ] 有没有循环内做 IO 操作（数据库、API 调用、文件读写）
- [ ] 有没有不必要的重复计算
- [ ] 数据量大时会不会内存溢出
- [ ] 有没有 N+1 查询问题（ORM 场景最常见）
- [ ] 异步代码有没有忘记 await

---

## 第三维度：可维护性审查

AI 生成的代码在「让人类理解和维护」这一点上经常不及格。

### 命名模糊
\`\`\`python
# ❌ 无意义命名
def process_data(x, y):
    a = x + y
    return a * 1.5

# ✅ 自描述命名
def calculate_tax_with_surcharge(base_amount, surcharge_rate):
    taxed_amount = base_amount + surcharge_rate
    return taxed_amount * 1.5
\`\`\`

### 函数过长
AI 倾向于生成一个大函数完成所有事。Review 时注意函数是否超过 30 行，是否做了太多事。

### 魔法数字
\`\`\`python
# ❌ AI 经常直接使用数字
if status == 2:
    discount = price * 0.8

# ✅ 应定义常量或枚举
PAID_STATUS = 2
VIP_DISCOUNT_RATE = 0.8
\`\`\`

### 没有错误处理
\`\`\`python
# ❌ 假设一切正常
response = requests.get(url)
return response.json()

# ✅ 考虑异常
try:
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
except requests.RequestException as e:
    logger.error(f"Request failed: {e}")
    return None
\`\`\`

### 可维护性审查清单
- [ ] 变量/函数/类命名是否自描述
- [ ] 函数是否超过 30 行，是否需要拆分
- [ ] 有没有魔法数字需要提取为常量
- [ ] 有没有异常处理，还是假设一切正常
- [ ] 代码有没有必要的注释
- [ ] 有没有重复代码可以提取

---

## 第四维度：边界条件审查

这是 AI 最常翻车的地方。AI 善于处理「通常情况」，但「极端情况」经常被忽略。

### 空值与 None
\`\`\`python
# ❌ 未处理 None
user = get_user(user_id)
return user.name  # user 为 None 时报 AttributeError

# ✅ 安全访问
user = get_user(user_id)
return user.name if user else "Unknown"
\`\`\`

### 空列表与空集合
\`\`\`python
# ❌ 未处理空列表
average = sum(scores) / len(scores)  # scores 为空时 ZeroDivisionError

# ✅ 检查边界
average = sum(scores) / len(scores) if scores else 0
\`\`\`

### 极端大值与极端小值
\`\`\`python
# ❌ 未考虑溢出或精度
def calculate_discount(price, rate):
    return price * rate  # 处理超大金额时可能精度丢失

# ✅ 用 Decimal 处理货币
from decimal import Decimal
def calculate_discount(price, rate):
    return Decimal(str(price)) * Decimal(str(rate))
\`\`\`

### 并发与竞态条件
AI 默认写单线程代码，Review 时要确认：
- 多线程环境下的共享变量有没有加锁
- 数据库操作有没有事务保护
- 缓存更新有没有考虑一致性问题

---

## Code Review Checklist（可直接复用）

把这份 Checklist 放在项目根目录，每次 Review AI 代码时逐条检查：

### 功能性
- [ ] 代码逻辑是否正确，符合需求描述
- [ ] 核心功能在典型输入下是否工作
- [ ] 有没有遗漏功能点

### 安全性
- [ ] 所有用户输入是否做了验证和转义
- [ ] 数据库查询是否使用参数化
- [ ] 文件操作是否有路径白名单
- [ ] 密钥和敏感信息是否从环境变量读取
- [ ] 暴露的 API 端点是否有鉴权

### 性能
- [ ] 有没有循环内的 IO 调用
- [ ] 是否存在 N+1 查询
- [ ] 数据量大时内存是否可控
- [ ] 有没有不必要的重复计算
- [ ] 异步代码结构是否合理

### 可维护性
- [ ] 命名是否自描述
- [ ] 函数长度是否合理（小于 30 行优先）
- [ ] 有没有魔法数字
- [ ] 异常处理是否完整
- [ ] 代码风格是否和项目一致

### 边界条件
- [ ] 空值/None 是否处理
- [ ] 空列表/空集合是否考虑
- [ ] 输入长度极端情况是否处理
- [ ] 并发场景是否安全
- [ ] 类型是否正确（尤其 Python 动态类型）

---

## 用 AI 辅助 Review AI 代码

一个高效的技巧：**用一个 AI 生成的代码，交给另一个 AI 做 Review**。不同模型有不同的盲区，交叉审查效果更好。

### 实用 Prompt

\`\`\`
我有一段 AI 生成的 Python 代码，请从以下几个方面审查：

1. 安全性：是否有 SQL 注入、路径遍历、密钥泄露风险
2. 性能：是否有循环内 IO、N+1 查询、内存溢出风险
3. 可维护性：命名、函数长度、异常处理是否合理
4. 边界条件：空值、空列表、极端输入是否处理

请按优先级排序，先列出最严重的问题，再列出优化建议。
\`\`\`

### 结合人类判断
AI 辅助 Review 的局限在于：AI 不知道你的业务上下文。它可能标记出「安全的代码」为有风险，也可能漏掉和你的业务逻辑相关的问题。所以：

- 用 AI 做第一遍粗筛，标记可疑点
- 人工做第二遍，结合业务上下文做最终判断
- 把 Checklist 的结果记录下来，形成团队的 Review 知识库

---

## 你现在应该做什么？

1. 找一段最近 AI 生成的代码，用上面的四维度逐一过一遍
2. 把 Checklist 贴到项目里，下次 Review 直接拿来用
3. 让另一个 AI 审查这段代码，对比它发现的问题和你发现的问题
4. 把重复出现的问题记录下来，下次给 AI 写 Prompt 时就加上约束条件

---

**要点总结**
- AI 代码的特征是「表面正确，深处有坑」，安全性、边界条件是最薄弱环节
- Review 分四个维度：安全性、性能、可维护性、边界条件，每个维度有明确的检查重点
- 使用 Checklist 确保不遗漏，特别是循环内 IO 和空值处理这些高频率问题
- 用 AI 辅助 Review 是高效技巧，但最终判断需要结合业务上下文
- 把重复发现的问题加入 Prompt，让 AI 在下一次生成时就避免同类问题
