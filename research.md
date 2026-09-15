# research.md — 结构化编辑 × AI 落地调研

## 引言

让 AI 修改代码，目前主流的几条路径各有其固有缺陷：

- **文本 diff / patch**：模型输出 SEARCH/REPLACE 或 unified diff，以上下文文本定位。定位与代码语义无关；当上下文不唯一或模型记忆出现偏差时，补丁会误配或直接应用失败。
- **直接让 LLM 输出代码**：由模型整份重写文件或代码块。仅对**极小的 snippet** 可行；内容一大，成本与出错率都不可接受，且容易漏改、误改无关部分。
- **让模型用 grep / sed / awk 改代码**：把编辑降级为 shell 文本操作。看似灵活，实则隐患明显：正则与转义脆弱，跨行结构（括号、缩进）难以正确处理，模型「读到的」与「改成的」容易不一致，且失败常常是静默的。

三条路径的共同点：**编辑发生在文本层，而代码的含义在结构层**。本笔记分两部分：**第一部分是本文方案**（ast-edit × text-edit × LLM 的混合编辑方案），**第二部分是相关工作**（他人已落地的产品/工具/API 与评测）。论文向内容另见 `related-works.md`，项目参考与灵感来源见本文件第 5 节。

本文主张：**结构化编辑承担正确性与可靠性，direct 承担 token 效率**；并证明该混合方案的**端到端成功成本**低于 udiff 与纯 ast-edit——因为失败更少，重试更少。

一手来源访问日期：2026-09-13（Aider、ast-grep 页面于本次复核时重新抓取）。

---

# 一、本文方案：ast-edit × text-edit × LLM

## 1. 混合编辑方案：ast-edit × text-edit

综合本次调研，我们提出一个混合方案：**宏观用结构化编辑，微观用 direct（由 LLM 直接输出 snippet）**。

### 1.1 动机

混合方案的两极，对应能力分布的两种错位：**微观顺应 AI 的优势区间，宏观补文本工具的短板**。

- **微观顺应 AI 优势**：LLM 以自回归直接输出文本/代码为训练目标，**「直接输出代码（text）」本就是它的优势区间**。在微观，`direct`（整块输出 snippet）应当被顺应，而非对抗。
- **宏观补文本工具短板**：由 `toplevel / mod / impl / fn` 构成的**层级嵌套**（以及文件内多语言的语言嵌套，见 §1.2），恰恰是现有 text edit（udiff / grep / sed）与 view 工具**效果最差**的一环——这正是结构化编辑该补位的地方（见 §1.4）。

文本层编辑（udiff、grep/sed/awk）的主要失效不是括号本身，而是**定位漂移**：补丁上下文与文件不匹配，`patch` 拒绝应用或误配，且失败常常是静默的；括号失配只是深嵌套下的一个高发子类。mri-project 的评测显示：`direct` 只在小 snippet 上 token 开销小、准确率高，规模一大，便难以扩展——真正的原因是**上下文窗口**：直接写代码要求模型同时把「要改的代码」与「推演过程」都放进上下文，而思考本身也吃 context；codebase 一大这份上下文就放不下，实际可行的几乎只剩 sed / udiff 这类只需局部上下文的文本手段。也就是说，udiff / sed 在大 codebase 上的统治地位并非因为可靠，而是被上下文约束逼出来的——这恰恰放大了它们高失败率的代价。`ast-edit` 准确率显著提升，但 token 开销大（主要来自 agent 试错，且当前仅为基础实现）。flatten 的适用面不止小 snippet：真实 codebase 里存在一类密集、样板、或结构表达力难以覆盖的区域，human 与 AI 同样写不清楚——这类「结构编辑也讲不明白」的区域正是 `direct` 的靶心。

本方案的**落地目标是 Rust**（括号深嵌套），**Racket 仅作为验证 infra**。方案默认**结构化编辑的词汇表表达力足够**；该表达力（能否用 primitive 到达期望程序）跟踪于 `mri-project`，属外部依赖。

### 1.2 方案

- **flatten 内核**：算法核心尽量摊平所有可摊平的部分，交给 `direct` 整块输出——即**顺应 LLM 最擅长的优势区间**。判据不是「足够小」，而是「结构表达力覆盖不到、human 与 AI 同样写不清楚」：很多 codebase 里真实存在的密集/样板段落属于此类。
- **flatten 判定与 fallback**：内核判定哪些区域可摊平，交给 `direct`；无法摊平的区域，退回结构化编辑。回退不以单次操作成本论优劣：在 flatten 失败的深层/嵌入区域，diff 的成功率同样低，结构化编辑 + viewer 反而收窄了失败面（见 §1.3）。
- **code action 已有 / 易实现**：flatten 所需的 code action 目前已具备；以下 AST 变换也容易补上——extract variable / function、**extract module / split file**、flatten mod info files（Rust）。
- **结构化编辑负责两类「嵌套」**，以提高补丁成功率：
  - **语法嵌套**：由 `toplevel / mod / impl / fn` 形成的层级。
  - **语言嵌套（嵌入语言与文件内多语言）**：Rust 的 macro 与 DSL、以及 HTML / JS / CSS，常与宿主代码共处**同一个文件**。「拆成细碎小文件」是业界既有做法，也是当前 agent 能力边界下的**安全基线**——本方案不与它对抗，而是顺着纹理走：把 extract module / split file 补成结构化 code action，让拆文件由工具完成、代价（跨文件跳转、更多 IR token、review 变难）压到最低。在此基线上方才做增量：结构化编辑**识别并隔离同一文件内的子语言区域**，让每段只在自身范围内被处理，从而**约束 LLM 的处理范围**、避免跨区域干扰。这是期权而非赌注——拆文件始终是可回落的**下行界**。
- **large pattern match / clause**：同样宏观用结构编辑按 clause 组织，微观交给 `direct`。
- **目标**：在省 token、提成功率的前提下，完整替代 udiff / grep / sed / awk 这些高失败率的文本手段。

### 1.3 成本模型与主张

**主口径：端到端成功成本**——从开始到改对为止、含失败与重试的总 token，对照 udiff 与纯 ast-edit。

- **机理**：失败更少 → 重试更少 → 端到端成本更低。
- **回退区域占优**：单次操作上结构编辑更贵，但在 flatten 失败的深层/嵌入区域，diff 同样低成功率；结构化 viewer + scope 收窄降低定位与括号错误，省下重试，端到端占优。这正是本主张要测的部分。
- **失败分级**：A 语法/编译失败、B 语义不等价、C 需人工介入；A、B 触发重试（计入成本），C 计为人工成本。

### 1.4 附带收益：结构化 viewer 与 symbol query

- **结构化 viewer**：可视化嵌套层级与括号配对，既便于查看、也便于修改，并**缩小 scope**、规避各种奇怪 case——这是文本工具（grep / diff）最弱的一环。
- **symbol query**：可获取 toplevel 信息，便于 LLM 查询各类 symbol——**无需完整 semantics，成本更低**；同时也利于组织文档。

### 1.5 实现分析

- **relative node id（tree path）**：位置偶有偏移，对 LLM 推理不友好；但天然适合并发。
- **absolute node id**：对 LLM 推理友好；但需要中心化的 uid 分配器，UUID 生成方式不够鲁棒。
- **并行策略**：并行时划分 relative prefix，单个 agent 内部使用 absolute id——引用与分配都很直接，也不会引入中心锁。
- **节点引用（默认方案）**：对 LLM 暴露**层级语法路径**（如 `mod[0]/impl[1]/fn[2]`），内部映射到稳定 ID；并发以前缀分区，并对重叠写做冲突检测。
- **reactive programming（后续阶段）**：某个 node 变更后，按相关 edge 触发 review/refactor；edge 通过染色连接成 DAG。借助 `pub / priv / protected / internal` 让 DAG 更接近一棵 tree（逻辑更简单）；回边问题交由 structure-editor 处理。此部分风险最高，暂不纳入首期闭环。

### 1.6 future work

1. 增加更多 semantics 查询。
2. 让 LLM 自行编写微观的 code action（VSCode / LSP），再调用这些工具执行重构——即由 LLM 生成 code action tooling。
3. 提供常用 code action：把高频、好用的优先暴露给 LLM 使用。

### 1.7 可交给实习生的任务

mri-project 目前仅在一个小测试集合上打通并优化整套 infra，下一步需要**更大的 eval / benchmark**。其中的优化方向目标清晰、彼此独立，适合交接给实习生：

- **扩大测试集合**：从小集合扩展到更大、更真实的语料（当前为 Racket / LeetCode，后续扩展到真实 Racket 程序）。
- **优化 prompt**：针对 `ast-edit` 的试错开销做 prompt 调优。
- **提供更多 primitive**：把更多编辑动作补成 agent 可直接调用的原语，减少自由发挥。
- **提高 editor 完成度**：先在小测试集合上打磨整套 infra，再扩展到真实程序。

### 1.8 验收标准（MVP）

eval / benchmark 即 MVP 的验收载体，不是一次性测试，因此按 production 定标准：

**核心 kill test**：单文件函数体的三臂对照——udiff / 纯 ast-edit / 混合，在真实、深嵌套、多文件 Rust 上量端到端成功成本；若不低于 `tuned-diff + verify + retry`，后面全都不用做。对照基线须**包含拆文件后的工作流**（`tuned-diff + 拆文件 + verify + retry`）：拆文件是既有习惯，混合方案必须赢过的是它，而非「不拆文件」——否则赢的是稻草人。

- **三臂对照**：udiff / 纯 ast-edit / 混合方案。
- **任务集**：真实、深嵌套、多文件 Rust，并保留 held-out，防止在小集合上过拟合。
- **泛化**：验收覆盖未见过的真实仓库，而非仅通过精选 bench；「bench 通过」不等于「可用」。
- **指标**：成功率、语义等价、端到端 token、编辑次数；失败按区域类型分列**定位错误率**与**括号错误率**，以及 A/B/C 失败率。
- **可恢复性**：失败必须可回退、可重试，不产生不可逆的坏状态。

---

# 二、相关工作

## 2. Aider：LLM 编辑格式的设计与实证

Aider（https://github.com/Aider-AI/aider）把「LLM 如何输出一次代码编辑」当作可设计、可量化、可消融的接口：为不同模型选择不同编辑格式（`whole` / `diff` / `diff-fenced` / `udiff` / `editor-diff`），并可用 `--edit-format` 强制指定。

**结论**：

- **编辑格式是一等设计对象，且随模型而变**，不能只支持一种。四条原则：FAMILIAR（选模型见过的格式，如 unified diff）、SIMPLE（避免转义与行号等**脆弱定位符**）、HIGH LEVEL（整块替换函数/方法，而非逐行最小改动）、FLEXIBLE（解释编辑指令时尽量宽容）。
- **unified diff 让 GPT-4 Turbo 在「偷懒」基准上从 20% 提升到 61%**（`gpt-4-1106-preview`，89 个 Python 重构任务）。
- **行号对 LLM 极不友好**；Aider 让模型省略 hunk 行号，把每个 hunk 当作内容寻址的搜索/替换。
- **高层 diff 提示不可或缺**：不做时编辑错误增加 30–50%。
- **灵活打补丁不可或缺**：关闭灵活匹配后，Exercism 基准上编辑错误增加 9 倍（规范化 hunk、找回漏标增行、相对缩进匹配、大块拆分、逐级放宽）。
- **不要把源码塞进 JSON / function calling**：转义易错，OpenAI structured format 反而更差。
- 关键判断：「any naive or direct use of structured diff formats is pretty much doomed to failure」——结构化补丁直接照搬会失败，须配合高层化 + 灵活应用。
- 结论来自 2023 年 GPT-4 世代，但「格式—模型」耦合、行号脆弱、灵活匹配三点至今仍有参考价值。

来源：
- Edit formats — https://aider.chat/docs/more/edit-formats.html
- Unified diffs make GPT-4 Turbo 3X less lazy（2023-12-21, Paul Gauthier）— https://aider.chat/2023/12/21/unified-diffs.html
- 重构基准源码 — https://github.com/Aider-AI/refactor-benchmark

---

## 3. 结构感知工具 × AI 智能体

这一组的共性：**底层是结构感知能力，上层通过 skill / MCP / prompt 交给 AI 使用**。它们是「结构化编辑」从「人能用的 CLI」走向「AI 能用的接口」的现成样本。

### 3.1 搜索 / 变换工具

- **ast-grep（重点）**——基于 tree-sitter AST 的多语言模式搜索/重写，核心是「用和代码同构的模式去搜索代码」，与文本 grep/ripgrep 相对；可做「没有错误处理的 async 函数」「参数多于 3 个的函数」「类方法里的 console.log」等**结构级**查询。https://ast-grep.github.io/guide/introduction.html
- **Semgrep**——AST/数据流静态分析与规则化搜索，也做 autofix。https://semgrep.dev
- **Codemod.com**——codemod 平台 + JSSG（JS 版结构化搜索/变换 DSL，含语义分析）+ registry。https://docs.codemod.com
- **Comby**——面向解析树结构的语言感知搜索/替换，对 `()`、`{}`、`[]`、字符串和注释用**通用解析器**（而非每种语言做完整解析）；通用回退匹配器可处理新数据格式。https://comby.dev/docs/overview
- **GritQL**——声明式结构化查询/重写语言（Rust，tree-sitter 多语言），可批量修复。https://github.com/getgrit/gritql
- **jscodeshift**——JS/TS 的 AST 到 AST codemod 运行器，提供类 jQuery 的集合 API、builder 方法、解析器选择（`babel`/`flow`/`ts`/`tsx`），经 recast 保留风格；遍历/修改节点后调 `.toSource()` 打印。https://github.com/facebook/jscodeshift
- **recast**——jscodeshift 内部依赖的"JavaScript 语法树变换器、**非破坏性** pretty-printer 与自动 source map 生成器"。核心机制：`recast.parse` 对 AST 做影子拷贝，每个节点保留 `.original` 反向引用；`recast.print` 据此**只重打被修改的节点**，从而保证身份 `recast.print(recast.parse(source)).code === source`。https://github.com/benjamn/recast

### 3.2 有官方支持的结构化编辑器实例

上面是「给现有工具接上 AI」，下面两个则是**结构化编辑 / 语言感知能力本身由官方提供给 AI** 的例子。

**ProjecturEd**——一个 projectional / structure editor：增量、响应式、以 widget 渲染，且**「AI 原生」**。

- https://projectured.org/
- 前身演示视频：
  - [Creating a Simple Web Page with a Projectional Editor](https://www.youtube.com/watch?v=s05SlmZ7ZPc)
  - [Implementing Factorial in Common Lisp](https://www.youtube.com/watch?v=iR2cIF51SVk)

**MoonBit AI toolchain**——把语言感知操作通过 CLI 暴露给 agent 的编译器工具链。

- https://www.moonbitlang.com/
- `moon ide` 把 `moon-lsp` 的能力做成面向 AI 的 CLI：`moon ide peek-def`、`moon ide find-references`、`moon ide rename`、`moon ide hover`、`moon ide outline`、`moon ide analyze`、`moon ide doc`、`moon ide workspace-symbols`
- 相关命令：`moon check`（语法）、`moon prove`（形式验证）、`moon fmt`（格式化）、`moon explain`（编译器诊断）

### 3.3 AI 接入方式：llms.txt / SKILL / MCP

结构化工具接入 AI 目前有三条由浅入深的路径，ast-grep 是覆盖最完整的样本：

1. **提示层（`AGENTS.md`）**——在系统提示里让智能体默认优先使用 `ast-grep --lang <lang> -p '<pattern>'`，除非明确要求纯文本搜索。局限：**依赖模型对工具有较新知识，否则可能不照做**。
2. **喂完整文档（`llms.txt` / `llms-full.txt`）**——把整份文档拼进上下文，显著降低模型「幻觉」出错误规则的概率。已提供官方 llms.txt 的：**ast-grep、Semgrep、Codemod.com**。
3. **Skill（`SKILL.md`）**——`ast-grep/claude-skill` 指导 Claude 编写并使用 ast-grep 规则。局限：**技能不会自动触发**——README 指出「截至 2025 年 11 月，Claude Code 无法自动检测何时应为所有适用场景使用 ast-grep」，仍需在提示层显式引导。
4. **MCP（工具化试错，最可控）**——`ast-grep-mcp` 提供 dump AST、dump pattern query、针对示例代码测试规则等工具，让 AI 迭代开发规则：
   ```
   1. 把用户查询拆成更小的部分
   2. 识别可用于匹配代码的子规则
   3. 用 relational / composite rules 组合成单一规则
   4. 若规则不匹配示例代码，删减子规则并调试不匹配的部分
   5. 用 ast-grep mcp 工具 dump AST 或 dump pattern query
   6. 用 ast-grep mcp 工具针对示例代码测试规则
   ```
   Semgrep 的 MCP 已并入主仓库；Codemod 有官方 MCP；GritQL 只有社区 MCP（`johannhartmann/gritql-mcp`、`goedelsoup/gritql-mcp`）。

来源：
- ast-grep — "Using ast-grep with AI Tools"，https://ast-grep.github.io/advanced/prompting.html
- ast-grep — "What is ast-grep?"，https://ast-grep.github.io/guide/introduction.html
- ast-grep/claude-skill — https://github.com/ast-grep/claude-skill
- ast-grep/ast-grep-mcp — https://github.com/ast-grep/ast-grep-mcp
- 规则开发提示词：https://github.com/ast-grep/ast-grep-mcp/blob/main/ast-grep.mdc

### 3.4 对我们的启示

- **AST 工具正在主动"AI 化"**：skill / MCP / llms.txt 三条路径对应三种可靠性层级（提示依赖模型知识 < 全文档上下文 < 工具化试错迭代）。我们的编辑器若面向智能体，MCP 式"可 dump AST、可测试规则"的接口比纯提示更可控。
- **"技能不会自动触发"是现实约束**，接口设计不能假设模型会自主选择结构化工具，需要显式引导或框架级强制。
- **recast 的 `.original` 影子拷贝**给出了一种轻量的"最小重打印"机制：不修改的代码原样保留，天然是保留格式编辑器的实现范式。

---

## 4. 评测与背景

- **Can It Edit?（CodeEdit 基准）**——一个**基于指令的代码编辑任务**基准，并提供一套许可宽松的训练集。它暴露出"最先进的开源与闭源模型能力之间存在显著差距"，并表明在此数据集上微调开源代码 LLM 能大幅改善编辑能力。对我们而言：它证明「编辑」是与「生成」不同的、需要单独度量与训练的能力——这正是结构化编辑器要服务的场景。来源：Cassano et al., "Can It Edit? Evaluating the Ability of Large Language Models to Follow Code Editing Instructions"，arXiv:2312.12450，https://arxiv.org/abs/2312.12450

---

## 5. 参考项目与灵感来源

本节是**原始链接**，即最初促成这个项目的阅读清单，不是整理过的论证。

- **Future of Programming Lab** — https://neurocy.notion.site/Future-of-Programming-Lab-241d162461a04064ae1fd9ae32bf4cb1
