# related-works.md — 结构化编辑 × AI 相关论文

本文件收录偏研究/论文性质的工作，作为 `research.md` 的对偶：后者记录落地的产品/工具/API，本文件记录方法学来源。
所有来源访问于 2026-09-13。

---

## 1. 结构感知的程序表示与生成

- **code2vec**（Alon et al., arXiv:1803.09473）——把代码片段表示为定长向量：把代码"分解为抽象语法树中的一组路径"，联合学习路径表示与聚合方式；在 1400 万个方法上做方法名预测。https://arxiv.org/abs/1803.09473
- **Learning to Represent Programs with Graphs**（Allamanis et al., arXiv:1711.00740）——"用图表示代码的语法和语义结构"，用门控图神经网络，在 VarNaming、VarMisuse 上评估，报告在成熟开源项目中发现了缺陷。https://arxiv.org/abs/1711.00740
- **Tree-to-tree Neural Networks for Program Translation**（Chen et al., arXiv:1802.03691）——把源树翻译为目标树，每一步借助注意力把源子树翻译为目标子树。https://arxiv.org/abs/1802.03691
- **CodeT5**（Wang et al., arXiv:2109.00859）——标识符感知的编码器-解码器预训练模型，带标识符标注目标与 NL-PL 双模态双重生成。https://arxiv.org/abs/2109.00859
- **StructCoder**（Tipirneni et al., arXiv:2206.05239）——通过"源代码的语法树与数据流图"让编码器结构感知，并加入解码器辅助任务"AST 路径预测和数据流预测"；报告在 CodeXGLUE 翻译与文本到代码上达 SOTA。https://arxiv.org/abs/2206.05239
- **中间填充（Fill-in-the-middle, FIM）**（Bavarian et al., arXiv:2207.14255）——把一段文本从中间移到末尾来训练自回归 LM 做填充；作者建议"未来的自回归语言模型默认用 FIM 训练"。https://arxiv.org/abs/2207.14255

## 2. 基于编辑的模型（Edit-based Models）

- **CoditT5**（Zhang et al., arXiv:2208.05446）——提出"显式建模编辑"的预训练目标，用于注释更新、缺陷修复、自动代码评审，并报告用标准生成模型重排后三者均达 SOTA。https://arxiv.org/abs/2208.05446
- **Learning to Represent Edits**（Yin et al., arXiv:1810.13337）——学习"编辑的分布式表示"：把"神经编辑器"与"编辑编码器"结合，使一个编辑可被应用到新的输入上；适用于自然语言与源代码。https://arxiv.org/abs/1810.13337

## 3. 约束 / 语法引导解码

- **PICARD**（Scholak et al., arXiv:2109.05093）——用**增量解析**约束自回归解码，"在每个解码步拒绝不可接受的 token"；把勉强可用的 T5 文本到 SQL 模型变为 Spider/CoSQL 上的 SOTA。https://arxiv.org/abs/2109.05093
- **Synchromesh**（Poesia et al., arXiv:2201.11227）——提出**约束语义解码（CSD）**，"把输出约束到一组合法程序"，在**不重新训练**的情况下强制"语法、作用域、类型规则和上下文逻辑"。https://arxiv.org/abs/2201.11227
- **Grammar Prompting**（Wang et al., arXiv:2305.19234）——把 BNF 语法作为外部领域约束输入；推理时 LLM 先预测一个语法，再按该语法生成输出。https://arxiv.org/abs/2305.19234
- **Outlines / 基于 FSM 的引导生成**（Willard & Louf, arXiv:2307.09702）——把生成"重新表述为有限状态机状态之间的转移"，通过索引模型词表实现正则与上下文无关语法引导，"保证生成文本的结构"。https://arxiv.org/abs/2307.09702
- **Monitor-guided decoding（MGD）**（Agrawal et al., arXiv:2306.10763）——用静态分析作为监视器、结合仓库上下文引导解码，提升编译率与下一标识符匹配率；报告 SantaCoder-1.1B 在这两项指标上可胜过 text-davinci-003。https://arxiv.org/abs/2306.10763

## 4. 静态上下文与语言服务（IDE 集成）

- **Statically Contextualizing Large Language Models with Typed Holes**（Blinn et al., arXiv:2409.00921，OOPSLA 2024）——主张"AI 也需要 IDE"：把 LLM 代码生成接入 Hazel 实时程序草图环境，由 **Hazel Language Server 识别待填充洞（hole）的类型与类型上下文**（即便存在错误），从而用**不在光标附近、甚至不在同一文件**、但与目标语义相关的全库上下文来提示；生成结果再通过与语言服务器的多轮对话迭代精化。作者发布 **MVUBench**（MVU Web 应用基准，挑战点在应用专有数据结构），发现**用类型定义做上下文尤其有效**，并把方法移植到 TypeScript 以验证在资源丰富语言上的适用性。最后提出 **ChatLSP**——LSP 的一个保守扩展，供语言服务器把这类静态上下文暴露给不同设计路线的 AI 补全系统。https://arxiv.org/abs/2409.00921

---

## 5. 程序修复

- **CURE**（Jiang et al., arXiv:2103.00073）——基于 NMT 的自动程序修复，含三部分：PL 模型预训练、"一种新的代码感知搜索策略（专注于可编译补丁以及与缺陷代码长度相近的补丁）"、子词分词；报告修复 57 个 Defects4J 与 26 个 QuixBugs 缺陷。https://arxiv.org/abs/2103.00073

---

## 6. 由这些工作提炼的设计问题（供我们参考）

- **表示层**：AST 路径（code2vec）、程序图（GNN）、树到树（translation）、FIM（局部填充）代表四种"结构如何进入模型"的路线。
- **编辑层**：CoditT5 / Learning to Represent Edits 把"编辑"本身作为可学习对象，对应 Aider 的"编辑格式"思路——只是前者学表示、后者定协议。
- **约束层**：PICARD（语法）、Synchromesh（语义/类型/作用域）、Grammar Prompting（语法作为输入）、Outlines（FSM 保证结构）、MGD（静态分析全局上下文）——约束越强，生成越可用，但需权衡灵活性与开销。
- **上下文层**：typed holes / ChatLSP（类型与绑定结构经语言服务器暴露）说明"把语义局部、跨文件的静态上下文喂给模型"本身是一类独立手段——这正是结构化编辑器的天然能力。
