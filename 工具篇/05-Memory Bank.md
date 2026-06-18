# 05-Memory Bank

> 读完本文你将了解：什么是 Memory Bank、为什么 AI 辅助开发需要它、怎么为一个项目搭建 Memory Bank 结构、以及如何在跨 session 开发中保持 AI 的上下文连续
> 注：本文聚焦「为项目建立持久上下文」，适合单人/小团队的 AI 辅助开发场景。Memory Bank 是实践方案，不是标准规范——你可以根据自己的项目调整。

---

## 你可能遇到过的问题

你在用 AI 工具开发一个项目，几个月后遇到了这些状况：

- **新开一个 AI 对话，它完全不记得之前讨论过的架构决策**——你又要从零解释一遍项目背景
- **AI 给你出了三个方案，和三个月前否决的那个方案一模一样**——它不知道之前为什么否决，你又得解释一遍
- **同一个项目换了一个工具来写，新工具不知道项目约定**——命名规范、组件结构、路由规则全凭自己猜
- **项目做到一半放了两周，回来继续写时连自己都忘了当时的思路**——AI 更没有记忆，你和 AI 要一起重新理解项目
- **多个 AI 工具切换时，每个工具都需要重复告知项目上下文**

这些问题都指向同一个根源：**AI 工具没有持久上下文。每一轮新对话、每一次工具切换，它都是从零开始理解你的项目。你的项目背景、决策记录、约定规则，需要你主动帮它记住。**

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| AI 会自动记住我的项目 | AI 的「记忆」只存在于当前对话中。一旦关闭对话或切换工具，所有上下文清零 |
| 代码就是最好的文档，看代码就知道了 | 代码只能告诉 AI「现状是什么」——不能告诉 AI「为什么这么设计」、「之前否决了什么方案」、「未来计划做什么」 |
| 写文档是浪费时间，不如多写代码 | Memory Bank 不是传统意义上的文档——它是「给 AI 读的上下文快照」，写好了能让 AI 输出质量提升 50% 以上 |
| 一个项目一个 README 就够了 | README 面向人类开发者，Memory Bank 面向 AI——两者内容不同、结构不同、详略不同 |
| 跨 session 上下文管理是团队才需要的 | 单人开发反而更需要——因为没有人替你交接上下文，AI 就是你唯一的「文档型队友」|

---

## 常见问题快速解决

| 你遇到的问题 | 现在可以怎么做 |
|-------------|---------------|
| 新对话 AI 不记得项目结构 | 创建一个 `memory-bank/project.md` 文件，里面写项目的技术栈、目录结构、核心架构决策 |
| AI 重复已经被否决的方案 | 在 `memory-bank/decisions.md` 中记录「已否决的方案和原因」——新对话的 AI 能读到 |
| 多个项目之间切换，上下文混淆 | 每个项目独立存放 Memory Bank，AI 通过当前目录加载对应项目的上下文 |
| 放了两周的项目回来继续做 | 先让 AI 读 memory-bank/，然后再开始写代码 |
| 换工具时上下文丢失 | Memory Bank 是纯文本 Markdown——所有工具都能读，是跨工具共享上下文的唯一标准格式 |

---

## 什么是 Memory Bank

Memory Bank 是一组 Markdown 文件，放在项目根目录的 `memory-bank/` 文件夹下。它的作用是**为 AI 提供项目级别的持久上下文**。

每次你开新 AI 对话、切换 AI 工具、或者从休假回来继续项目时，只要让 AI 先读 `memory-bank/`，它就立刻知道你当前项目的全貌——不需要你从零解释。

一个完整的 Memory Bank 通常包含以下文件：

```
memory-bank/
├── project.md         # 项目概览：技术栈、目录结构、核心目标
├── architecture.md    # 架构设计：分层、数据流、关键模式
├── decisions.md       # 决策记录：技术选型、方案变更、否决原因
├── techstack.md       # 技术栈详情：依赖、版本、配置说明
├── progress.md        # 进度追踪：完成了什么、正在进行什么、待办事项
└── .gitkeep
```

---

## 各文件详解

### project.md — 项目概览

这是 AI 第一个读的文件。读完它，AI 应该能回答「这是什么项目？用什么技术？目录大概怎么组织的？」

```markdown
# 项目: AI 笔记助手

## 项目目标
一个支持 Markdown 编辑和 AI 自动摘要的个人笔记应用。

## 技术栈
- 前端: Next.js 14 (App Router) + Tailwind CSS
- 后端: Next.js API Routes
- 数据库: SQLite (better-sqlite3)
- AI: OpenAI API (text-embedding-3-small + gpt-4o-mini)
- 部署: Vercel

## 核心目录结构
```
src/
├── app/          # Next.js App Router 页面
│   ├── layout.tsx
│   ├── page.tsx          # 笔记列表页
│   └── notes/
│       ├── [id]/page.tsx # 单篇笔记详情
│       └── new/page.tsx  # 新建笔记
├── components/   # 可复用组件
├── lib/          # 工具函数和 AI 相关逻辑
└── db/           # 数据库操作
```

## 核心约定
- 全部使用 TypeScript
- 使用 named export，不使用 default export
- 数据库操作统一在 db/ 目录下
- AI 调用统一在 lib/ai.ts 中封装
```

### architecture.md — 架构设计

描述项目的架构思路、分层策略、数据流。AI 需要这个来理解「你的项目怎么组织的」和「加到哪个位置」。

```markdown
# 架构设计

## 整体架构
Next.js App Router + API Routes。前端和服务端在同一个项目中。

## 数据流
用户创建笔记 → API Route 接收 → validate 输入 → 写入 SQLite → 异步调用 AI 生成摘要 → 更新笔记记录

## 关键设计决策
- 使用 SQLite 而非 Postgres：原因是单人使用、数据量小、部署简单（Vercel 需用 Vercel Postgres 替代）
- AI 调用使用 Queue 模式：创建笔记后立即返回，AI 处理异步进行，避免用户等待
- Markdown 渲染使用 react-markdown + rehype-sanitize：支持几乎全部 Markdown 语法，并做 XSS 防护

## 组件树
```
Layout (Header + Sidebar)
├── NoteList          # 左侧笔记列表
│   └── NoteCard      # 单篇笔记卡片
├── NoteEditor        # 编辑器区域
│   ├── MarkdownInput
│   └── Preview
└── AISidePanel       # AI 建议面板
    ├── SummaryCard
    └── TagSuggestions
```
```

### decisions.md — 决策记录

记录「为什么这么做」和「为什么不那么做」。这是 AI 最容易遗漏的信息维度——它能从代码看到现状，但看不到决策过程。

```markdown
# 决策记录

## 2026-06-01: 数据库选型 SQLite 而非 Postgres
**背景**：单人笔记应用，部署在 Vercel，数据量预估很小
**决策**：使用 SQLite（better-sqlite3），配合读写分离
**理由**：
- 零运维，不用配置数据库服务
- Vercel Serverless 函数场景下，SQLite 可以通过读写分离方案支持（写入时用单独 process）
- 如果后续用户量增长，可以平滑迁移到 Turso 或 Vercel Postgres
**否决方案**：Postgres — 对单人项目来说过度设计，运维成本与收益不匹配

## 2026-06-10: AI 摘要采用 Queue 模式
**背景**：创建笔记后需要 AI 生成摘要，直接调用需要用户等 2-3 秒
**决策**：创建笔记 API 立即返回，AI 摘要通过异步任务完成
**理由**：用户体验优先——用 2 秒等待换「用户感觉不到延迟」
**否决方案**：SSE 流式输出摘要 — 增加复杂性，对 MVP 阶段意义不大
```

### techstack.md — 技术栈详情

不是重复 project.md 中的清单，而是详细到版本号、配置说明、特殊用法。

```markdown
# 技术栈详情

## Next.js 14.2.5
- App Router
- Server Components 默认，客户端组件用 'use client'
- 中间件用于简单鉴权

## Tailwind CSS 3.4
- 自定义颜色在 tailwind.config.ts 中定义
- 深色模式用 class 策略

## better-sqlite3 11.0
- 数据库文件在 data/notes.db
- 所有查询都用参数化方式，禁止字符串拼接
- 使用 WAL 模式提升并发读性能

## OpenAI API
- embedding: text-embedding-3-small
- chat: gpt-4o-mini (摘要功能)
- 所有 AI 调用在 lib/ai.ts 中，外部不直接引用 openai SDK
```

### progress.md — 进度追踪

让 AI 知道「现在做到哪了」和「接下来做什么」。

```markdown
# 进度追踪

## 已完成
- [x] 项目初始化，Next.js + Tailwind 配置
- [x] 数据库 Schema 设计和实现（笔记表 + 标签表）
- [x] 笔记 CRUD API（GET/POST/PUT/DELETE）
- [x] 基础编辑器和 Markdown 预览组件
- [x] AI 摘要功能

## 进行中
- [ ] AI 标签推荐功能——API 已规划，前端 UI 待开发

## 待办
- [ ] 全文搜索（FTS5）
- [ ] 深色模式
- [ ] 移动端适配
- [ ] 部署到 Vercel

## 已知问题
- AI 摘要偶尔会截断长笔记的内容——需要在 prompt 中加 token 限制
- SQLite WAL 文件在 dev 模式下不会自动清理——加个定期清理脚本
```

---

## Memory Bank 的维护节奏

Memory Bank 的价值在于**持续更新**。不需要每天都写，但要跟着项目节奏更新：

| 事件 | 需要更新的文件 |
|------|--------------|
| 新增一个依赖 | techstack.md |
| 做了一个架构决策 | decisions.md |
| 完成了一个功能 | progress.md |
| 修改了目录结构 | project.md |
| 新增了设计模式 | architecture.md |
| 准备开始新阶段 | 读一遍所有文件，确保和代码一致 |

**重要**：每次和 AI 开始新对话时，第一句话让 AI 读 Memory Bank。

```
// 在 Cursor Chat 中：
"先读 memory-bank/ 了解项目全貌，然后继续实现标签推荐功能"

// 在 Codex CLI 中：
// 先读 memory-bank/ 了解项目全貌
// 然后继续实现标签推荐功能
```

---

## 进阶：Auto-update 模式

当你熟悉基本用法后，可以尝试让 AI 维护 Memory Bank 本身：

```
每次 commit 前，让 AI：
1. 检查代码变更
2. 如果有架构变化或新依赖，更新对应的 memory-bank 文件
3. 更新 progress.md 到最新状态
```

这样 Memory Bank 的维护就不再是你的负担了。

---

## 要点总结

1. **Memory Bank 不是文档**：它是给 AI 读的上下文快照——让 AI 从零理解你的项目只需要读 5 个 Markdown 文件
2. **最核心的文件是 project.md 和 decisions.md**：一个告诉 AI 项目是什么，一个告诉 AI 为什么这么做
3. **决策记录的价值被严重低估**：代码只能告诉 AI 现状，决策记录告诉 AI 过程——这直接影响 AI 的输出质量
4. **更新节奏跟着项目事件走**：加了依赖、做了决策、完成了功能——顺手更新对应文件，不需要额外时间
5. **跨 session 和跨工具的解决方案**：纯 Markdown，所有工具都能读——这是目前唯一通用的跨工具上下文载体
6. **让 AI 自己维护 Memory Bank**：熟练后，让 AI 在每次 commit 前自动更新相关文件，把维护负担降到最低
