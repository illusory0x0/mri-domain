# 结构化编辑 + AI

**范围：** "结构感知 / 结构化编辑"在 AI 相关工具、API、智能体编程系统与研究中的各种形态——AST/语法树编辑、结构化 diff 与补丁、基于编辑的模型，以及 schema/语法约束生成。

**所有来源的访问日期：** 2026-09-13。
**方法：** 仅使用一手来源（第一方文档、源码仓库，以及论文本身）。每条论述均就地标注来源。无法确认的条目标记为 **UNVERIFIED（未验证）**。本笔记区分 **(a) 已落地的产品/API**、**(b) 研究原型/论文** 和 **(c) 演示**。

---

## 1. 定义

在一手来源中，"结构化编辑"并非单一概念；它指代若干彼此不同的技术思路。下文的每个含义都锚定到某个一手来源实际使用它的地方。

1. **AST 感知的搜索与重写（结构而非文本）。** ast-grep 自述为"一个基于 AST 管理代码的新工具"，其核心机制是"用相同模式的代码去搜索代码"，并与文本工具作对比："用基于文本的工具搜索代码很快，但不精确。我们通常更愿意把代码解析成抽象语法树，以获得精确匹配。" [来源：ast-grep — "What is ast-grep?"，https://ast-grep.github.io/guide/introduction.html，访问于 2026-09-13]
2. **匹配解析树的语法结构。** Comby 是"一种轻量级地匹配程序解析树中语法结构的方式，例如表达式与函数块"，它通过对成对分隔符、字符串和注释的通用解析器实现，而非为每种语言做完整解析。[来源：Comby — "Overview"，https://comby.dev/docs/overview，访问于 2026-09-13]
3. **增量维护语法树。** Tree-sitter 是"一个解析器生成工具和增量解析库"，它"能为源文件构建具体语法树，并在源文件被编辑时高效更新语法树"，且"足够健壮，即使在存在语法错误时也能给出有用的结果"。[来源：Tree-sitter — "Introduction"，https://tree-sitter.github.io/tree-sitter/，访问于 2026-09-13]
4. **语义化、保留格式的树变换（重构）。** OpenRewrite "通过修改表示源代码的无损语义树（Lossless Semantic Trees，LST），再把修改后的树打印回源代码"来工作。LST 是"代码的一种树表示"，它带有**类型属性**且**保留格式**（空白被存储在树中，因此输出"能重建原始源代码而不破坏格式"）。[来源：OpenRewrite — "Introduction"，https://docs.openrewrite.org/，访问于 2026-09-13] [来源：OpenRewrite — "Lossless Semantic Trees (LST)"，https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees，访问于 2026-09-13]
5. **AST 到 AST 的 codemod。** jscodeshift 是"一个用于在多个 JavaScript 或 TypeScript 文件上运行 codemod 的工具箱"；它封装了 recast，"一个 AST 到 AST 的变换工具，也尽量保留原始代码的风格"。变换通过遍历/修改 AST 节点集合进行，并调用 `.toSource()` 打印。[来源：facebook/jscodeshift — README，https://github.com/facebook/jscodeshift，访问于 2026-09-13]
6. **面向 LLM 编辑的结构化 diff / 补丁格式。** Aider 列举了 LLM 返回文件改动的多种"编辑格式"：`whole`（整文件）、`diff`（类似 git 冲突标记 `<<<<<<< SEARCH` / `>>>>>>> REPLACE` 的搜索/替换块）、`diff-fenced`，以及 `udiff`（一种"简化并修改过"的统一 diff）。[来源：Aider — "Edit formats"，https://aider.chat/docs/more/edit-formats.html，访问于 2026-09-13]
7. **schema 约束生成（结构化输出）。** OpenAI 的 Structured Outputs "确保模型始终生成符合你所提供 JSON Schema 的响应"，这有别于 JSON 模式下仅仅是合法的 JSON。[来源：OpenAI — "Structured model outputs"，https://platform.openai.com/docs/guides/structured-outputs，访问于 2026-09-13] Anthropic 的 strict tool use "通过把模型的 token 采样约束到符合 schema 的输出，保证 Claude 的工具输入匹配你的 JSON Schema（这项技术称为语法约束采样 grammar-constrained sampling）。" [来源：Anthropic — "Strict tool use"，https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use，访问于 2026-09-13] Model Context Protocol（MCP）将工具参数编码为 JSON Schema（`inputSchema`），并将可选的输出结构编码为 `outputSchema`。[来源：Model Context Protocol — "Tools"，https://modelcontextprotocol.io/specification/2025-06-18/server/tools，访问于 2026-09-13]
8. **语法/解析器约束解码。** PICARD 通过"增量解析"约束自回归解码器，方式是"在每个解码步拒绝不可接受的 token"。[来源：Scholak et al., "PICARD: Parsing Incrementally for Constrained Auto-Regressive Decoding from Language Models"，arXiv:2109.05093，https://arxiv.org/abs/2109.05093，访问于 2026-09-13] Synchromesh 的"约束语义解码（Constrained Semantic Decoding, CSD）"通过"对部分输出的约束"，把输出约束到"目标语言中一组合法程序的集合"。[来源：Poesia et al., "Synchromesh: Reliable code generation from pre-trained language models"，arXiv:2201.11227，https://arxiv.org/abs/2201.11227，访问于 2026-09-13]
9. **投射式编辑（Projectional editing）。** JetBrains MPS 提供一款"投射式编辑器（Projectional Editor）"，用户借此"使用非文本记号进行投射式编辑，包括数学记号、图表和表单"。[来源：JetBrains — "MPS: The Domain-Specific Language Creator"，https://www.jetbrains.com/mps/，访问于 2026-09-13]（专门的 `/mps/concepts/` 页面在抓取时未渲染出服务端内容；见"缺口"部分。）
10. **把编辑作为一等建模对象。** CoditT5 提出"一种显式建模编辑的新预训练目标"，用于"与软件相关的编辑任务"，例如注释更新、缺陷修复和代码评审。[来源：Zhang et al., "CoditT5: Pretraining for Source Code and Natural Language Editing"，arXiv:2208.05446，https://arxiv.org/abs/2208.05446，访问于 2026-09-13] "Learning to Represent Edits" 通过把一个"神经编辑器"与一个"编辑编码器"结合，学习"编辑的分布式表示"，从而能把一个编辑应用到新的输入上。[来源：Yin et al., "Learning to Represent Edits"，arXiv:1810.13337，https://arxiv.org/abs/1810.13337，访问于 2026-09-13]

---

## 2. 产品与 API

### 2a. 结构感知的搜索 / 重写工具（非 LLM 引擎）

- **ast-grep** — 基于 AST 的搜索、lint 与重写。模式在树上匹配；重写可通过关系/组合规则组合。它通过 Tree-sitter 支持多种语言。文档示例：`ast-grep --pattern 'var code = $PAT' --rewrite 'let code = $PAT' --lang js`。[来源：ast-grep — "What is ast-grep?"，https://ast-grep.github.io/guide/introduction.html，访问于 2026-09-13]
- **Comby** — 面向解析树结构的、语言感知的搜索/替换，对 `()`、`{}`、`[]`、字符串和注释使用通用解析器；通用回退匹配器可处理新数据格式。[来源：Comby — "Overview"，https://comby.dev/docs/overview，访问于 2026-09-13]
- **Tree-sitter** — 解析器生成器加增量 CST 库；附带多种语法和绑定（C、Go、Java、JavaScript/Wasm、Python、Rust、Swift 等），并明确为逐键编辑和容错而设计。文档列出了其在解析/IDE 研究方面的设计影响（如 "Efficient and Flexible Incremental Parsing"）。[来源：Tree-sitter — "Introduction"，https://tree-sitter.github.io/tree-sitter/，访问于 2026-09-13]
- **OpenRewrite** — 基于 LST 的自动重构引擎。修改在"Visitors"中完成，并聚合为"Recipes"。其 LST 带类型属性（跨文件/项目解析符号）且保留格式；在本地，LST"必须能放进内存"。[来源：OpenRewrite — "Introduction"，https://docs.openrewrite.org/，访问于 2026-09-13] [来源：OpenRewrite — "Lossless Semantic Trees (LST)"，https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees，访问于 2026-09-13]
- **jscodeshift** — 面向 JavaScript/TypeScript 的 AST 到 AST codemod 运行器；提供类似 jQuery 的集合 API、构建器方法、解析器选择（`babel`、`flow`、`ts`、`tsx`），并通过 recast 保留风格。[来源：facebook/jscodeshift — README，https://github.com/facebook/jscodeshift，访问于 2026-09-13]

### 2b. AI 编程智能体/工具如何表示编辑

- **Aider（已落地的开源产品）。** Aider "使用各种'编辑格式'让 LLM 编辑源文件"，并按模型选择格式。格式包括：`whole`、`diff`（SEARCH/REPLACE 块）、`diff-fenced`（路径写在围栏内，用于 Gemini 模型）、`udiff`（统一 diff，"主要用于……GPT-4 Turbo 系列"）。[来源：Aider — "Edit formats"，https://aider.chat/docs/more/edit-formats.html，访问于 2026-09-13] 在其设计说明中，统一 diff 让 GPT-4 Turbo 在 Aider 的"懒惰性"重构基准上从 **20% 提升到 61%**，而关闭 Aider 的灵活补丁匹配"会出现 9 倍于原来的编辑错误"。Aider 指出"GPT 极不擅长处理源代码行号"，因此它指示模型省略 hunk 行号，并把 hunk 当作搜索/替换处理。[来源：Aider — "Unified diffs make GPT-4 Turbo 3X less lazy"，https://aider.chat/2023/12/21/unified-diffs.html，访问于 2026-09-13]
- **Anthropic 文本编辑器工具（已落地的 API）。** Anthropic 提供一个由 schema 定义的 `text_editor` 工具（名为 `str_replace_based_edit_tool`），命令包括 `view`、`str_replace`、`create` 和 `insert`。`str_replace` 接收 `old_str`（"必须精确匹配，包括空白和缩进"）与 `new_str`；`insert` 在 1 起始的行号之后插入文本；`view` 返回文件内容，可选地加上行号前缀。[来源：Anthropic — "Text editor tool"，https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool，访问于 2026-09-13]
- **Anthropic 的工具使用总体（已落地的 API）。** 客户端工具返回 `tool_use` 块，由应用执行并以 `tool_result` 应答；工具通过 `input_schema` 描述。[来源：Anthropic — "Tool use with Claude"，https://docs.anthropic.com/en/docs/build-with-claude/tool-use，访问于 2026-09-13]
- **OpenAI Structured Outputs + 函数调用（已落地的 API）。** Structured Outputs 通过 `text.format` / `response_format` 支持 JSON Schema，并有把原生类型绑定的库（Pydantic、Zod）。函数调用用 JSON Schema 定义工具，并有 `strict` 模式："把 `strict` 设为 `true` 将确保函数调用可靠地遵守函数 schema，而非尽力而为"，它通过 Structured Outputs 特性实现，并要求 `additionalProperties: false` 且所有字段必填。[来源：OpenAI — "Structured model outputs"，https://platform.openai.com/docs/guides/structured-outputs，访问于 2026-09-13] [来源：OpenAI — "Function calling"，https://platform.openai.com/docs/guides/function-calling，访问于 2026-09-13]
- **Anthropic strict tool use（已落地的 API）。** 在工具定义上设置 `"strict": true`；随后 `input` "严格遵循 `input_schema`"。Anthropic 说明其机制为语法约束采样。[来源：Anthropic — "Strict tool use"，https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use，访问于 2026-09-13]
- **MCP 工具（已落地的协议规范）。** MCP 服务器声明 `tools` 能力；客户端调用 `tools/list` 和 `tools/call`。一个工具有 `name`、`description`、`inputSchema`、可选的 `outputSchema` 和 `annotations`。结果可以是无结构的 `content` 或 `structuredContent`；若给定 `outputSchema`，"服务器必须提供符合该 schema 的结构化结果"。注意：MCP 明确指出 `structuredContent`"与 LLM 的'结构化输出'（schema 约束的模型生成）无关"。[来源：Model Context Protocol — "Tools"，https://modelcontextprotocol.io/specification/2025-06-18/server/tools，访问于 2026-09-13]
- **Cursor（已落地的产品）。** Cursor 文档描述其为一个"用于构建宏大软件的编程智能体"，能够"规划并构建功能"、"查找并修复缺陷"，以及通过检查 diff 和运行检查"评审改动"。抓取到的文档**未**记录 AST 级别的编辑表示。[来源：Cursor — "Cursor Documentation"，https://cursor.com/docs，访问于 2026-09-13]
- **OpenAI Codex CLI（已落地的开源智能体）。** 被描述为"一个在本地计算机上运行的、来自 OpenAI 的编程智能体"。[来源：openai/codex — README，https://github.com/openai/codex，访问于 2026-09-13] 其内部补丁/工具格式在此**未验证**（见"缺口"）。

---

## 3. 研究

### 3a. 结构感知的程序表示与生成

- **code2vec** 通过"把代码分解为其抽象语法树中的一组路径"，把代码片段表示为定长向量，联合学习路径表示与聚合；用于在 1400 万个方法上做方法名预测。[来源：Alon et al., "code2vec: Learning Distributed Representations of Code"，arXiv:1803.09473，https://arxiv.org/abs/1803.09473，访问于 2026-09-13]
- **Learning to Represent Programs with Graphs** 提出"用图表示代码的语法和语义结构"，使用门控图神经网络，在 VarNaming 和 VarMisuse 上评估；据报告在成熟开源项目中发现了缺陷。[来源：Allamanis et al., "Learning to Represent Programs with Graphs"，arXiv:1711.00740，https://arxiv.org/abs/1711.00740，访问于 2026-09-13]
- **树到树神经网络用于程序翻译** 把源树翻译为目标树，在每一步借助注意力机制把源子树翻译为目标子树。[来源：Chen et al., "Tree-to-tree Neural Networks for Program Translation"，arXiv:1802.03691，https://arxiv.org/abs/1802.03691，访问于 2026-09-13]
- **CodeT5** 是一个标识符感知的编码器-解码器预训练模型，带标识符标注目标以及 NL-PL 双模态双重生成。[来源：Wang et al., "CodeT5: Identifier-aware Unified Pre-trained Encoder-Decoder Models…"，arXiv:2109.00859，https://arxiv.org/abs/2109.00859，访问于 2026-09-13]
- **StructCoder** 通过"源代码的语法树与数据流图"使编码器结构感知，并加入解码器辅助任务"AST（Abstract Syntax Tree）路径预测和数据流预测"。它报告在 CodeXGLUE 翻译与文本到代码上达到 SOTA。[来源：Tipirneni et al., "StructCoder: Structure-Aware Transformer for Code Generation"，arXiv:2206.05239，https://arxiv.org/abs/2206.05239，访问于 2026-09-13]
- **中间填充（Fill-in-the-middle，FIM）** 通过把一段文本从中间移到末尾，训练自回归 LM 做填充；作者"建议未来的自回归语言模型默认用 FIM 训练"。[来源：Bavarian et al., "Efficient Training of Language Models to Fill in the Middle"，arXiv:2207.14255，https://arxiv.org/abs/2207.14255，访问于 2026-09-13]

### 3b. 基于编辑的模型

- **CoditT5** 用一个"显式建模编辑"的目标预训练，在注释更新、缺陷修复和自动代码评审上微调，并报告通过用标准生成模型重排后在三者上均达 SOTA。[来源：Zhang et al., "CoditT5…"，arXiv:2208.05446，https://arxiv.org/abs/2208.05446，访问于 2026-09-13]
- **Learning to Represent Edits** 引入"学习编辑的分布式表示"，适用于自然语言和源代码。[来源：Yin et al., arXiv:1810.13337，https://arxiv.org/abs/1810.13337，访问于 2026-09-13]

### 3c. 约束 / 语法引导解码

- **PICARD** 通过增量解析约束解码，"在每个解码步拒绝不可接受的 token"，并把勉强可用的 T5 文本到 SQL 模型变为 Spider/CoSQL 上的 SOTA。[来源：Scholak et al., arXiv:2109.05093，https://arxiv.org/abs/2109.05093，访问于 2026-09-13]
- **Synchromesh** 引入约束语义解码（CSD），"把输出约束到一组合法程序"，在不重新训练的情况下强制"语法、作用域、类型规则和上下文逻辑"。[来源：Poesia et al., arXiv:2201.11227，https://arxiv.org/abs/2201.11227，访问于 2026-09-13]
- **语法提示（Grammar prompting）** 把 BNF 语法作为外部领域约束输入；推理时 LLM 先预测一个语法，再按该语法生成输出。[来源：Wang et al., "Grammar Prompting for Domain-Specific Language Generation with Large Language Models"，arXiv:2305.19234，https://arxiv.org/abs/2305.19234，访问于 2026-09-13]
- **基于 FSM 的引导生成（Outlines）** 把生成"重新表述为有限状态机状态之间的转移"，通过索引模型词表实现正则和上下文无关语法引导，并"保证生成文本的结构"。[来源：Willard & Louf, "Efficient Guided Generation for Large Language Models"，arXiv:2307.09702，https://arxiv.org/abs/2307.09702，访问于 2026-09-13]
- **监视器引导解码（Monitor-guided decoding，MGD）** 用静态分析作为监视器，结合仓库上下文引导解码，提升编译率和下一个标识符匹配率；作者报告 SantaCoder-1.1B 在这些指标上可胜过 text-davinci-003。[来源：Agrawal et al., "Guiding Language Models of Code with Global Context using Monitors"，arXiv:2306.10763，https://arxiv.org/abs/2306.10763，访问于 2026-09-13]

### 3d. 程序修复

- **CURE** 是一种基于 NMT 的自动程序修复技术，含三个组成部分：PL 模型预训练、"一种新的代码感知搜索策略，通过专注于可编译补丁以及与缺陷代码长度相近的补丁来找到更多正确修复"，以及子词分词；它报告修复了 57 个 Defects4J 和 26 个 QuixBugs 缺陷。[来源：Jiang et al., "CURE: Code-Aware Neural Machine Translation for Automatic Program Repair"，arXiv:2103.00073，https://arxiv.org/abs/2103.00073，访问于 2026-09-13]

### 3e. 编辑 / 仓库级智能体的基准

- **SWE-bench**：横跨 12 个 Python 仓库的 2,294 个真实 GitHub issue/PR 问题，需要协调地跨多个函数/类/文件编辑；在发表时，最好的模型（Claude 2）仅解决"区区 1.96%"。[来源：Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?"，arXiv:2310.06770，https://arxiv.org/abs/2310.06770，访问于 2026-09-13]
- **Can It Edit?（CodeEdit 基准）**：一个基于指令的代码编辑任务基准，外加一个许可宽松的训练集；它暴露出"最先进的开源与闭源模型能力之间存在显著差距"，并表明在此数据集上微调开源代码 LLM 能大幅改善编辑能力。[来源：Cassano et al., "Can It Edit? Evaluating the Ability of Large Language Models to Follow Code Editing Instructions"，arXiv:2312.12450，https://arxiv.org/abs/2312.12450，访问于 2026-09-13]

---

## 4. 演示 / 原型

- **SWE-agent** — 带"智能体-计算机接口（agent-computer interface，ACI）"的研究系统。论文称该 ACI"显著增强智能体创建和编辑代码文件、导航整个仓库以及执行测试的能力"，发表时在 SWE-bench 上 pass@1 为 12.5%，在 HumanEvalFix 上为 87.7%；代码/数据/演示见 swe-agent.com。[来源：Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering"，arXiv:2405.15793，https://arxiv.org/abs/2405.15793，访问于 2026-09-13]
- **OpenHands** — 面向智能体的开放平台，它们"以与人类开发者相似的方式与世界交互：编写代码、与命令行交互、浏览网页"，MIT 许可。[来源：Wang et al., "OpenHands: An Open Platform for AI Software Developers as Generalist Agents"，arXiv:2407.16741，https://arxiv.org/abs/2407.16741，访问于 2026-09-13]
- **code2vec 交互式演示** — 在线演示见 code2vec.org（论文中提及）。[来源：Alon et al., arXiv:1803.09473，https://arxiv.org/abs/1803.09473，访问于 2026-09-13]
- **面向 AI 智能体的 ast-grep** — ast-grep 项目提供一个"Claude Code 插件市场"，其中包含一个技能，"教 Claude 如何编写和使用 ast-grep 规则以执行高级代码搜索"，另有一个 MCP 服务器（`ast-grep-mcp`）用于迭代式规则开发（导出 AST、测试规则、修改）。[来源：ast-grep/agent-skill — README，https://github.com/ast-grep/agent-skill，访问于 2026-09-13] [来源：ast-grep — "Using ast-grep with AI Tools"，https://ast-grep.github.io/advanced/prompting.html，访问于 2026-09-13]
- **MPS 投射式编辑器** — JetBrains 将 MPS 定位为一个 DSL 工作台，其"Projectional Editor"支持非文本记号。[来源：JetBrains — "MPS"，https://www.jetbrains.com/mps/，访问于 2026-09-13]

---

## 5. 开放问题 / 局限（由一手来源陈述）

- **对 LLM 而言行号很脆弱。** Aider："GPT 极不擅长处理源代码行号。这是关于编辑格式中*任何*行号用法的一个普遍观察。" [来源：Aider — "Unified diffs…"，https://aider.chat/2023/12/21/unified-diffs.html，访问于 2026-09-13]
- **把源代码包进 JSON 容易出错。** Aider："把源代码塞进 JSON 既复杂又容易出错……由于转义问题，GPT 的代码从 JSON 解包时常常语法不正确，或者 JSON 解码直接失败。" [来源：Aider — "Unified diffs…"，https://aider.chat/2023/12/21/unified-diffs.html，访问于 2026-09-13]
- **僵化的补丁应用经常失败。** Aider：关闭灵活补丁匹配后，在其 Exercism 基准上编辑错误**增加 9 倍**；因此 Aider 会规范化、重新生成 diff 并放宽匹配。[来源：Aider — "Unified diffs…"，https://aider.chat/2023/12/21/unified-diffs.html，访问于 2026-09-13]
- **没有约束时 LLM 会违反语法/语义。** Synchromesh：模型"常违反其输出语言的语法和语义规则，限制了实际可用性。" [来源：Poesia et al., arXiv:2201.11227，https://arxiv.org/abs/2201.11227，访问于 2026-09-13] PICARD：微调后的模型"常生成无效代码，使其无法使用。" [来源：Scholak et al., arXiv:2109.05093，https://arxiv.org/abs/2109.05093，访问于 2026-09-13]
- **LM 缺乏仓库/全局上下文。** Monitor-guided decoding：LM"对这类全局上下文感知有限，最终产生幻觉。" [来源：Agrawal et al., arXiv:2306.10763，https://arxiv.org/abs/2306.10763，访问于 2026-09-13]
- **自动程序修复（APR）的搜索空间/策略局限。** CURE：现有 NMT 修复方法的"搜索空间常常不包含正确修复"，且"搜索策略忽略软件知识，例如严格的代码语法。" [来源：Jiang et al., arXiv:2103.00073，https://arxiv.org/abs/2103.00073，访问于 2026-09-13]
- **仓库级编辑很难。** SWE-bench：解决问题"常常需要理解并协调跨多个函数、类甚至文件的改动"，而模型只解决了极少数。[来源：Jimenez et al., arXiv:2310.06770，https://arxiv.org/abs/2310.06770，访问于 2026-09-13]
- **开源/闭源编辑差距。** Can It Edit? 发现开源与闭源模型在基于指令的代码编辑上存在"显著差距"。[来源：Cassano et al., arXiv:2312.12450，https://arxiv.org/abs/2312.12450，访问于 2026-09-13]
- **智能体接口设计很重要。** SWE-agent 的论点在于 ACI 设计"会影响智能体的行为与性能"。[来源：Yang et al., arXiv:2405.15793，https://arxiv.org/abs/2405.15793，访问于 2026-09-13]
- **结构化输出有 schema 限制。** OpenAI strict 模式："JSON schema 的某些特性不受支持"；对微调模型，schema 缓存存在延迟/ZDR 方面的注意事项。[来源：OpenAI — "Function calling"，https://platform.openai.com/docs/guides/function-calling，访问于 2026-09-13]
- **strict tool use 有数据约束。** Anthropic：PHI"不得包含在工具 schema 定义中"；计算机使用与浏览器使用工具集不接受 `strict: true`。[来源：Anthropic — "Strict tool use"，https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use，访问于 2026-09-13]
- **MCP 安全。** MCP 要求服务器验证输入、客户端验证结果；工具注解"除非来自受信任的服务器，否则必须……被视为不可信"。[来源：Model Context Protocol — "Tools"，https://modelcontextprotocol.io/specification/2025-06-18/server/tools，访问于 2026-09-13]
- **本地 LST 内存受限。** OpenRewrite：本地"LST **必须**能放进内存"（不同于 Moderne 的分片处理）。[来源：OpenRewrite — "Lossless Semantic Trees (LST)"，https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees，访问于 2026-09-13]
- **智能体技能可能不会自动触发。** ast-grep 技能 README 指出："截至 2025 年 11 月，Claude Code 无法自动检测何时应为所有适用场景使用 ast-grep。" [来源：ast-grep/agent-skill — README，https://github.com/ast-grep/agent-skill，访问于 2026-09-13]

---

## 6. 无法验证的内容 / 缺口

- **终端 AI 编辑器的 AST 感知能力。** 我抓取到的 Cursor 文档描述了智能体式规划/diff 评审，但未描述 AST/CST 级别的编辑表示。此处不主张 Cursor 以结构化方式编辑。**UNVERIFIED。**
- **OpenAI Codex 内部补丁格式。** 仓库 README 描述了 Codex CLI，但未描述其 `apply_patch`/补丁语法；猜测的 `codex-rs/apply-patch/README.md` 原始 URL 返回 404。**UNVERIFIED。**
- **JetBrains MPS 投射式编辑器机制。** 主页确认了"Projectional Editor"的说法，但 `/mps/concepts/` 页面在抓取时未返回服务端渲染内容。**UNVERIFIED（深度）。**
- **GumTree / 细粒度 AST 差分。** 该经典论文的主机（hal.science）以机器人挑战拦截了抓取，因此未纳入关于它的任何一手论述。**UNVERIFIED。**
- **模型/预览状态。** 若干抓取到的第一方文档（OpenAI 与 Anthropic，日期为 2026 年）引用了超出常识范围的模型名和预览；我引用的是概念而非模型能力，但读者应把涉及具体模型的措辞视为某一时点的情况。
- **我最初尝试的两个 arXiv ID 是错的**（2208.01105 和 2306.02021 指向无关论文）；CoditT5（2208.05446）和 StructCoder（2206.05239）的正确 ID 已抓取，也是实际引用的那些。

---

## 7. 参考文献

产品 / API / 源码仓库（均访问于 2026-09-13）：

1. Aider — "Edit formats"，https://aider.chat/docs/more/edit-formats.html
2. Aider — "Unified diffs make GPT-4 Turbo 3X less lazy"，https://aider.chat/2023/12/21/unified-diffs.html
3. ast-grep — "What is ast-grep?"，https://ast-grep.github.io/guide/introduction.html
4. ast-grep — "Using ast-grep with AI Tools"，https://ast-grep.github.io/advanced/prompting.html
5. ast-grep/agent-skill — README，https://github.com/ast-grep/agent-skill
6. Comby — "Overview"，https://comby.dev/docs/overview
7. Tree-sitter — "Introduction"，https://tree-sitter.github.io/tree-sitter/
8. OpenRewrite — "Introduction"，https://docs.openrewrite.org/
9. OpenRewrite — "Lossless Semantic Trees (LST)"，https://docs.openrewrite.org/concepts-and-explanations/lossless-semantic-trees
10. facebook/jscodeshift — README，https://github.com/facebook/jscodeshift
11. Anthropic — "Tool use with Claude"，https://docs.anthropic.com/en/docs/build-with-claude/tool-use
12. Anthropic — "Text editor tool"，https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool
13. Anthropic — "Strict tool use"，https://platform.claude.com/docs/en/agents-and-tools/tool-use/strict-tool-use
14. OpenAI — "Structured model outputs"，https://platform.openai.com/docs/guides/structured-outputs
15. OpenAI — "Function calling"，https://platform.openai.com/docs/guides/function-calling
16. Model Context Protocol — "Tools"，https://modelcontextprotocol.io/specification/2025-06-18/server/tools
17. Cursor — "Cursor Documentation"，https://cursor.com/docs
18. OpenAI — openai/codex README，https://github.com/openai/codex
19. JetBrains — "MPS: The Domain-Specific Language Creator"，https://www.jetbrains.com/mps/

研究论文（均访问于 2026-09-13）：

20. Allamanis et al., "Learning to Represent Programs with Graphs"，arXiv:1711.00740，https://arxiv.org/abs/1711.00740
21. Alon et al., "code2vec: Learning Distributed Representations of Code"，arXiv:1803.09473，https://arxiv.org/abs/1803.09473
22. Bavarian et al., "Efficient Training of Language Models to Fill in the Middle"，arXiv:2207.14255，https://arxiv.org/abs/2207.14255
23. Cassano et al., "Can It Edit? Evaluating the Ability of Large Language Models to Follow Code Editing Instructions"，arXiv:2312.12450，https://arxiv.org/abs/2312.12450
24. Chen et al., "Tree-to-tree Neural Networks for Program Translation"，arXiv:1802.03691，https://arxiv.org/abs/1802.03691
25. Jiang et al., "CURE: Code-Aware Neural Machine Translation for Automatic Program Repair"，arXiv:2103.00073，https://arxiv.org/abs/2103.00073
26. Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?"，arXiv:2310.06770，https://arxiv.org/abs/2310.06770
27. Poesia et al., "Synchromesh: Reliable code generation from pre-trained language models"，arXiv:2201.11227，https://arxiv.org/abs/2201.11227
28. Scholak et al., "PICARD: Parsing Incrementally for Constrained Auto-Regressive Decoding from Language Models"，arXiv:2109.05093，https://arxiv.org/abs/2109.05093
29. Tipirneni et al., "StructCoder: Structure-Aware Transformer for Code Generation"，arXiv:2206.05239，https://arxiv.org/abs/2206.05239
30. Wang et al. (Bailin), "Grammar Prompting for Domain-Specific Language Generation with Large Language Models"，arXiv:2305.19234，https://arxiv.org/abs/2305.19234
31. Wang et al. (Xingyao), "OpenHands: An Open Platform for AI Software Developers as Generalist Agents"，arXiv:2407.16741，https://arxiv.org/abs/2407.16741
32. Wang et al. (Yue), "CodeT5: Identifier-aware Unified Pre-trained Encoder-Decoder Models…"，arXiv:2109.00859，https://arxiv.org/abs/2109.00859
33. Willard & Louf, "Efficient Guided Generation for Large Language Models"，arXiv:2307.09702，https://arxiv.org/abs/2307.09702
34. Yang et al., "SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering"，arXiv:2405.15793，https://arxiv.org/abs/2405.15793
35. Yin et al., "Learning to Represent Edits"，arXiv:1810.13337，https://arxiv.org/abs/1810.13337
36. Zhang et al., "CoditT5: Pretraining for Source Code and Natural Language Editing"，arXiv:2208.05446，https://arxiv.org/abs/2208.05446
37. Agrawal et al., "Guiding Language Models of Code with Global Context using Monitors"，arXiv:2306.10763，https://arxiv.org/abs/2306.10763

**引用的一手来源总数：37**（19 个产品/API/仓库来源 + 18 篇论文）。
