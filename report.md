# MemEvolve 记忆系统探索报告

## 1. 实验设置

本实验使用 DeepSeek-V4-Flash 作为搜索 Agent 与记忆系统分析/生成模型，在 xBench DeepSearch 任务上比较 No-Memory、ExpeL、Agent-KB 与 MemEvolve-evolved 设置。xBench 使用第 81-100 题共 20 条作为评测集，所有系统均使用 `max_steps=6`。记忆系统通过 Flash-Searcher 的 `memory_provider` 接口接入：在规划前调用 `provide_memory(BEGIN)`，把返回内容作为 `Memory System Guidance` 注入 planning prompt；每个执行 step 前调用 `provide_memory(IN)`，如果 provider 支持 IN 阶段则继续注入 execution guidance；任务结束后若未关闭 memory evolution，则由 `take_in_memory()` 接收轨迹、答案和 `is_correct` 元数据，用于更新本地 memory。

为了更清楚地比较四个 memory system，这里先按代码中的 memory 组织形式展开，而不是按最终效果描述。

| System | 代码中的 memory unit | 本地存储 | 检索索引 | 有效注入阶段 | 更新策略 |
|---|---|---|---|---|---|
| Agent-KB | `WorkflowInstance` | `storage/agent_kb/agent_kb_database.json` | 只对 `query` 建 TF-IDF + embedding 索引 | BEGIN | 只吸收成功轨迹 |
| ExpeL | `Insight` + `SuccessTrajectory` | `storage/expel/insights.json`，`storage/expel/success_trajectories.json` | insight 用 TF-IDF；success trajectory 用 text + semantic | 主要 BEGIN | 成功/失败都抽 insight，成功额外存 trajectory |
| SynapseFlow | `SynapseInstance` | `storage/synapse_flow/knowledge_base.json` | `query/task_specific/domain_level/general_level` 多字段 embedding | BEGIN + IN | 只吸收成功轨迹，merge + utility pruning |
| Cognition-Loom | `ExperienceNode` + semantic edge | `storage/cognition_loom_memory/graph.json` | `query/planning/experience` 的 TF-IDF + embedding graph 检索 | BEGIN + IN | 成功和失败都吸收，失败额外存 counter-strategy |


## 2. 主要结果

### xBench 81-100

| System | Accuracy | Cost | Time | Notes |
|---|---:|---:|---:|---|
| No-Memory | 14/20 = 70.0% | 1,807,654 tokens | 172.69s/query | 错题：84, 86, 87, 94, 97, 99。 |
| ExpeL | 16/20 = 80.0% | 3,312,656 tokens | 160.79s/query | 错题：85, 86, 94, 97。 |
| Agent-KB | 14/20 = 70.0% | 2,452,564 tokens | 204.75s/query | 错题：84, 86, 87, 94, 97, 99。 |
| MemEvolve-evolved (`synapse_flow`) | 14/20 = 70.0% | 2,431,630 tokens | 350.18s/query | 基于 1-10 条 seed 轨迹 evolve 得到；错题：84, 86, 87, 94, 97, 99。 |
| MemEvolve-evolved (`cognition_loom_memory`) | 17/20 = 85.0% | 9,361,102 tokens | 751.86s/query | 基于 1-20 条 seed 轨迹 evolve 得到；错题：94, 97, 99。 |

（注：llm judge时将两个memevolve的结果判为93错，实际检查发现93回答正确，因此在错题集中将93删除，report.txt中未重判。）

本实验最核心的发现有三点：

1. evolve数据量对MemEvolve比较重要——基于10条seed的synapse_flow没有超过No-Memory，扩充到20条后cognition_loom_memory才超过ExpeL，论文中的evolve数据量为60，我因为时间和成本限制改为了10、20，且只进化一轮，可能因此大大减少了mem系统的进化程度；
2. 更复杂的memory不等于更高性价比，cognition_loom_memory达到17/20，但token成本是ExpeL的2.8倍、是No-Memory的5.2倍，成本爆炸；
3. memory不一定有效，agent_kb相较于no-memory没有提升，错误题目完全一致，但token消耗和时延变长，


从错题集合可以看到，两轮 MemEvolve 产物表现差异明显：第一轮基于 1-10 条 seed 数据得到的 `synapse_flow` 错题为 84, 86, 87, 94, 97, 99，与 No-Memory 持平；在补充 11-20 条数据后，基于 1-20 条轨迹 evolve 得到的 `cognition_loom_memory` 错题变为 94, 97, 99，准确率提升到 17/20，高于 Agent-KB、ExpeL 和 No-Memory，但付出约 3.82 倍 Agent-KB token 成本。



## 3. Case 分析 （有memory相较于no-memory对比）

### 3.1 成功 Case 分析（存在无memory失败，加入memory成功完成的系统）

#### Case 84：文学作品与电影幕后细节

题目要求从福楼拜《包法利夫人》进一步定位 1991 年电影中服毒道具的味道，标准答案是“苦杏仁味”。难点不是前半段实体识别，而是最后的细粒度限定词：只答“杏仁味”会错。

`ExpeL` 做对，主要收益来自相似 episodic memory：虽然历史轨迹没有直接给出“苦杏仁味”，但帮助快速定位福楼拜、《包法利夫人》和相关搜索路径。`Cognition-Loom` 也做对，依靠多轮 phase-aware guidance 反复提醒检查 “almond / bitter almond” 的差异，最终保留了正确限定词。相反，`Agent-KB` 的 BEGIN guidance 方向正确，但没有 IN 阶段纠偏，最终没输出答案；`SynapseFlow` 更典型，它多次把候选压缩成“almond flavor / 杏仁味”，反而把模型锚定到近似错误答案。

该 case 说明文学影视幕后细节题适合 episodic + procedural memory，但 memory 必须保留答案粒度。方向正确但不够精确的 memory 只能缩小搜索空间，不能保证最终答案正确；不精确的 memory 甚至会强化错误近义词。

#### Case 86：食物文化线索到国家最高峰

题目需要从“美国南方文学食物 + 特定食谱变体”桥接到国家，再查该国最高峰海拔。标准答案是日本富士山 3776 米。该题难点在中间桥接：食物描述和国家之间没有直接实体名。

`No-Memory`、`Agent-KB`、`ExpeL` 都没有输出答案，说明没有相关记忆召回时，Agent 很难完成隐含实体识别。`SynapseFlow` 提供了有用搜索方向，如 `digestive biscuit cornflake crust cream cheese`，但候选验证不足，最终错误锚定到南非并答 3450。`Cognition-Loom` 做对，原因是它在多轮 IN 阶段持续要求用食谱特征缩小候选国家，并在国家确认后再查最高峰。

该 case 说明 semantic memory 对“表面关键词不同但概念相关”的任务有价值，procedural memory 则负责约束“先定国家，再查山峰”的顺序。但 memory 必须带有候选排除机制，否则会像 SynapseFlow 一样把搜索方向转化为错误锚定。

#### Case 87：数值单位/答案口径

题目最终问 OpenAI Five 使用多少个 `GPU 集群 × 10 个月`，标准答案按 benchmark 口径为 256。多数系统都能找到 “a cluster of 256 GPUs over 10 months” 这个事实，真正难点是单位和答案口径。

`No-Memory`、`Agent-KB`、`SynapseFlow` 都答 1，把 “a cluster of 256 GPUs” 理解成“1 个集群”。其中 `Agent-KB` 虽然提醒过区分 GPU 数和 cluster count，但只在 BEGIN 阶段出现，最终未能防止误读；`SynapseFlow` 更明显是负面 memory，guidance 直接写成“1 个 GPU 集群（含 256 块 GPU）”，把错误解释提前固化。`ExpeL` 和 `Cognition-Loom` 答 256，收益来自更强的“保留关键数字、按题目格式输出”的 procedural guidance。

该 case 说明 memory 对“事实已找到但最终抽取容易错”的任务很有价值。有效 memory 不一定是提供更多搜索词，而是约束答案格式、单位解释和最终数值口径。

#### Case 99：《乐队的夏天》女性成员统计

题目要求枚举《乐队的夏天》各季 top5 乐队并统计女性成员，标准答案是 6。难点是集合枚举和计数边界：要确定每季 top5、逐队查成员，并处理节目时期阵容和是否重复计入。

`No-Memory`、`Agent-KB`、`SynapseFlow` 都没有完成最终汇总，说明仅靠普通搜索很容易在多实体枚举中耗尽步骤。`ExpeL` 做对，它召回的相似案例并非音乐综艺事实，但迁移了有效流程：拆分子目标、逐项验证、最后合计。`Cognition-Loom` 完成了枚举但答 7，错误来自边界条件控制不足，尤其是第三季女性成员计数口径。

该 case 说明枚举计数题适合 procedural memory，但 procedural memory 必须足够具体：强制输出逐项表格、标注来源、处理重复和节目时期阵容。只推动 Agent “完成枚举”还不够，还要约束计数规则。

### 3.2 失败 Case 分析（有无记忆均失败）

#### Case 94：短视频拍摄地定位

题目要求定位 IShowSpeed 一条截至 2025 年 5 月 18 日播放量过亿的 YouTube 物理挑战短视频，标准答案是“长沙”。难点在于“物理挑战”不是唯一标题，IShowSpeed 有多个类似短视频，搜索结果中很容易混入其他平台或其他地点的挑战。

`No-Memory` 没有稳定输出；`Agent-KB` 的 guidance 虽然提醒先确认视频标题/链接，但后续仍被聚合页面带到“河北邯郸八卦坑”。`ExpeL` 召回的是无关多跳案例，只提供通用搜索策略；`SynapseFlow` 和 `Cognition-Loom` 的 guidance 更贴近任务，但很快接受 “spiral pit / 八卦坑” 候选，并在后续搜索中不断强化该方向。

该 case 说明短视频/社交媒体定位任务需要 metadata/tool-use memory：先锁定唯一 YouTube short URL 或 video ID，再核验播放量截止日期和拍摄地点。单纯文本 memory 只能增加搜索词，无法防止相似视频锚定。

#### Case 97：结构化数据筛选

题目要求统计科创板“注册生效”且“申报历时 <= 180 天”、注册地在沿海地区的公司数量，标准答案 86。这是典型结构化数据筛选任务，需要完整公司列表、状态、日期、注册地和日期差计算。

`No-Memory`、`Agent-KB`、`SynapseFlow` 的主要失败是拿不到完整结构化表格：它们知道要找哪些字段，但 `web_search/crawl_page` 无法稳定导出上交所或东方财富的动态表格。`Agent-KB` 和 `SynapseFlow` 的文字 guidance 方向正确，却没有真正的数据接口和计算能力。`ExpeL` 同样缺少可执行数据处理方案，最后还触发 final-output 格式错误；`Cognition-Loom` 则出现 tool-call 格式生成/解析错误，并产生极高成本。

该 case 说明结构化统计任务不适合只靠 episodic/procedural text memory。真正需要的是 tool-use memory 或可执行 workflow memory：数据源/API、字段映射、分页抓取代码、日期差计算、沿海省份集合和 spot check 规则。

### 3.3 有无 memory 均成功 Case 分析

五个系统均做对的 query 共 13 条：81, 82, 83, 88, 89, 90, 91, 92, 93, 95, 96, 98, 100。它们的共同特点是检索入口清晰、实体线索明确、答案来自少数公开网页，推理链多为“定位实体 -> 查属性/日期/数量 -> 简单确认”。这类题包括实体属性查询、公开网页事实、小规模日期/数量计算，以及少量线性文化娱乐多跳题。

| System | Avg Tokens | Avg Time | Avg API Calls | 相对 No-Memory |
|---|---:|---:|---:|---|
| No-Memory | 69,106 | 139.44s | 9.54 | 基线 |
| Agent-KB | 103,297 | 167.37s | 14.38 | tokens +49.5%，time +20.0% |
| ExpeL | 141,960 | 131.29s | 13.23 | tokens +105.4%，time -5.8% |
| SynapseFlow | 89,393 | 240.76s | 25.15 | tokens +29.4%，time +72.7% |
| Cognition-Loom | 131,359 | 297.46s | 25.62 | tokens +90.1%，time +113.3% |

这些 case 说明，简单事实检索或线性多跳题不一定需要 memory。memory 在这里更多只是增加搜索模板和交叉验证提醒，但准确率没有提升，token 和 API call 通常上升。更合理的策略是 memory gating：题面线索清楚时使用 No-Memory 或 very-light memory；只有在检索失败、候选歧义、单位口径不清或需要集合统计时，再启用更重的 memory。

### 3.4 memory 加入导致失败 Case 分析

#### Case 85：交通性价比判断

题目要求比较从 UPenn 到 CMU 的汽车和火车性价比，标准答案是“汽车”。实体链路并不难：Taylor Swift -> 宾夕法尼亚 -> 费城 UPenn / 匹兹堡 CMU；真正判断点是“时间和经济成本综合考虑”。

`No-Memory`、`Agent-KB`、`SynapseFlow`、`Cognition-Loom` 都答对，说明这类线性事实 + 简单权衡任务不需要重 memory。`ExpeL` 答“火车”，负收益来自不相关 episodic memory：召回案例只提供通用拆分和交叉验证策略，没有约束交通性价比的权衡口径，导致它过度重视票面费用差异，低估汽车显著节省时间的价值。

该 case 说明 memory 并不总是有益。对于实体链路清楚、判断规则简单的任务，No-Memory 或 very-light procedural memory 更合适；长 episodic memory 可能引入噪声和错误权重。

## 4. 不同记忆类型适合什么任务

| Memory Type | 适合任务 | 本实验中的证据 | 主要风险 |
|---|---|---|---|
| Episodic memory | 与历史成功 case 相似的多跳检索、实体链路迁移、节目/作品/人物类搜索 | ExpeL 在 Case 84、87、99 上有收益，尤其 Case 99 用相似成功轨迹推动“枚举 -> 查成员 -> 合计”流程 | 相似度不够时会引入长噪声；不能保证答案粒度和结构化计算 |
| Semantic memory | 跨领域概念桥接、别名/同义词、隐含实体识别、候选空间收缩 | Case 86 需要从食物文化线索桥接到国家变体；Case 84 需要区分 almond/bitter almond | 抽象过度会丢失限定词和边界条件，SynapseFlow 在 Case 84、86 被错误候选锚定 |
| Procedural memory | 搜索流程、任务分解、查询模板、答案口径校验、小规模枚举计数 | Agent-KB 在多题给出拆解模板；Cognition-Loom 在 Case 87 通过阶段化提示纠正数值口径；Case 99 需要逐项表格式计数流程 | 如果只在 BEGIN 注入、缺少 IN 阶段纠偏，容易知道该怎么做但执行不到位；过度注入会增加成本 |
| Tool-use memory | 表格抓取、API/CSV 使用、日期差计算、大规模筛选、可复现统计 | Case 97 所有系统失败，说明文本 guidance 无法替代真实的数据处理 workflow | 需要和工具/代码执行绑定，否则只会停留在“应该抓表计算”的文字建议 |

从 xBench 结果看，memory 是否有效不取决于“有没有记忆”，而取决于记忆形态是否匹配任务瓶颈。这里把本实验中的系统抽象成四类：ExpeL 更接近 episodic memory，Agent-KB 更接近 procedural memory，SynapseFlow/Cognition-Loom 是 semantic + procedural + phase-aware guidance 的混合体；tool-use memory 在当前几个系统里没有被充分实现，但 Case 97 明确暴露了它的必要性。

在xBench 81-100任务测试上，没有单一最优形态：episodic memory综合性价比最好，tool-use memory是当前最大缺口，procedural memory有效但需要IN阶段支撑，semantic memory风险最高。

**Episodic memory** 
适合“当前任务和过去某个成功任务结构相似”的场景。ExpeL 的 memory 内容通常是 `Similar successful case`，本质上不是直接给答案，而是复用历史轨迹中的搜索策略。Case 84 中，它没有直接存储“苦杏仁味”，但相似的《包法利夫人》任务帮助它快速定位作品和人物；Case 99 中，它检索到的案例并不是《乐队的夏天》，但“拆分多跳线索、逐项验证、最后合计”的历史策略迁移有效。因此 episodic memory 适合文学影视、人物作品、节目排名、公司投资者等可迁移搜索链任务。它不适合 Case 97 这类金融表格统计，因为相似轨迹只能提醒“先拿列表再筛选”，不能真正完成数据抽取和计算。

**Semantic memory** 
适合解决“关键词表面不相同，但概念上相关”的任务。比如 Case 86 的关键不是查最高山，而是把“美国南方文学食物 + 酸奶/奶油奶酪 + 消化饼干/玉米片饼底”映射到正确国家；Case 84 也需要把砒霜、almond、bitter almond 和电影道具细节联系起来。这类任务需要保存实体别名、概念关系、候选约束和容易混淆的相近表达。问题在于 semantic memory 很容易在抽象时丢掉精确限定：SynapseFlow 在 Case 84 多次把候选压缩成“杏仁味”，丢掉“苦”；在 Case 86 又把食谱特征错误锚定到南非，输出 3450。因此 semantic memory 适合隐含实体和跨域桥接，但必须保留来源、限定词和反例候选。

**Procedural memory** 
适合“方法比事实更重要”的任务。它记录的是怎么拆题、怎么搜索、怎么验证、最后怎么按题目口径输出。Case 87 是典型例子：事实“256 GPUs over 10 months”并不难搜，难点是题目问 `GPU 集群 × 10 个月` 时应该按 benchmark 口径保留 256；Cognition-Loom 的阶段化 procedural guidance 能持续约束答案格式，所以做对。Case 99 也需要 procedural memory，但应该更具体：先列三季 top5，再按节目时期阵容列女性成员，最后检查重复和口径。Agent-KB 的问题是很多 guidance 只在 BEGIN 阶段出现，后续执行阶段如果偏航就无法纠正；Cognition-Loom 能在 IN 阶段纠偏，但代价是 token 和时延过高。

**Tool-use memory** 
最适合结构化数据任务，尤其是表格筛选、日期计算、批量枚举和可复现统计。Case 97 所有系统都失败，说明“去上交所/东方财富查注册生效公司、计算申报历时、筛沿海省份”这种文字建议不够。真正有效的 memory 应该保存可执行 workflow，例如数据源 URL/API、字段名映射、分页抓取方式、日期差计算代码、沿海省份集合、抽样复核规则。Case 98 这类小规模列车数查询可以靠网页搜索完成，但 Case 97 这种大规模金融筛选必须依赖 tool-use memory，否则只会消耗大量 tokens 而没有稳定结果。

综合来看，短链事实题和五个系统均成功的 13 个 case 不需要重 memory；此时 No-Memory 已经足够，memory 主要增加成本。文学影视幕后细节、隐含实体、多跳检索适合 episodic + semantic + procedural 的组合；单位/答案口径敏感题适合 procedural memory 的 IN 阶段纠偏；大规模统计题必须引入 tool-use memory。后续如果继续改 MemEvolve，我会优先做 memory gating 和 memory type routing：先判断任务瓶颈，再决定是否注入 episodic、semantic、procedural 或 tool-use memory，而不是每一步都注入长 guidance。


## 5. memevolve与harness自进化关系
### 相同点
两者都是对 agent 系统某个模块的自动化架构进化：不依赖人工重新设计，而是基于运行反馈，自动发现当前结构的缺陷并生成改进版本。本质上都是把一个模块分解为若干可替换的组件，通过对组件实现的修改和重组，迭代出性能更好的整体。
### 不同点
两者进化的对象不同，作用在系统的不同层级：
MemEvolve 的 meta-evolution 进化的是记忆模块，将 memory system 分解为 encode / store / retrieve / manage 四个组件，自动修改"经验如何被提炼、存储、召回和维护"，解决的是 agent 学什么、怎么学 的问题。记忆架构本身是进化对象，agent 的行动逻辑不变。
Harness 自进化进化的是 agent 运行框架本身，包括 prompt 模板、工具调用策略、多 agent 协作方式、循环终止条件等，解决的是 agent 怎么行动、怎么决策 的问题。框架结构是进化对象，记忆模块不变。

两者互补，分别作用于系统的不同层次：meta-evolution 决定"用什么方式积累和利用经验"，harness 自进化决定"如何基于经验采取行动"。MemEvolve 优化的是harness中的记忆一个板块。

## 6. MemEvolve 的局限与改进

通过 xBench 81-100 的实验，我认为 MemEvolve 当前最大的局限并不在于 memory retrieval 能力不足，而在于缺乏对 memory 机制收益的归因，以及缺乏对 memory 成本的约束。从实验结果看，MemEvolve 确实能够进化出更复杂的 memory system，但尚未证明其能够进化出更高效的 memory system。相比于继续增加 memory 复杂度，我认为下一步更重要的是让系统学会判断“什么 memory 有效”“什么时候应该使用 memory”。


### Limitation 1：MemEvolve知道如何增加Memory，但不知道哪些Memory真正有效

从 Agent-KB → SynapseFlow → Cognition-Loom 的演化过程中，可以观察到 memory 结构不断变复杂：从 workflow retrieval，到多层抽象 memory，再到 graph memory、failure memory 和 phase-aware guidance。然而 MemEvolve 的评估信号主要是最终 accuracy 和 token cost，因此系统只能知道某个 memory system 整体表现更好，却无法知道具体是 graph、failure learning、IN-stage guidance 还是其他机制带来了收益。

例如 cognition_loom_memory 在本实验中达到 17/20，但其内部同时包含 graph retrieval、多字段索引、失败经验学习、阶段化 guidance 等多个创新机制。当前框架无法回答究竟是哪一个机制贡献了准确率提升，也无法判断哪些机制只是增加了 token 消耗。

因此，MemEvolve 当前更像是在搜索“复杂 memory system”，而不是搜索“有效 memory mechanism”。



### Limitation 2：进化过程倾向于复杂化，而非高性价比,tokens爆炸

实验结果显示：

No-Memory: 70.0%, 1.81M tokens
ExpeL: 80.0%, 3.31M tokens
Cognition-Loom: 85.0%, 9.36M tokens

虽然 Cognition-Loom 获得了最高准确率，但 token 消耗达到 No-Memory 的 5.2 倍、ExpeL 的 2.8 倍。我在实验中只做了一轮evolve、对20条数据测试就花费了近¥10。cognition_loom_memory在20道题上消耗9.36M tokens，单题平均751秒。这个成本在学术环境下基本不可持续，更不用说论文中建议的多轮Pareto筛选。这是MemEvolve目前最实际的障碍，不是方法问题，是工程成本问题。

进一步分析发现，Cognition-Loom 在 20 个任务中共注入 186 次 memory guidance，而 Agent-KB 仅注入 20 次。也就是说，当前进化过程倾向于通过增加 memory 数量、memory 长度和 memory 复杂度来获得性能提升，而不是提高 memory 的利用效率。

因此，我认为当前 MemEvolve 的一个核心问题是：它学会了增加 memory，却没有学会控制 memory。

### Limitation 3：Memory System 缺少质量诊断与可观测性

当前 MemEvolve 的 validator 主要关注 memory provider 是否能够被正确导入、初始化以及调用接口，而对 memory system 的实际行为缺乏深入检查。

在实验过程中，我发现即使一个 memory system 能够通过 validator，它仍然可能存在以下问题：

只实现 BEGIN 阶段 memory，而缺少 IN 阶段支持；
take_in_memory() 返回成功但没有真正写入 storage；
无法学习失败轨迹；
返回过长 guidance 导致 token 开销异常；
memory 内容为空或 metadata 不完整。

这些问题不会导致系统直接崩溃，因此容易通过现有验证流程，但会影响 memory system 的实际质量和可维护性。

因此，当前 MemEvolve 更缺少的是对 memory 行为的验证能力，而不是单纯的 memory retrieval 能力。

### 我的 Patch：validator 诊断增强

当前已实现的小 patch 位于：

```text
Flash-Searcher-main/MemEvolve/phases/phase_validator_update.py
```
我实际实现的 patch 是 validator 诊断增强，其作用主要是提升 memory system 的可观测性和可调试性，而非直接提高 accuracy。
Patch 的核心思路是：validator 不只检查 provider 是否能 import、init、调用接口，还要检查 memory sys 是否具备更完整的行为闭环。

主要修改包括：

```python
# 1. 同时检查 BEGIN 和 IN 阶段的 provide_memory()
for phase_name, phase_status in [
    ("begin", MemoryStatus.BEGIN),
    ("in", MemoryStatus.IN),
]:
    response = provider.provide_memory(test_request)

# 2. 汇总 memory 输出规模和内容预览
{
    "count": len(memories),
    "total_content_chars": total_chars,
    "max_content_chars": max_chars,
    "metadata_keys": ...
}

# 3. 对超长 guidance 给出 warning
if phase_summary["max_content_chars"] > 4000:
    diagnostic_warnings.append("guidance is very long")

# 4. 同时测试成功轨迹和失败轨迹的 take_in_memory()
failed_trajectory = TrajectoryData(
    query="test query for failed memory ingestion simulation",
    metadata={"is_correct": False, "status": "failed"}
)

# 5. 对 storage 前后做 snapshot/diff，检查是否真的持久化
storage_changes = self._diff_storage_snapshots(storage_before, storage_after)
```

该 patch 的改善方向是“质量门禁和诊断增强”，可以更早发现以下问题：

- memory sys 只支持 BEGIN，不支持 IN；
- `take_in_memory()` 返回成功但没有真实写入 storage；
- 只吸收成功轨迹，不吸收失败轨迹；
- 每次返回过长 memory，存在 token 成本风险；
- memory 内容为空、metadata 缺失或不可解释。

但这个 patch 不会直接提升 81-100 accuracy。它适合多候选、多轮 evolve 的筛选阶段；如果只生成一个 memory sys 且不基于 warning 触发 repair/regenerate，则它主要是诊断工具。由于多轮evolve token消耗太大，没有进行实验验证效果。

### 未来修改方向：Memory Gating + Mechanism Attribution

基于实验结果，我认为未来更有价值的方向并不是继续生成更复杂的 memory system，而是让系统学会：

1. 哪些机制有效（mechanism attribution）
2. 哪些任务需要 memory（memory gating）
3. 哪些 memory 值得保留（memory efficiency）

例如，本实验中有 13/20 个任务所有系统均成功完成，此时 memory 的主要作用只是增加 token 成本。因此 memory 应该首先判断任务是否真正需要记忆，再决定是否启用 episodic、semantic、procedural 或 tool-use memory。

我认为这比继续增加 graph、routing 或 memory hierarchy 更符合实际 agent 系统对于成本和性能平衡的需求。


## 7. 运行命令

以下命令在 `Flash-Searcher-main` 目录执行。

### xBench No-Memory 81-100

```bash
python run_flash_searcher_mm_xbench.py \
  --infile data/xbench/DeepSearch.csv \
  --outfile xbench_output/eval_no_memory_81_100.jsonl \
  --task_indices 81-100 \
  --concurrency 1 \
  --max_steps 6 \
  --direct_output_dir xbench_output/eval_no_memory_81_100 \
  --judge_model deepseek/deepseek-v4-flash \
  2>&1 | tee ./run_logs/eval_no_memory_81_100.log
```

### xBench ExpeL 81-100

```bash
python run_flash_searcher_mm_xbench.py \
    --infile ./data/xbench/DeepSearch.csv \
    --outfile ./xbench_output/eval_expel_cold_81_100.jsonl \
    --task_indices 81-100 \
    --memory_provider expel \
    --concurrency 1 \
    --max_steps 6 \
    --direct_output_dir ./xbench_output/eval_expel_cold_81_100 \
    --judge_model deepseek/deepseek-v4-flash \
    2>&1 | tee ./run_logs/eval_expel_cold_81_100.log
```


### xBench agent_kb 81-100

```bash 
python run_flash_searcher_mm_xbench.py \
    --infile ./data/xbench/DeepSearch.csv \
    --outfile ./xbench_output/eval_agent_kb_cold_81_100_0618.jsonl \
    --task_indices 81-100 \
    --memory_provider agent_kb \
    --concurrency 1 \
    --max_steps 6 \
    --direct_output_dir ./xbench_output/eval_agent_kb_cold_81_100_0618 \
    --judge_model deepseek/deepseek-v4-flash \
    2>&1 | tee ./run_logs/eval_agent_kb_cold_81_100_0618.log
``` 



### xBench MemEvolve-evolved 81-100

```bash
python run_flash_searcher_mm_xbench.py \
  --infile data/xbench/DeepSearch.csv \
  --outfile xbench_output/eval_cognition_loom_memory_cold_81_100.jsonl \
  --task_indices 81-100 \
  --concurrency 1 \
  --max_steps 6 \
  --memory_provider cognition_loom_memory \
  --direct_output_dir xbench_output/eval_cognition_loom_memory_cold_81_100 \
  --judge_model deepseek/deepseek-v4-flash \
  2>&1 | tee ./run_logs/eval_cognition_loom_memory_cold_81_100.log
```

### xBench MemEvolve-evolved SynapseFlow 81-100

```bash
python run_flash_searcher_mm_xbench.py \
  --infile data/xbench/DeepSearch.csv \
  --outfile xbench_output/eval_synapse_flow_cold_81_100_max6.jsonl \
  --task_indices 81-100 \
  --concurrency 1 \
  --max_steps 6 \
  --memory_provider synapse_flow \
  --direct_output_dir xbench_output/eval_synapse_flow_cold_81_100_max6 \
  --judge_model deepseek/deepseek-v4-flash \
  2>&1 | tee ./run_logs/eval_synapse_flow_cold_81_100_max6.log
```

### 生成 MemEvolve-evolved 系统

```bash
python evolve_cli.py \
    --work-dir ./memevolve_work/xbench_agent_kb_seed_1_20 \
    --model deepseek/deepseek-v4-flash \
    run-all ./xbench_output/memevolve_seed_agent_kb_1_20 \
    --provider agent_kb \
    --creativity 0.5 \
    2>&1 | tee ./memevolve_work/xbench_agent_kb_seed_1_20/run_all.log
```

### 多轮自动 evolve （未完成）

```bash
python evolve_cli.py \
  --work-dir memevolve_work/auto_xbench \
  auto-evolve \
  --dataset xbench \
  --provider agent_kb \
  --num-rounds 1 \
  --num-systems 3 \
  --task-batch-x 20 \
  --top-t 2 \
  --extra-sample-y 5 \
  --creativity 0.5 \
  --use-pareto-selection \
  --yes
```

### 结果汇总

```bash
cat xbench_output/eval_no_memory_81_100/report.txt
cat xbench_output/eval_expel_cold_81_100/report.txt
cat xbench_output/eval_agent_kb_cold_81_100_0618/report.txt
cat xbench_output/eval_cognition_loom_memory_cold_81_100/report.txt
cat xbench_output/eval_synapse_flow_cold_81_100_max6/report.txt
```

## 8. 总结

在xBench 81-100上，我复现并比较了No-Memory、Agent-KB、ExpeL以及两种MemEvolve进化得到的memory system。

结果表明：

(1) memory并非总能提升性能；
(2) episodic memory在准确率与成本之间具有最佳综合性价比；
(3) tool-use memory是当前系统最大的能力缺口；
(4) MemEvolve能够进化出更复杂的memory architecture，但尚未证明其能够进化出更高效的memory mechanism。

基于实验观察，我认为未来工作应更多关注memory gating、mechanism attribution和memory efficiency，达到能在较短的轮次进化出优秀的mem sys进而减少成本，而非继续增加memory复杂度。