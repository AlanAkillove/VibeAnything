# 02-Codex 与 Cursor 实战

> 读完本文你将了解：Cursor 和 Codex CLI 的安装配置方法、两个工具在真实项目中的操作流程、常用命令和技巧、以及什么场景更适合用哪一个
> 注：本文以动手实操为主，以 TypeScript/React 项目为例，但你完全可以用到任何编程语言上。

---

## 你可能遇到过的问题

你已经了解了 AI 编码工具的大致分类，决定试试 Cursor 和 Codex CLI，结果上手第一天就卡住了：

- **安装好 Cursor 打开项目，AI 完全不懂我的代码**——问它一个函数在哪儿定义的，它瞎编了一个路径
- **在 Codex CLI 里说「帮我把这个文件改成支持深色模式」**——结果它把整个 API 路由改了，跑起来全是 404
- **不知道什么时候该用 Cursor 的 Chat，什么时候该用 Composer**——在不同模式下反复试，效率反而低了
- **Codex CLI 改完文件没通知我**——我完全不知道它动了哪些位置，合并代码时一团乱
- **两个工具都试了一圈，感觉不如老老实实用 VS Code + 手动写**——这是最危险的信号，说明还没找到正确的使用姿势

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| 装好 Cursor 就能自动理解项目 | Cursor 需要时间建立 Codebase Index——刚打开大项目的前几分钟，AI 对代码的理解是「盲人摸象」 |
| Cursor 的 Chat 和 Composer 只是名字不同 | Chat 适合单文件问答和修改，Composer 适合跨文件编辑和重构。用反了会事倍功半 |
| Codex CLI 会自动知道我的项目结构 | Codex CLI 每一次对话都是在文件系统层面操作，它不知道你的项目「约定」——需要你主动告诉它架构信息 |
| 一个工具走到底 | Cursor 做日常编码+Codex CLI 做快速脚本和实验，这是很多人的实际组合方案 |
| 用上 AI 工具就不需要自己看代码了 | 工具生成代码的能力很强，但**审查和验证代码的能力没有提高**——你必须保持对代码的理解和掌控 |

---

## 常见问题快速解决

| 你遇到的问题 | 现在可以怎么做 |
|-------------|---------------|
| Cursor 刚打开项目，AI 乱猜代码 | 等右下角 Indexing 进度条走完（大型项目约 1-5 分钟），或者手动触发重新索引：Cmd+Shift+P → "Cursor: Reindex" |
| Codex CLI 改多了不知道改了哪 | 每次修改后先 `git diff` 看一眼改动，确认无误再继续下一步 |
| Cursor 的 AI 建议代码风格和自己的不一致 | 在 `.cursorrules` 文件中定义自己的代码风格、命名规范、框架约定 |
| Codex CLI 对话太长，上下文溢出 | 每完成一个小功能就结束对话、git commit，然后开新对话继续——不要一个对话写到底 |
| Cursor 总是建议改不需要改的文件 | 用 @file 语法精确指定要操作的文件，别让 AI 自己猜范围 |

---

## Cursor 实战：从安装到项目级操作

### 安装和初始设置

1. 去 [cursor.com](https://cursor.com) 下载对应系统的版本
2. 安装后登录你的 GitHub 账号——免费版有 2000 次 Composer 补全和 50 次慢速高级请求/月
3. 打开设置（Cmd+Shift+P → "Preferences: Open Settings"），关键设置项：
   - **Cursor Tab**：开启 Tab 补全（默认开），这是最有生产力的功能
   - **Cursor Chat**：设置默认模型（GPT-4o 日常够用，Claude Sonnet 适合复杂推理）
   - **Enable Codebase Indexing**：确认已开启，否则 Cursor 不知道你的项目结构
4. 创建 `.cursorrules` 文件放在项目根目录，内容示例：

```
You are a senior TypeScript/React developer.
- Use functional components with hooks
- Use named exports, not default exports
- Use Tailwind CSS for styling
- Use Zod for runtime validation
- Write unit tests for all utility functions
- Prefer concise, readable code over overly abstract patterns
```

这个文件告诉 Cursor 你的项目约定，AI 输出质量会因此大幅提升。

### Cursor 的三种模式

**Chat（对话模式）**：Cmd+L 触发
- **用途**：问问题、读代码、改少量文件
- **典型操作**：「@file:utils.ts 解释这个函数在做什么」→看完后「帮我在 findUser 函数里加一个 null check」
- **注意**：一次只改一两个文件时用这个，别让它同时改 10 个文件

**Composer（编辑器模式）**：Cmd+I 触发
- **用途**：跨文件编辑、重构、创建新功能
- **典型操作**：「创建一个新的仪表盘路由，包含 /dashboard 页面、对应的 API handler、和单元测试」→ Composer 会一次性创建 3-4 个文件
- **重要技巧**：用 @ 符号引用相关文件——`@api.ts @types.ts @db.ts 给所有用户 API 加上分页参数`，这样 AI 参考了文件上下文再动手

**Agent（代理模式）**：在 Composer 中切换开关
- **用途**：需要执行终端命令的场景——安装 npm 包、运行测试、启动 dev server
- **典型操作**：「装一下 zod 并创建一个用户注册 schema，然后运行测试确认没问题」

**最佳实践**：在功能开发时，先 Chat 理解现有代码，再 Composer 写新功能。不要一上来就用 Composer 改不熟悉的代码。

### 真实项目流程举例

场景：为一个已有的聊天应用添加 Markdown 渲染支持。

```
1. Chat 模式：@Chat.tsx @messageService.ts "现在消息是怎么渲染的？有没有做富文本处理？"
2. Chat 模式看完代码结构后 → Composer 模式：
   "创建一个 react-markdown 封装的组件，放在 components/MarkdownRenderer.tsx
   然后在 Chat.tsx 的消息内容中调用这个组件
   确保代码安全性：过滤 XSS"
3. Agent 模式："react-markdown 装了吗？没装的话帮我装一下，然后跑一下测试"
4. 手动审查 Composer 生成的代码，确认改动合理
5. git commit
```

---

## Codex CLI 实战：终端中的 AI 对话式开发

### 安装和初始设置

1. 确保你有 Node.js 18+（`node --version` 查看）
2. 全局安装：`npm install -g @openai/codex`
3. 设置 API Key（二选一）：
   - 环境变量：`$env:OPENAI_API_KEY="sk-xxx"`（或者 `export` 在 Linux/Mac）
   - 配置文件：在项目目录下创建 `.env` 文件，写入 `OPENAI_API_KEY=sk-xxx`
4. 在项目目录中运行 `codex` 启动对话
5. 首次运行会提示你确认文件操作权限——推荐选「每次手动确认」，不要无脑全放权

### 基本操作流程

进入 Codex CLI 后，你面对的不是编辑器，是一个终端 Prompt。基本节奏如下：

```
$ codex
> // 项目: 一个简单的 CLI 笔记工具
> // 我想添加一个新功能: 支持 tag 过滤，用户可以用 --tag work 只显示某个 tag 的笔记
```

Codex CLI 会：
1. 读取当前项目的文件结构
2. 分析已有代码，理解你的架构
3. 生成修改内容，展示 diff
4. 询问你是否确认应用这个修改
5. 应用后询问是否需要继续

**关键命令**（在 Codex CLI 对话中输入）：
- `/reset`：清空当前上下文，开始新话题
- `/status`：查看当前会话状态
- `/undo`：撤销最后一次文件操作
- `Ctrl+C`：退出当前对话

### 真实项目流程举例

场景：用 Codex CLI 创建一个新的 Python 数据分析脚本。

```
// 先在项目根目录下运行 codex
// 第一轮对话：
$ codex
> // 我要创建一个分析脚本 analyze_logs.py
> // 它读取 logs/ 目录下所有 .log 文件
> // 提取每行 JSON 格式的日志中的 level 和 timestamp 字段
> // 输出每个日志级别的统计数量
> // 输出格式: Markdown 表格
```

Codex CLI 生成 `analyze_logs.py` → 你确认 → 让它跑一下 →

```
> // 运行一下看看有没有错误
```

Codex CLI 自动执行 `python analyze_logs.py` → 如果有报错，它自己看报错信息然后修复 → 直到能跑通。

```
> // 现在加一个新功能: 输出时间范围内的日志分布，按小时分组
```

继续迭代。每完成一个功能点，就结束对话，`git commit`，再开新对话。

---

## Cursor vs Codex CLI：什么时候用哪个

| 场景 | 首选 | 原因 |
|------|------|------|
| 每天开发的大项目（几千个文件） | Cursor | Codebase Index 让它真正理解你的项目 |
| 快速写个脚本/原型 | Codex CLI | 零启动成本，几句话就能跑起来 |
| 复杂的跨文件重构 | Cursor (Composer) | 多文件编辑体验更好，diff 查看更清楚 |
| 自动化测试+修复 | Codex CLI | 它能自主执行命令、读输出、修复，循环快 |
| 代码审查和架构讨论 | 两者都行 | Cursor 适合看代码聊，Codex CLI 适合让 AI 先分析再汇报 |
| 在不熟悉的语言里快速探索 | Codex CLI | 对话式试错更自然——写一句、跑一下、看结果 |
| 日常写 Markdown 文档/笔记 | Codex CLI | 不用离开终端，从写作到 git diff 到 commit 一条龙 |

---

## 要点总结

1. **Cursor 的核心优势是 Codebase Index**：它需要时间建立索引才能发挥威力，别刚打开项目就着急用
2. **Cursor 分清楚 Chat 和 Composer**：Chat 用于问答和单文件修改，Composer 用于跨文件重构和创建新功能
3. **Codex CLI 的核心优势是零启动和自动循环**：写 → 跑 → 看报错 → 改 → 再跑，这个循环是它的核心价值
4. **两个工具不冲突**：很多人同时使用——Cursor 做主 IDE，Codex CLI 做快速实验和小任务
5. **控制 AI 的作用范围**：用 @file（Cursor）或精确描述（Codex CLI）告诉 AI 只改哪些文件，避免 AI 擅自扩大修改范围
6. **Commit 习惯是基础**：无论用哪个工具，每完成一个可用状态就 commit——这是你面对 AI 修改的唯一「回滚按钮」
