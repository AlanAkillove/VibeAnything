# 05-LaTeX 与学术写作工具

> 读完本文你将了解：LaTeX 是什么、为什么 CS 学生必须学 LaTeX、怎么用 Overleaf 在线写论文、以及怎么用 AI 辅助 LaTeX 写作

---

## 你可能遇到过的问题

你开始写实验报告、课程论文、或者毕业设计。用 Word 写的时候遇到了这些情况：

- **公式排版噩梦**——复杂的数学公式在 Word 里怎么也对不齐，换行、编号搞得想砸电脑
- **参考文献管理灾难**——手动维护参考文献列表，引用顺序一变就要全部手动改
- **格式打架**——写完之后发现格式和老师要求的不一样，不得不从头改
- **你看到学长学姐的论文排版特别漂亮**——问了一下，他们说"用 LaTeX 写的"
- **你试着学了一下 LaTeX**——被各种 `\` 开头的命令搞晕了

这些问题有一个共同的解决方案：**LaTeX**。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| LaTeX 是一个软件 | LaTeX 是一个**排版系统**——你写的是纯文本代码，编译后才生成 PDF |
| LaTeX 很难学 | LaTeX 的基础结构 **30 分钟就能上手**——之后遇到不会的查一下就行 |
| LaTeX 只适合写论文 | LaTeX 也适合写简历、做幻灯片（Beamer）、写书籍、做海报 |
| Word 排版更灵活 | 恰恰相反——**LaTeX 自动处理排版**，你只需要关注内容，格式由模板决定 |
| 有了 AI 就不用学 LaTeX 了 | AI 能帮你**写 LaTeX 代码**，但你需要能读懂和修改它 |

---

## 遇到这些问题怎么办？

| 你遇到的问题 | 现在可以怎么做 |
|-------------|---------------|
| 公式排不好 | 用 AI 生成 LaTeX 公式代码，直接复制到文档里 |
| 参考文献不知道怎么管 | 用 BibTeX（`.bib` 文件），AI 可以帮你格式化参考文献 |
| LaTeX 编译报错看不懂 | 把错误信息复制给 AI，让 AI 解释和修复 |
| 不知道论文模板在哪找 | Overleaf 上有大量模板（论文、简历、Beamer 幻灯片） |
| 不想从零学 LaTeX | 从 Overleaf 模板开始修改，遇到不会的语法问 AI |

---

## 什么是 LaTeX？

LaTeX（读作 "lah-tech" 或 "lay-tech"）是一种**基于标记语言的排版系统**。它不是像 Word 那样的"所见即所得"编辑器——你写的是带标记的纯文本，然后编译成漂亮的 PDF。

```
你写的是：                    编译出来的是：
\documentclass{article}        ┌────────────────────────┐
\title{我的论文}               │                        │
\author{张三}                  │    我的论文             │
\begin{document}               │    张三                │
\maketitle                     │                        │
这是一篇关于 AI 的论文。       │ 这是一篇关于 AI 的论文。│
\end{document}                 └────────────────────────┘
```

### 为什么 CS 学生必须学 LaTeX

- **CS 论文全部用 LaTeX 写**——ACM、IEEE 的模板都是 LaTeX 格式
- **公式排版无可替代**——复杂数学公式在 LaTeX 里清晰、规范
- **参考文献自动管理**——用 BibTeX 管理引用，格式自动生成
- **版本控制友好**——`.tex` 是纯文本，可以用 Git 管理
- **AI 友好**——AI 能理解和生成 LaTeX 代码

---

## Overleaf：在线 LaTeX 编辑器

安装 LaTeX 本地环境（TeX Live / MiKTeX）有几十 GB，对新手不友好。**Overleaf** 是在线 LaTeX 编辑器，不需要安装任何东西。

### 快速开始

1. 打开 [overleaf.com](https://www.overleaf.com)，注册免费账号
2. 点击 "New Project" → "Blank Project"
3. 粘贴以下代码：

```latex
\documentclass{article}
\usepackage[UTF8]{ctex}  % 中文支持

\title{我的第一篇 LaTeX 论文}
\author{你的名字}

\begin{document}
\maketitle

\section{引言}
这是一篇用 LaTeX 写的论文。公式可以这样写：

\begin{equation}
E = mc^2
\end{equation}

引用文献用 \cite{example} 实现。

\begin{thebibliography}{9}
\bibitem{example} 作者, 《论文标题》, 期刊名, 2024.
\end{thebibliography}

\end{document}
```

4. 点击 "Recompile"，就能看到排好版的 PDF

---

## LaTeX 基础语法

### 文档结构

```latex
\documentclass{article}        % 文档类型：article, report, book
\usepackage{amsmath}           % 数学宏包
\title{标题}
\author{作者}

\begin{document}
\maketitle                    % 生成标题

\section{一级标题}
\subsection{二级标题}
\subsubsection{三级标题}

正文内容写在 section 之间。

\end{document}
```

### 数学公式

```latex
% 行内公式
爱因斯坦提出 $E = mc^2$。

% 独立公式（带编号）
\begin{equation}
    \int_a^b f(x) \, dx = F(b) - F(a)
\end{equation}

% 不带编号的独立公式
\[
    \sum_{i=1}^{n} i = \frac{n(n+1)}{2}
\]

% 矩阵
\[
\begin{bmatrix}
    1 & 2 \\
    3 & 4
\end{bmatrix}
```

### 图片和表格

```latex
% 插入图片
\usepackage{graphicx}

\begin{figure}[h]
    \centering
    \includegraphics[width=0.8\textwidth]{图片文件名.png}
    \caption{图片标题}
    \label{fig:example}
\end{figure}

% 表格
\begin{table}[h]
    \centering
    \begin{tabular}{|c|c|c|}
        \hline
        模型 & 准确率 & 参数量 \\
        \hline
        GPT-3 & 85\% & 175B \\
        GPT-4 & 92\% & 未公开 \\
        \hline
    \end{tabular}
    \caption{模型对比}
    \label{tab:comparison}
\end{table}
```

### 引用

```latex
% 在文档中引用
如文献 \cite{author2024} 所述……

% 参考文献文件（.bib）
@article{author2024,
    author  = {作者},
    title   = {论文标题},
    journal = {期刊名},
    year    = {2024}
}
```

---

## 用 AI 辅助 LaTeX 写作

AI 在 LaTeX 写作中特别有用：

| 场景 | 可以问 AI |
|------|-----------|
| 生成公式代码 | "用 LaTeX 写出这个公式：F 等于 G 乘以 m1m2 除以 r 的平方" |
| 格式化参考文献 | "把这篇论文格式化成 BibTeX 条目：……" |
| 修复编译错误 | "我的 LaTeX 报错 'Undefined control sequence'，代码是这样的：……" |
| 生成表格代码 | "用 LaTeX 生成一个三列的表格，标题是：模型、参数、年份" |
| 代码优化 | "这段 LaTeX 代码太乱了，帮我整理一下缩进" |

---

## 一篇 CS 论文的文件结构

对于一个课程论文或毕业设计，推荐的文件组织方式：

```
project/
├── main.tex              # 主文件，\include{chapters/…}
├── references.bib        # BibTeX 参考文献库
├── figures/              # 图片文件夹
│   ├── architecture.png
│   └── results.png
├── chapters/             # 各章节
│   ├── 01-introduction.tex
│   ├── 02-methodology.tex
│   ├── 03-experiments.tex
│   └── 04-conclusion.tex
└── template/             # 模板文件
    └── ieee-template.tex
```

主文件中用 `\include{}` 引入各章节：

```latex
\documentclass{article}
\begin{document}
\include{chapters/01-introduction}
\include{chapters/02-methodology}
\include{chapters/03-experiments}
\include{chapters/04-conclusion}
\bibliography{references}
\end{document}
```

---

## 你现在应该做什么？

1. 打开 Overleaf，注册账号
2. 创建一个空白项目，粘贴上面的示例代码
3. 编译并查看 PDF 效果
4. 试着改一些内容：标题、作者、公式
5. 把 AI 生成的 LaTeX 公式代码粘贴到文档里
6. 去 Overleaf 的模板库找一个你感兴趣的模板（比如简历模板）

---

## 更进一步

- [Overleaf 文档](https://www.overleaf.com/learn) — 官方 LaTeX 教程
- [LaTeX 数学符号速查表](https://oeis.org/wiki/List_of_LaTeX_mathematical_symbols) — 常用数学符号

---

**要点总结**
- **LaTeX 是 CS 学术写作的标准工具**——论文、简历、幻灯片都用它
- **Overleaf 是在线 LaTeX 编辑器**——不用安装任何东西就能开始写
- LaTeX 不是"所见即所得"，而是**写代码 → 编译 → PDF** 的流程
- AI 可以帮你生成 LaTeX 公式、格式化参考文献、修复编译错误
- 推荐文件结构：`main.tex` 用 `\include{}` 引入各章节，参考文献用 `.bib` 文件管理
