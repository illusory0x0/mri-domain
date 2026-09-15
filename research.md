# research.md — 结构化编辑 × AI 落地调研

本笔记记录**".AI 编辑代码的接口与表示"**相关的已落地产品/工具/API，面向我们自己的结构化编辑器设计。
主题是广义的：既包括结构感知（AST/LST）方案，也包括文本 diff、编辑协议与智能体编辑接口——因为后者恰恰构成了"为什么需要结构化"的反面证据。
论文向的内容见 `related-works.md`；"背景与评测"见本文件第 3 节。

一手来源访问日期：2026-09-13（Aider、ast-grep 页面于本次复核时重新抓取）。

---

## 1. Aider：LLM 编辑格式的设计与实证

Aider 是"终端里的 AI 结对编程"工具（https://github.com/Aider-AI/aider）。它与本主题最相关之处在于：
**它把"LLM 如何输出一次代码编辑"当成一个可设计、可量化、可消融的接口**，而不是顺带实现一堆 diff 解析。
Aider 为不同模型选择不同的"编辑格式（edit format）"，并可用 `--edit-format` 强制指定。

### 1.1 编辑格式一览

| 格式 | 机制 | 适用/动机 |
| --- | --- | --- |
| `whole` | 返回**整份**更新后的文件（路径写在围栏前） | 最简单，但慢且贵；即便只改几行也要吐全文件 |
| `diff` | 一系列 SEARCH/REPLACE 块，语法类似 git 冲突标记 `<<<<<<< SEARCH` / `=======` / `>>>>>>> REPLACE` | 只返回改动部分，高效 |
| `diff-fenced` | 同 `diff`，但**路径写在围栏内部** | 主要给 Gemini 家族，它们常无法遵守 fence 外的路径写法 |
| `udiff` | 基于标准 unified diff，但**经过修改与简化** | 主要用于 GPT-4 Turbo 家族，显著抑制其"偷懒" |
| `editor-diff` / `editor-whole` | 即 diff/whole 的"精简版"，配合 `--editor-edit-format` 用于 architect 模式 | 编辑器模型只需解释 architect 模型的纯文本改动指令，只专注编辑而非解题 |

来源：Aider — "Edit formats"，https://aider.chat/docs/more/edit-formats.html 。

### 1.2 核心设计原则（四条）

Aider 在博客里把成功的编辑格式归结为四条原则（原文大写：FAMILIAR / SIMPLE / HIGH LEVEL / FLEXIBLE）：

1. **FAMILIAR（熟悉）**——选模型训练数据里见过很多次的格式。unified diff 是 `git diff` 的默认输出，模型见得最多。
2. **SIMPLE（简单）**——避免转义、语法开销，以及行号/行数这类**脆弱定位符**。
3. **HIGH LEVEL（高层）**——鼓励模型把编辑组织为"函数/方法等实质代码块的新版本"，而不是逐行的外科手术式最小改动。
4. **FLEXIBLE（灵活）**——在**解释**模型给出的编辑指令时尽可能宽容。

### 1.3 关键实证结论（可量化）

- **unified diff 让 GPT-4 Turbo 在"偷懒"基准上从 20% 提升到 61%**（`gpt-4-1106-preview`，89 个 Python 重构任务；偷懒任务从 12 个降到 4 个）。旧 `gpt-4-0613` 从 26% 提升到 59%（其 8k 上下文对 28% 的大文件放不下，理论上限 72%）。
- **行号对 LLM 极不友好**："GPT is terrible at working with source code line numbers… a general observation about *any* use of line numbers in editing formats"。因此 Aider 让模型**省略 hunk 行号**，把每个 hunk 当作搜索/替换执行。
- **高层 diff 提示至关重要**：不做 high-level diff 提示时，**编辑错误增加 30–50%**（diff 应用失败或应用错误产生非法代码）。原因一：整块替换比逐行交错更不易混乱；原因二：高层 hunk 更长，更不容易误配到代码别处。
- **灵活打补丁至关重要**：关闭灵活匹配后，在 Exercism 基准上**编辑错误增加 9 倍**。Aider 的容错策略包括：
  - 对 minus/space 行与 space/plus 行重新做一次真正的 unified diff 来**规范化** hunk；
  - 找回模型想新增却忘了标 `+` 的行；
  - 用**相对前导空白**匹配，容忍整体缩进/反缩进；
  - 把大 hunk 拆成只含单个连续 `+`/`-` 段的**重叠子 hunk**，逐个独立尝试；
  - 调整上下文窗口大小/偏移，并逐级放宽应用策略。
- **不要用 JSON / function calling 做代码编辑**："stuffing *source code* into JSON is complicated and error prone"，转义问题会让解包后的代码语法错误或直接解码失败；OpenAI 的 structured format 反而更差。
- **关键句**："any naive or direct use of structured diff formats is pretty much doomed to failure"——结构化补丁格式"直接照搬"会失败，必须配合高层化 + 灵活应用。

来源：
- Aider — "Edit formats"，https://aider.chat/docs/more/edit-formats.html
- Aider — "Unified diffs make GPT-4 Turbo 3X less lazy"（2023-12-21, Paul Gauthier），https://aider.chat/2023/12/21/unified-diffs.html
- Aider 重构基准源码：https://github.com/Aider-AI/refactor-benchmark

### 1.4 对我们的启示

- **编辑格式是 AI 编辑器的一等设计对象**，且**格式选择随模型而变**，不能只支持一种。
- 定位应尽量**避免行号**，转为**语义块的搜索/替换**（内容寻址）。
- 应用补丁必须**宽容**：规范化、找回漏标的增行、相对缩进匹配、大块拆分。僵化应用是主要错误来源。
- 不要把源码塞进 JSON；模型对"被程序消费的文本数据"（diff）比对"结构化 JSON"更严谨。
- 这些结论来自 2023 年的 GPT-4 世代，但"格式—模型"耦合、行号脆弱、灵活匹配这三点对今天仍有参考价值。

---

## 2. B. 结构感知工具 × AI 智能体

这一组的共同点：**底层是 AST/结构感知工具，上层通过 skill / MCP / prompt 交给 AI 智能体使用**。它们是"结构化编辑"从"人能用的 CLI"走向"AI 能用的接口"的现成样本。

### 2.1 ast-grep（重点）

ast-grep 是"基于 AST 管理代码的新工具"，核心是"用相同模式的代码去搜索代码"，与基于文本的 grep/ripgrep 相对。它为 AI 使用提供了**四种由浅入深的接入方式**：

1. **Claude Code Skill**——`ast-grep/claude-skill` 技能教 Claude 编写和使用 ast-grep 规则做高级代码搜索，可发现"没有错误处理的 async 函数""使用某个 hook 的 React 组件""参数多于 3 个的函数""类方法里的 console.log"等**结构级**查询。放入 `~/.claude/skills/` 即可。
2. **`AGENTS.md` 简单提示**——在系统提示里让智能体默认优先使用 `ast-grep --lang <lang> -p '<pattern>'`，除非明确要求纯文本搜索。原文提醒：**这要求模型对 ast-grep 有较新知识，否则可能不照做**。
3. **喂完整文档给 LLM**——`llms-full.txt`（https://ast-grep.github.io/llms-full.txt）把整份文档拼成上下文，显著降低模型"幻觉"出错误规则的概率。
4. **MCP + 子智能体（进阶）**——`ast-grep-mcp` 是一个实验性 MCP 服务器，连接 Cursor / Claude Code 等，让 AI 通过**试错迭代**开发规则，工具包括 dump AST、dump pattern query、针对示例代码测试规则。其规则开发流程被固化为：
   ```
   1. 把用户查询拆成更小的部分
   2. 识别可用于匹配代码的子规则
   3. 用 relational / composite rules 组合成单一规则
   4. 若规则不匹配示例代码，删减子规则并调试不匹配的部分
   5. 用 ast-grep mcp 工具 dump AST 或 dump pattern query
   6. 用 ast-grep mcp 工具针对示例代码测试规则
   ```

已知局限：**智能体技能可能不会自动触发**。skill README 指出"截至 2025 年 11 月，Claude Code 无法自动检测何时应为所有适用场景使用 ast-grep"，所以仍需在提示层显式引导。

来源：
- ast-grep — "Using ast-grep with AI Tools"，https://ast-grep.github.io/advanced/prompting.html
- ast-grep — "What is ast-grep?"，https://ast-grep.github.io/guide/introduction.html
- ast-grep/claude-skill — https://github.com/ast-grep/claude-skill
- ast-grep/ast-grep-mcp — https://github.com/ast-grep/ast-grep-mcp
- 规则开发提示词：https://github.com/ast-grep/ast-grep-mcp/blob/main/ast-grep.mdc

### 2.2 其余结构感知工具（非 LLM 引擎，作地基）

- **Comby**——面向解析树结构的语言感知搜索/替换，对 `()`、`{}`、`[]`、字符串和注释用**通用解析器**（而非每种语言做完整解析）；通用回退匹配器可处理新数据格式。https://comby.dev/docs/overview
- **Tree-sitter**——解析器生成器 + 增量 CST 库，"在源文件被编辑时高效更新语法树"，且容错（有语法错误也能给有用结果）；提供多语言语法与绑定（C/Go/Java/JS-Wasm/Python/Rust/Swift…），专为逐键编辑与 IDE 设计。https://tree-sitter.github.io/tree-sitter/
- **OpenRewrite**——基于**无损语义树（LST）**的自动重构引擎：LST 带**类型属性**（跨文件/项目解析符号）且**保留格式**（空白存在树里，能重建原始源码而不破坏格式）。改动写在 **Visitors** 里，聚合成 **Recipes**。局限：本地 LST"必须能放进内存"。https://docs.openrewrite.org/ ；LST 页面：https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees
- **jscodeshift**——JS/TS 的 AST 到 AST codemod 运行器，提供类 jQuery 的集合 API、builder 方法、解析器选择（`babel`/`flow`/`ts`/`tsx`），经 recast 保留风格；遍历/修改节点后调 `.toSource()` 打印。https://github.com/facebook/jscodeshift
- **JetBrains MPS**——投射式编辑器（Projectional Editor），用户用**非文本记号**（数学记号、图表、表单）进行投射式编辑，定位为 DSL 工作台。https://www.jetbrains.com/mps/
  - 注意：专门的 `/mps/concepts/` 页面在抓取时未返回服务端内容，机制细节**未验证**。

### 2.3 对我们的启示

- **AST 工具正在主动"AI 化"**：skill / MCP / llms.txt 三条路径对应三种可靠性层级（提示依赖模型知识 < 全文档上下文 < 工具化试错迭代）。我们的编辑器若面向智能体，MCP 式"可 dump AST、可测试规则"的接口比纯提示更可控。
- **"技能不会自动触发"是现实约束**，接口设计不能假设模型会自主选择结构化工具，需要显式引导或框架级强制。
- **保留格式 + 带类型的树 + 增量更新**三件事（分别来自 OpenRewrite / OpenRewrite / Tree-sitter）是结构化编辑器需要同时满足的工程属性。

---

## 3. 背景与评测

本节收录**文本层**的智能体编辑系统与评测基准。它们不做结构感知编辑，因此不是我们要复刻的对象；但它们说明了"编辑接口设计"和"编辑能力评测"这两个问题，是本调研的动机与验证手段。

### 3.1 SWE-agent：编辑接口设计影响性能（保留）

SWE-agent 是一个研究系统，核心贡献是**智能体-计算机接口（agent-computer interface, ACI）**。论文声称该 ACI"显著增强智能体创建和编辑代码文件、导航整个仓库以及执行测试的能力"；发表时在 SWE-bench 上 pass@1 为 **12.5%**，在 HumanEvalFix 上为 **87.7%**（代码/数据/演示见 swe-agent.com）。其核心论点是 **ACI 设计"会影响智能体的行为与性能"**。

对我们而言：SWE-agent 的编辑器仍是文本/行导向的（不是 AST），但它是"编辑接口是一等设计变量"这一命题最直接的实证来源——与 Aider 的结论同向（格式/接口决定成败），只是发生在智能体工具层而非模型输出层。
来源：Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering"，arXiv:2405.15793，https://arxiv.org/abs/2405.15793

### 3.2 Can It Edit?：编辑能力是可独立评测的（保留）

Can It Edit?（CodeEdit 基准）是一个**基于指令的代码编辑任务**基准，外加一个许可宽松的训练集。它暴露出"最先进的开源与闭源模型能力之间存在显著差距"，并表明在此数据集上微调开源代码 LLM 能大幅改善编辑能力。

对我们而言：它证明"编辑"是与"生成"不同的、需要单独度量与训练的能力——这正是结构化编辑器要服务的场景。
来源：Cassano et al., "Can It Edit? Evaluating the Ability of Large Language Models to Follow Code Editing Instructions"，arXiv:2312.12450，https://arxiv.org/abs/2312.12450

### 脚注：边缘相关，暂不展开

- **OpenHands**（Wang et al., arXiv:2407.16741，https://arxiv.org/abs/2407.16741）——通用智能体平台，以"与人类开发者相似的方式"写代码、用命令行、浏览网页。与编辑表示关系很弱，仅作生态背景。
- **SWE-bench**（Jimenez et al., arXiv:2310.06770，https://arxiv.org/abs/2310.06770）——横跨 12 个 Python 仓库的 2,294 个真实 issue/PR，需跨函数/类/文件协调编辑（发表时最好模型仅解决 1.96%）。它是**任务难度基准**，不涉及编辑表示；若我们日后要做端到端评测，可复用其 harness。

