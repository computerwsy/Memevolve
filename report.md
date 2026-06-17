# MemEvolve 记忆系统实验报告

## 1. 实验设置

本实验使用 DeepSeek-V4-Flash 作为搜索 Agent 与记忆系统分析/生成模型，在 xBench DeepSearch 任务上比较 No-Memory、ExpeL、Agent-KB 与 MemEvolve-evolved 设置。xBench 使用第 81-100 题共 20 条作为冷启动评测集；这里的 cold 表示先用前序数据完成 memory/evolve，再在 81-100 上评测，不把 81-100 执行过程中继续产生的 memory 积累作为前置条件。MemEvolve-evolved 包括两条由框架生成/演化出的 memory sys：第一轮先基于 xBench 1-10 条 seed 轨迹 evolve 得到 `synapse_flow`，代码位于 `EvolveLab/providers/synapse_flow_provider.py`；后续在 81-100 冷启动测试中发现 `synapse_flow` 效果很差，甚至低于 No-Memory，因此继续补充 11-20 条 seed 数据，基于 1-20 条轨迹重新 evolve，得到 `cognition_loom_memory`。

## 2. 主要结果

### xBench 81-100

| System | Accuracy | Cost | Notes |
|---|---:|---:|---|
| No-Memory | 14/20 = 70.0% | 1,807,654 tokens | 错题：84, 86, 87, 94, 97, 99。 |
| ExpeL | 16/20 = 80.0% | 3,312,656 tokens | 错题：85, 86, 94, 97。 |
| Agent-KB | 16/20 = 80.0% | 2,427,046 tokens | 错题：87, 94, 97, 99。 |
| MemEvolve-evolved (`synapse_flow`) | 13/20 = 65.0% | 2,431,630 tokens | 基于 1-10 条 seed 轨迹 evolve 得到；错题：84, 86, 87, 93, 94, 97, 99。 |
| MemEvolve-evolved (`cognition_loom_memory`) | 16/20 = 80.0% | 9,361,102 tokens | 基于 1-20 条 seed 轨迹 evolve 得到；错题：93, 94, 97, 99。 |
| Patch版 | 待补评测 | 待补评测 | 当前 patch 为 validator 诊断增强，主要改善候选 memory sys 的筛选/诊断能力，不直接改变任务执行行为；需单独跑 81-100 后补结果。 |

从错题集合可以看到，两轮 MemEvolve 产物表现差异明显：第一轮基于 1-10 条 seed 数据得到的 `synapse_flow` 错题为 84, 86, 87, 93, 94, 97, 99，低于 No-Memory；在补充 11-20 条数据后，基于 1-20 条轨迹 evolve 得到的 `cognition_loom_memory` 错题变为 93, 94, 97, 99，准确率提升到 16/20，但仍没有超过 seed 系统 `agent_kb`，并付出约 3.86 倍 token 成本。



## 3. 成功 Case 分析

### Case 84：文学作品与电影幕后细节

题目要求从“19 世纪法国作家、因伤风败俗引发诉讼的作品”定位到《包法利夫人》，再追踪 1991 年电影版本中女主演服毒自杀戏份使用的道具毒药味道。No-Memory 输出“杏仁味”，与标准答案“苦杏仁味”差一个限定词，判错。

`agent_kb` 和 MemEvolve-evolved 都提供了有用记忆：先把任务拆成文学作品识别、电影版本识别、幕后细节搜索三个子目标，再建议同时使用中文和英文关键词，例如 “Madame Bovary 1991 poison taste” 和 “包法利夫人 1991 服毒 道具 味道”。这类记忆的帮助在于把模糊问题转化为可搜索链条，并提醒 Agent 交叉验证电影版本和演员信息。最终 `agent_kb` 与 `cognition_loom_memory` 都回答“苦杏仁味”。

但该 case 也暴露成本问题：`cognition_loom_memory` 做对了，但注入了 27 次 memory guidance，消耗 1,177,351 tokens；`agent_kb` 只注入 1 次 guidance，消耗 120,861 tokens。

### Case 86：食物文化线索到国家最高峰

题目通过美国南方文学与《油炸绿番茄》引出食物线索，再要求识别一个国家变体，并回答该国最高山海拔。No-Memory 没有给出最终答案。`agent_kb` 的记忆提示将任务拆成三步：确认原食物、根据“酸奶/奶油奶酪、消化饼干混合玉米片饼底”等特征定位国家、查询最高山海拔。更关键的是，后续 IN 阶段 guidance 明确指出该变体指向日本轻乳酪蛋糕，最高峰为富士山，海拔 3776 米。

这个 case 说明结构化任务分解记忆对多跳问题有帮助。有效的不是简单存储答案，而是存储“如何从文化线索转到实体识别，再转到地理事实查询”的流程。MemEvolve-evolved 也做对了，但同样成本很高：1,124,584 tokens、86 次 API calls、26 次 memory guidance。

### Case 87：No-Memory 与 agent_kb 失败，MemEvolve/ExpeL 成功

Case 87 是 MemEvolve-evolved 相比 `agent_kb` 的主要增益点。`agent_kb` 在该题上没有检索到有效 memory guidance，而 MemEvolve-evolved 提供了更主动的阶段化提示：在 BEGIN 阶段给出分解方案，在 IN 阶段根据当前上下文继续提示下一步搜索/验证策略。该现象说明 evolve 后的 phase-aware memory 有潜在价值，尤其是在原始 `agent_kb` 检索为空或只在 BEGIN 阶段工作的情况下。

但是，这个成功并不能说明复杂 memory graph 全面优于原始系统，因为 MemEvolve-evolved 同时在 Case 93 上新增失败，并且整体成本远高于 `agent_kb`。

## 4. 失败 Case 分析

### Case 93：MemEvolve 新增失败

Case 93 在 No-Memory、ExpeL、agent_kb 下均做对，但 MemEvolve-evolved 与 SynapseFlow 做错。这个 case 是最重要的负面证据：更复杂的 memory 注入不一定提升准确率，反而可能引入干扰。MemEvolve-evolved 在该任务中多次注入 memory guidance，容易把 Agent 的注意力从当前任务证据拉向历史策略模板，导致搜索路径或判断标准发生偏移。

该现象说明 memory 的问题不只是“有没有”，还包括“什么时候注入、注入多少、是否与当前证据强相关”。IN 阶段记忆如果没有预算和相关性阈值，可能会造成过度指导。

### Case 97：所有系统均失败，MemEvolve 成本爆炸

Case 97 是“上交所科创板注册生效、申报历时小于等于 180 天、注册地在沿海地区的公司数量”这类数据统计任务。所有系统都失败，说明当前 memory 类型对大规模结构化数据筛选帮助有限。MemEvolve-evolved 在此题没有有效 memory guidance，但仍消耗 4,104,313 tokens 和 192 次 API calls。

这个 case 说明：对于需要爬取官方列表、清洗字段、计算日期差、定义地区集合并统计数量的任务，泛化的经验记忆不够。更适合的 memory 可能是工具型记忆或程序化数据处理模板，例如“如何从表格页抽取字段、如何定义沿海省份集合、如何复核申报历时”。纯文本策略指导很难稳定解决这种任务。

### Case 99：《乐队的夏天》女性成员统计

Case 99 中，ExpeL 做对，MemEvolve-evolved 做错。MemEvolve-evolved 给出的 guidance 基本正确：先确定各季 top5 乐队，再逐队统计女性正式成员，注意跨季重复与成员变动。但最终答案为 7，而标准答案为 6。错误来自具体实体统计阶段，而不是总体规划阶段。

这说明对于“枚举集合并精确计数”的任务，抽象策略记忆只能帮助 Agent 组织流程，不能替代事实核查。过多的通用 guidance 甚至可能让 Agent 对中间名单产生过度自信。更合适的 memory 应该记录可复用的核查约束，例如“计数题必须输出逐项表格，并对每个成员给出来源；跨季重复是否重复计入必须与题目口径一致”。

## 5. 不同记忆类型适合什么任务

从 xBench 结果看，不同 memory 类型的适用场景有明显差异。

No-Memory 适合事实链较短、搜索关键词明显、答案能从少量网页直接得到的问题。它成本最低，但遇到隐含实体、多跳推理和需要搜索策略的问题时容易失败。

ExpeL 适合与历史成功案例相似的任务。它提供的是“相似成功轨迹总结”，对文学、影视、国家识别、节目统计这类可迁移搜索流程有帮助。缺点是它容易返回较长的相似案例，如果相似度不够高，可能带来噪声。

Agent-KB 适合需要任务分解和搜索策略迁移的问题。它在本实验中性价比最好：从 14/20 提升到 16/20，成本只从 1.81M 增至 2.43M。它的问题是主要在 BEGIN 阶段工作，IN 阶段支持弱，失败经验利用不足。

MemEvolve-evolved 的 `cognition_loom_memory` 适合探索 phase-aware、多字段检索、失败记忆、图式组织等更复杂机制。它在 Case 87 上体现出潜力，但当前版本的主要问题是过度复杂和成本失控。它适合多轮 evolve 框架中的候选探索，而不一定适合作为单轮最终系统。

MemEvolve-evolved 的 `synapse_flow` 则是一个反例：它是第一轮基于 1-10 条 seed 轨迹 evolve 出来的系统，同样引入了多层抽象、multi-field embedding retrieval 和 IN 阶段 guidance，但在本实验中只有 13/20。它的问题在于抽象后的 memory 容易丢失具体事实、单位和边界条件，同时失败轨迹利用不足，导致它没有救回 No-Memory 的错题，反而新增 Case 93 失败。这一结果促使后续补充 11-20 条 seed 数据，并基于 1-20 条轨迹重新 evolve 得到 `cognition_loom_memory`。这说明 evolve 生成的 memory sys 需要通过多目标选择和机制级归因筛选，不能只依赖“结构更复杂”。

对于数据统计、表格筛选、金融列表统计等任务，文本记忆的帮助有限，更适合 tool memory 或程序化 workflow memory。记忆不应只告诉 Agent “要交叉验证”，而应提供可执行的数据处理模式。

## 6. MemEvolve 的局限与改进

### Limitation 1：创新缺少机制级归因

当前 MemEvolve 可以生成新的 memory sys，但评估信号主要是最终准确率和 token 成本。对于 `cognition_loom_memory` 这种系统，我们只能看到它最终 16/20、成本 9.36M，却不能自动知道到底是 graph、多字段检索、IN 阶段 guidance、失败记忆中的哪一部分产生了收益或副作用。

因此，下一轮 evolve 的反馈不够精确。框架应该记录 memory 的使用归因，例如每一步注入了什么 memory、来自哪些 source task、字符数多少、后续是否改变搜索方向、最终任务是否正确。

### Limitation 2：生成 prompt 容易鼓励复杂堆叠

生成 prompt 目前强调 innovative、production-grade、复杂度不低于模板、LLM routing、graph memory 等方向。这会让 LLM 倾向于一次性加入多个复杂机制。`cognition_loom_memory` 就是典型结果：它引入了 multi-field retrieval、graph node/edge、BEGIN/IN guidance、failure pattern、dedup 等机制，但整体没有超过 `agent_kb`，成本却大幅上升。

更合理的 evolve 应该是结构化 mutation，而不是自由生成大杂烩。例如每轮生成多个候选：retrieval mutation、phase-routing mutation、failure-learning mutation、memory-compression mutation。这样 selection 才能判断哪个机制真正有效。

### Limitation 3：失败学习信号不足

当前 `take_in_memory()` 收到的 metadata 主要包括 `is_correct`、`status`、`task_id`、`full_query`，但缺少 `golden_answer`、`extracted_answer`、`grader_explanation`。因此 memory sys 即使想学习失败，也只能知道“失败了”，很难知道“为什么失败”。

这会导致失败记忆泛化为笼统建议，例如“需要交叉验证”“需要使用权威来源”，而不是形成具体 counter-strategy，例如“计数任务必须逐项列出成员并检查是否重复计入”。

### Limitation 4：IN 阶段 memory 缺少预算

`cognition_loom_memory` 在 xBench 81-100 中注入 186 次 memory guidance，总 guidance 字符数约 181k；`agent_kb` 只有 23 次、约 24k 字符。IN 阶段本身有价值，但当前缺少全局预算、相关性阈值和复用缓存，导致每个 step 都可能触发新的 memory retrieval/synthesis。

### 我的 Patch：validator 诊断增强

当前已实现的小 patch 位于：

```text
Flash-Searcher-main/MemEvolve/phases/phase_validator.py
```

原始文件备份为：

```text
Flash-Searcher-main/MemEvolve/phases/phase_validator.py.original_6035d56_20260617.bak
```

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

但这个 patch 不会直接提升 81-100 accuracy。它适合多候选、多轮 evolve 的筛选阶段；如果只生成一个 memory sys 且不基于 warning 触发 repair/regenerate，则它主要是诊断工具。

### 更适合后续实验的 Patch 方向

如果要体现 MemEvolve 作为多轮 evolve 服务的优势，我建议后续 patch 不应把生成变保守，而应加入“结构化探索 + 机制级归因”。

推荐 patch：

```text
mutation_strategy 分流生成 + richer fitness summary
```

原理：

1. 每轮生成多个不同 mutation theme 的 memory sys，例如 retrieval_precision、phase_adaptive_memory、failure_aware_learning、memory_compression。
2. 每个候选只重点探索一个机制，避免多个复杂创新同时堆叠。
3. 评测后记录 rescued tasks、lost tasks、tokens_per_correct、memory_guidance_steps、memory_guidance_chars。
4. 下一轮 analyzer 根据这些结构化结果判断哪些机制应保留、压缩或淘汰。

这个方向不会削弱 MemEvolve 的探索性，反而会让探索更可归因、更像真正的进化过程。

## 7. 运行命令（可复现）

以下命令在 `Flash-Searcher-main` 目录执行。

### xBench No-Memory 81-100

```bash
python run_flash_searcher_mm_xbench.py \
  --infile data/xbench/DeepSearch.csv \
  --outfile xbench_output/eval_no_memory_81_100.jsonl \
  --task_indices 81-100 \
  --concurrency 1 \
  --max_steps 6 \
  --direct_output_dir xbench_output/eval_no_memory_81_100
```

### xBench ExpeL 81-100

```bash
python run_flash_searcher_mm_xbench.py \
  --infile data/xbench/DeepSearch.csv \
  --outfile xbench_output/eval_expel_cold_81_100.jsonl \
  --task_indices 81-100 \
  --concurrency 1 \
  --max_steps 6 \
  --memory_provider expel \
  --disable_memory_evolution \
  --direct_output_dir xbench_output/eval_expel_cold_81_100
```


### xBench agent_kb 81-100
```bash 
python run_flash_searcher_mm_xbench.py \
    --infile ./data/xbench/DeepSearch.csv \
    --outfile ./xbench_output/eval_agent_kb_cold_81_100.jsonl \
    --task_indices 81-100 \
    --memory_provider agent_kb \
    --concurrency 1 \
    --max_steps 6 \
    --direct_output_dir ./xbench_output/eval_agent_kb_cold_81_100 \
    --judge_model deepseek/deepseek-v4-flash \
    2>&1 | tee ./run_logs/eval_agent_kb_cold_81_100.log
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
  --disable_memory_evolution \
  --direct_output_dir xbench_output/eval_cognition_loom_memory_cold_81_100
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
  --disable_memory_evolution \
  --direct_output_dir xbench_output/eval_synapse_flow_cold_81_100_max6
```

### 生成 MemEvolve-evolved 系统

```bash
python evolve_cli.py \
  --work-dir memevolve_work/xbench_agent_kb_seed_1_20 \
  run-all \
  --task-logs-dir xbench_output/memevolve_seed_agent_kb_1_20 \
  --provider agent_kb \
  --creativity 0.5
```

### 多轮自动 evolve（建议后续实验）

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
cat xbench_output/eval_cognition_loom_memory_cold_81_100/report.txt
cat xbench_output/eval_synapse_flow_cold_81_100_max6/report.txt
```
