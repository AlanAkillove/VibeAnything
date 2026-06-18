# 03-Git 与 GitHub 基础

> 读完本文你将了解：为什么在 AI 时代版本控制更重要、Git 最常用的几个命令、怎么把代码推到 GitHub、以及怎么用 Git 来管理 AI 生成的代码

---

## 你可能遇到过的问题

你用 AI 生成了很多代码——改了一版又一版。但：

- **AI 突然把整个文件改坏了**——你想回到上一版，但已经来不及了
- **你改了一堆东西，不知道怎么回到"一个小时前还能用"的状态**
- **你在实验室的电脑上写了一半，回宿舍想继续写**——只能 U 盘拷来拷去
- **你看到别人在 GitHub 上分享项目，想知道那些代码是怎么组织的**
- **你对 Git 的印象是"很难学的东西"**——光看到 checkout、rebase、merge 就头大

这些问题，80% 可以用三个 Git 命令解决。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| 一个人开发不需要 Git | **AI 时代一个人开发反而更需要 Git**——AI 改代码改得快，也改坏得快 |
| Git 很难学 | 日常开发 80% 的时间只用 `add`、`commit`、`push`、`pull`、`reset` 五个命令 |
| GitHub = Git | GitHub 只是一个"寄存 Git 仓库的云服务"，Git 本身是独立的版本控制系统 |
| 有 AI 帮我，不需要 commit 历史 | AI 没有版本意识——每次对话都是新的开始。**你的 commit 历史就是你的"消防通道"** |

---

## 遇到这些问题怎么办？

| 你遇到的问题 | 现在可以怎么做 |
|-------------|---------------|
| AI 改坏了代码 | `git restore 文件名` 恢复到上一个 commit 的版本 |
| 改了一堆但想全部放弃 | `git reset --hard HEAD` 回到最后一次 commit 的状态 |
| 想看看改了什么 | `git diff` 查看所有未暂存的改动 |
| 想保存当前进度但还没写完 | `git add .` + `git commit -m "草稿"` 先存起来 |
| 想把代码放到 GitHub | `git push` |

---

## 什么是 Git？为什么 AI 时代它更重要？

Git 是**版本控制系统**——它记录文件每一次变化的"快照"。任何时候你都可以回到任何一个历史版本。

在 AI 辅助开发的场景下，Git 变得更加重要：

- AI 能在 30 秒内生成 200 行代码——但也可能在下一轮把它改坏
- AI 没有"undo"——每次对话都是全新的上下文
- **你的 commit 历史，是你对 AI 修改的"撤销按钮"**

Vibe Coding 的最佳实践正是：每完成一个可验证的小步骤，就 commit 一次，然后清空 AI 上下文。

---

## 五个命令走天下

### 1. 初始化仓库

```bash
git init
```

在一个项目目录里执行一次，Git 就开始跟踪这个目录下的所有文件变更。

### 2. 查看状态

```bash
git status
```

**这是用得最多的命令。** 它会告诉你：哪些文件被改过、哪些还没提交、当前在哪个分支。

### 3. 暂存改动

```bash
git add 文件名   # 暂存指定文件
git add .        # 暂存所有改动
```

`add` 把改动放进"暂存区"——就像把要邮寄的东西装进包裹。

### 4. 提交快照

```bash
git commit -m "完成用户登录功能"
```

`commit` 把暂存区的改动保存为一个永久的版本记录。每条 commit 要写清楚改了什么的简短说明。

### 5. 推送到 GitHub

```bash
git push
```

把本地的 commit 记录上传到 GitHub。

---

## 一个完整的日常流程

你用 AI 生成了一段代码，测试通过后：

```bash
# 1. 看看改了哪些文件
git status

# 2. 把改动的文件加入暂存
git add .

# 3. 提交并写一条清晰的备注
git commit -m "feat: 用 AI 实现用户注册接口"
```

如果 AI 下一轮改坏了，你可以直接回到这个版本：

```bash
git reset --hard HEAD
```

或者只恢复某个文件：

```bash
git restore 文件名.py
```

---

## GitHub：把代码放到云端

### 怎么把本地仓库推到 GitHub

1. 在 [GitHub.com](https://github.com) 创建一个新仓库（不要勾选 README 和 .gitignore）
2. 在终端执行：

```bash
# 关联远程仓库
git remote add origin https://github.com/你的用户名/仓库名.git

# 推送到 GitHub
git push -u origin main
```

### 怎么从 GitHub 下载别人的项目

```bash
git clone https://github.com/用户名/仓库名.git
```

这会下载整个项目的代码和完整的 Git 历史。

---

## 必配：.gitignore

`.gitignore` 告诉 Git "这些文件不要跟踪"。你需要创建一个包含以下内容：

```
.venv/          # 虚拟环境（每个人环境不同）
__pycache__/    # Python 缓存
.env            # API Key（绝对不能上传）
*.pyc
.DS_Store       # macOS 系统文件
```

**特别注意 `.env` 文件**——里面存着你的 API Key。如果你忘了把它加到 `.gitignore` 里，推送到 GitHub 后你的 Key 就公开了。

---

## 和 AI 协作的 Git 工作流

Vibe Coding 的最佳实践配合 Git：

```
1. 打开一个新的 AI 会话
2. 告诉 AI 要实现什么功能
3. AI 生成代码
4. 测试功能是否正常
5. 测试通过 → git add + git commit
6. 清空 AI 上下文（/clear 或开新对话）
7. 回到步骤 1，开始下一个功能
```

这样每步都有记录，任何时候都可以回到任何一个可用的版本。

---

## 你现在应该做什么？

1. 安装 Git（[git-scm.com](https://git-scm.com)）
2. 创建一个 GitHub 账号
3. 把一个已有的项目目录做 `git init`
4. 做一次 `add` + `commit`
5. 创建一个 GitHub 仓库，把本地的项目推上去
6. 再做一次修改，再 `add` + `commit` + `push`——感受一下循环

---

## 更进一步

- [Git 官方文档](https://git-scm.com/doc) — 入门到精通
- [GitHub Hello World](https://docs.github.com/en/get-started/start-your-journey/hello-world) — GitHub 官方 10 分钟教程

---

**要点总结**
- **AI 时代 Git 更重要**——它是你应对 AI 改坏代码的"撤销按钮"
- 日常 80% 的操作只需 **`status`、`add`、`commit`、`push`、`reset`** 五个命令
- **`.gitignore` 一定要配**——特别是 `.env`（API Key）
- **每完成一个小步骤就 commit 一次**——这是 Vibe Coding 的最佳实践
- 遇到 Git 问题，可以直接问 AI：`"我不小心 commit 了不该 commit 的文件，怎么撤销？"`
