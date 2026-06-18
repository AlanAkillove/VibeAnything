# 05-LaTeX 与学术写作工具

> 读完本文你将了解：LaTeX 为什么是计算机学术写作的标准、怎么用 Overleaf 零门槛开始写论文、核心 LaTeX 语法速查、以及如何用 AI 高效辅助 LaTeX 写作
> 注：本文面向零基础新手，从在线工具入手，无需安装几十 GB 的本地环境，30 分钟就能写出第一篇规范论文。

---

## 你可能遇到过的问题

你开始写实验报告、课程论文，甚至准备做科研写毕设，用 Word 写作时总会遇到这些崩溃时刻：

- **公式排版地狱**——复杂的数学公式对齐、编号、换行调半天，稍微改一下内容格式全乱
- **参考文献灾难**——手动维护引用列表，增减一篇文献就要手动重排所有序号和格式
- **格式反复返工**——写完发现不符合学校/会议的模板要求，通篇调格式比写内容还费时间
- **听说 LaTeX 排版好看，但觉得特别难**——看到一堆反斜杠命令就头大，怕学不会
- **AI 能生成 LaTeX 代码，但自己看不懂改不了**——报错了完全不知道怎么排查

这些问题的行业标准解决方案，就是 **LaTeX + Overleaf**，再配合 AI 辅助，新手也能快速写出专业级排版的文档。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| LaTeX 是一个写作软件 | LaTeX 是一套**排版系统**，你写带标记的纯文本内容，编译后自动生成规范的 PDF，核心是「内容与格式分离」 |
| 用 LaTeX 必须装几十 GB 本地环境 | 用 **Overleaf 在线编辑器** 就能直接写，浏览器打开即用，什么都不用装 |
| LaTeX 特别难学，要背很多命令 | 基础结构 30 分钟就能上手，常用语法就那几个，复杂效果查文档/问 AI 就行 |
| LaTeX 只能用来写论文 | 简历、幻灯片（Beamer）、实验报告、书籍、海报全都能做，CS 领域几乎全场景覆盖 |
| Word 排版更灵活自由 | 恰恰相反，LaTeX 用模板统一格式，你只需要专注内容；Word 反而要手动反复调格式，长文档效率极低 |
| 有 AI 就不用学 LaTeX 了 | AI 能帮你生成代码，但你需要能看懂、能修改、能排查报错，基础语法依然是必备的 |

---

## 常见问题快速解决

| 你遇到的问题 | 快速操作 |
|-------------|---------|
| 复杂公式不知道怎么写 | 直接描述给 AI，让它生成 LaTeX 代码；或者手写拍照让 AI 识别转换 |
| 参考文献格式总不对 | 用 BibTeX 管理文献，AI 可以帮你生成标准 `.bib` 条目，Overleaf 还能直接搜索引用 |
| 编译报错看不懂 | 把报错信息复制给 AI，90% 的常见错误 AI 都能定位并给出修复方案 |
| 不知道去哪找论文模板 | Overleaf 模板库有几千套官方模板，IEEE、ACM、各大高校模板直接套用 |
| 不想从零学语法 | 从现成模板改起，遇到不会的语法边查边用，不用一次性全学会 |

---

## 什么是 LaTeX？为什么 CS 学生必须学

LaTeX（常读作 /ˈlɑːtɛx/）是一种基于标记语言的排版系统，和 Word「所见即所得」的模式完全不同：你用纯文本写内容和标记，它自动帮你完成排版、编号、引用、目录等所有格式工作。

```
你编写的纯文本代码               编译后生成的规范PDF
───────────────────               ┌───────────────────────┐
\documentclass{article}           │                       │
\title{大模型性能对比研究}        │    大模型性能对比研究   │
\author{张三}                     │        张三           │
\begin{document}                  │                       │
\maketitle                        │  1 引言               │
\section{引言}                    │  随着大模型技术发展... │
随着大模型技术快速发展...          │                       │
\begin{equation}                  │       E = mc²  (1)    │
E = mc^2                          │                       │
\end{equation}                    └───────────────────────┘
\end{document}
```

### 计算机领域必学的核心理由
1. **学术圈通用标准**：CS 领域所有顶级会议（ACM、IEEE、NeurIPS 等）全部提供 LaTeX 官方模板，投稿必须用 LaTeX 格式
2. **公式排版无可替代**：复杂的数学公式、算法伪代码、矩阵、积分，在 LaTeX 里规范又美观，是 Word 永远比不了的
3. **参考文献全自动管理**：用 BibTeX 维护文献库，引用序号、格式自动生成，增减文献不用手动改序号
4. **完美适配版本控制**：所有 `.tex` 文件都是纯文本，可以直接用 Git 管理版本、多人协作，和你写代码的工作流完全打通
5. **AI 友好度拉满**：大模型能精准理解和生成 LaTeX 代码，是 AI 辅助写作效率最高的文档格式

---

## Overleaf：零门槛在线 LaTeX 编辑器

本地安装 LaTeX 环境（TeX Live / MiKTeX）体积大、配置麻烦，对新手非常不友好。**Overleaf** 是全球最主流的在线 LaTeX 平台，全球超过 2500 万用户，高校、科研机构广泛使用，浏览器打开就能用。

### Overleaf 的核心优势
- **零安装**：打开浏览器就能写，自动编译，不用配置任何环境
- **双编辑器模式**：支持可视化编辑器（不用写代码也能排版）和代码编辑器，可随时切换，新手友好度拉满
- **海量官方模板**：期刊论文、学位论文、简历、Beamer 幻灯片、实验报告模板应有尽有，直接套用不用从零搭格式
- **多人协作**：支持多人同时编辑、评论批注，适合组队做项目、合写论文
- **内置文献管理**：可直接搜索论文生成引用条目，支持 BibTeX 自动格式化
- **版本历史**：自带项目历史记录，可回溯任意版本；付费版支持完整版本控制

免费版即可使用无限项目，满足个人学习完全够用；高校学生很多可通过学校邮箱解锁专业版权益。

### 快速上手步骤
1. 打开 [overleaf.com](https://www.overleaf.com)，用邮箱注册账号
2. 点击「New Project」，可以选择空白项目，也可以直接从模板库选一个感兴趣的模板（比如简历、课程报告）
3. 把下面的示例代码粘贴到左侧编辑器，点击「Recompile」，右侧就能看到排版好的 PDF

```latex
\documentclass{article}
\usepackage[UTF8]{ctex}  % 中文支持宏包
\usepackage{amsmath}     % 数学公式增强宏包

\title{我的第一篇 LaTeX 论文}
\author{你的名字}
\date{\today}

\begin{document}
\maketitle

\section{引言}
这是一篇用 LaTeX 编写的文档。行内公式可以这样写：爱因斯坦质能方程 $E = mc^2$。

独立编号的公式如下：
\begin{equation}
\int_a^b f(x) \, dx = F(b) - F(a)
\end{equation}

\end{document}
```

---

## LaTeX 核心语法速查

不用一次性全背，先掌握最常用的部分，遇到复杂效果再查文档或问 AI。

### 1. 文档基本结构
```latex
\documentclass{article}        % 文档类型：article(短文) / report(报告) / beamer(幻灯片)
\usepackage{amsmath}           % 导入宏包，扩展功能
\usepackage[UTF8]{ctex}        % 中文支持

\title{文档标题}
\author{作者名}
\date{\today}

\begin{document}
\maketitle                    % 生成标题页
\tableofcontents              % 自动生成目录

\section{一级标题}
\subsection{二级标题}
\subsubsection{三级标题}

正文内容写在这里。

\end{document}
```

### 2. 数学公式
这是 LaTeX 最核心的能力，也是 AI 最能帮上忙的部分。

#### 三种公式写法
```latex
% 1. 行内公式：嵌入正文里
梯度下降的更新规则为 $\theta = \theta - \eta \nabla L$。

% 2. 独立无编号公式
\[
\sum_{i=1}^{n} i = \frac{n(n+1)}{2}
\]

% 3. 独立带编号公式
\begin{equation}
\mathcal{L} = -\frac{1}{N}\sum_{i=1}^{N} y_i \log \hat{y}_i
\end{equation}
```

#### 常用数学符号速览
不用死记，常用的用几次就熟了，完整符号表可查阅文末的参考链接：
- 希腊字母：`\alpha` α、`\beta` β、`\gamma` γ、`\delta` δ、`\theta` θ、`\lambda` λ、`\pi` π
- 运算符号：`\times` ×、`\div` ÷、`\pm` ±、`\cdot` ·、`\sum` ∑、`\prod` ∏、`\int` ∫
- 关系符号：`\leq` ≤、`\geq` ≥、`\neq` ≠、`\approx` ≈、`\in` ∈、`\subset` ⊂
- 逻辑符号：`\forall` ∀、`\exists` ∃、`\neg` ¬、`\land` ∧、`\lor` ∨、`\implies` ⟹

### 3. 图片与表格
```latex
% 插入图片（需要先把图片上传到 Overleaf 项目）
\usepackage{graphicx}
\begin{figure}[htbp]
    \centering
    \includegraphics[width=0.8\textwidth]{architecture.png}
    \caption{模型整体架构图}
    \label{fig:arch}
\end{figure}

% 插入表格
\begin{table}[htbp]
    \centering
    \caption{不同模型性能对比}
    \label{tab:model-compare}
    \begin{tabular}{|c|c|c|}
        \hline
        模型名称 & 准确率 & 参数量 \\
        \hline
        DeepSeek V4 & 92.5\% & 671B \\
        Qwen 3.7 & 91.8\% & 72B \\
        \hline
    \end{tabular}
\end{table}
```

### 4. 参考文献（BibTeX）
这是 LaTeX 最省心的功能之一，完全不用手动管理引用格式。

第一步：在项目里新建 `references.bib` 文件，存放文献条目：
```bibtex
@article{zhang2026llm,
    author  = {张三, 李四},
    title   = {大模型长上下文技术研究},
    journal = {计算机学报},
    year    = {2026},
    volume  = {49},
    pages   = {123-145}
}
```

第二步：在正文里引用：
```latex
如文献 \cite{zhang2026llm} 所述，长上下文是大模型的核心发展方向。

% 文末生成参考文献列表
\bibliographystyle{IEEEtran}  % 引用格式，对应会议/学校要求
\bibliography{references}
```

编译后会自动生成规范的参考文献列表，引用序号自动排序，增减文献只需修改 `.bib` 文件，不用手动调整。

---

## AI 辅助 LaTeX 写作的实用场景

AI 是 LaTeX 新手的最佳搭档，可以帮你解决 90% 的琐碎问题，让你专注内容而非格式。

| 场景 | 你可以这样问 AI |
|------|----------------|
| 生成公式代码 | 「用 LaTeX 写出交叉熵损失函数的公式，要求带编号」「把这个手写的公式转成 LaTeX 代码」 |
| 格式化参考文献 | 「把这篇论文信息转成 BibTeX 条目：作者XXX，标题XXX，期刊XXX，2026年」 |
| 排查编译错误 | 「我的 LaTeX 报错 Undefined control sequence，代码是……，帮我定位问题并修复」 |
| 生成表格/图片代码 | 「帮我生成一个 4 列的 LaTeX 表格，列名：模型、上下文长度、输入价格、输出价格」 |
| 适配模板 | 「我有一段论文内容，帮我改成 IEEE 会议 LaTeX 模板的格式」 |
| 代码美化整理 | 「这段 LaTeX 代码缩进很乱，帮我整理规范，加上注释」 |

> 💡 小技巧：不要让 AI 一次性写完整篇论文，建议分章节生成，自己保留结构控制权，AI 负责填充细节和格式转换。

---

## 课程论文/毕设推荐文件结构

对于篇幅较长的文档，不要把所有内容堆在一个文件里，按章节拆分更易维护，也方便用 Git 管理版本：

```
thesis/
├── main.tex              # 主文件，负责导入模板、章节、参考文献
├── references.bib        # BibTeX 文献库，所有文献统一存在这里
├── figures/              # 图片文件夹
│   ├── framework.png
│   └── experiment-result.png
├── chapters/             # 按章节拆分的内容
│   ├── 01-introduction.tex
│   ├── 02-related-work.tex
│   ├── 03-methodology.tex
│   ├── 04-experiments.tex
│   └── 05-conclusion.tex
└── template/             # 学校/会议提供的模板文件
    └── thesis.cls
```

主文件通过 `\include` 引入各章节：
```latex
\documentclass{article}
\usepackage{ctex, amsmath, graphicx}

\begin{document}
\include{chapters/01-introduction}
\include{chapters/02-related-work}
\include{chapters/03-methodology}
\include{chapters/04-experiments}
\include{chapters/05-conclusion}

\bibliographystyle{plain}
\bibliography{references}
\end{document}
```

因为所有文件都是纯文本，你可以直接把整个论文项目用 Git 管理，推到 GitHub 备份，和你管理代码项目的方式完全一致。

---

## 你现在应该做什么？

1. 打开 Overleaf 注册账号，创建一个空白项目
2. 把本文的示例代码粘贴进去，点击编译，看看 PDF 效果
3. 试着修改标题、正文，添加一个简单的公式，熟悉编译流程
4. 让 AI 生成一个你感兴趣的公式/表格，粘贴到文档里看效果
5. 去 Overleaf 模板库找一个简历/课程报告模板，试着改成自己的内容
6. 新建一个 `.bib` 文件，添加一篇参考文献，在正文里引用试试

---

## 更进一步

- [Overleaf 官方学习中心](https://www.overleaf.com/learn) — 从入门到精通的完整 LaTeX 教程，中文内容丰富
- [LaTeX 数学符号完整速查表](https://oeis.org/wiki/List_of_LaTeX_mathematical_symbols) — 涵盖所有常用数学符号的 LaTeX 写法

---

**要点总结**
- **LaTeX 是计算机学术写作的行业标准**，论文、报告、简历、幻灯片全场景适用，核心优势是内容与格式分离
- **新手直接用 Overleaf 在线编辑器**，零安装、双模式编辑、海量模板，不用折腾本地环境
- 不用死记所有语法，掌握**文档结构、公式、图表、参考文献**四大核心模块就足够入门，复杂效果查文档/问 AI
- **AI 是 LaTeX 写作的强力辅助**，公式生成、错误排查、文献格式化都能交给 AI，但基础语法依然要懂
- 长文档按章节拆分文件，用 BibTeX 管理参考文献，纯文本格式天然适配 Git 版本管理