---
title: "Prediction in Dynamic SDRAM Controller Policies"
authors: "Ying Xu, Aabhas S. Agarwal, Brian T. Davis"
venue: "SAMOS 2009 (International Workshop on Embedded Computer Systems: Architectures, Modeling, and Simulation), LNCS vol 5657, pp. 128-138"
year: 2009
---

# Prediction in Dynamic SDRAM Controller Policies

## 基本信息
- **作者**：Ying Xu, Aabhas S. Agarwal, Brian T. Davis（Michigan Technological University）
- **发表**：SAMOS 2009 (International Workshop on Embedded Computer Systems), LNCS vol 5657, pp. 128-138

## 一句话总结
> 借鉴 branch predictor 的 two-level 预测结构，动态预测每次 SDRAM 访问应使用 Open Page 还是 Close Page 策略，以降低访问延迟。

## 问题与动机

SDRAM controller 传统上使用静态的 access control policy——要么全局 Open Page (OP)，要么全局 Close Page Autoprecharge (CPA)。OP 在 row hit 时延迟最低（仅需 `tcl`），但遇到 row conflict 时需额外付出 `trp + trcd` 的代价；CPA 则对 row conflict 友好（延迟恒定为 `tcl + trcd`），但浪费了 row hit 的 locality 收益。静态策略无法适应不同 benchmark 甚至同一 benchmark 内部访问模式的变化。

论文的核心动机在于：如果能逐访问地动态选择 OP 或 CPA，就能同时获得 row hit 时 OP 的低延迟和 row conflict 时 CPA 的低延迟。论文构造了一个理论上界 DYN_UPB（oracle，利用未来访问信息做出完美选择），证明了动态策略的潜在收益空间，然后用基于历史的预测器来逼近这个上界。

现有工作的不足：prior work [5] 仅预测 DRAM page 应保持 open 的时长，[9] 虽然同时预测是否 precharge 和下一个目标 page，但评估指标、benchmark 选择和预测器设计均不同。本文聚焦于仅预测"访问后是否 precharge"这一二值决策，并以 execution time 作为最终评估指标。

## 核心方法

### 关键思路

论文的核心 insight 是：SDRAM 的 row hit/conflict 行为具有可预测的 pattern，类似于 branch 的 taken/not-taken 历史。将"保持 row open (OP)"类比为 branch taken，"precharge (CPA)"类比为 branch not-taken，从而可以直接复用 Yeh & Patt 的 two-level adaptive branch predictor 结构。与 branch prediction 不同的关键简化点在于：policy misprediction 不需要 rollback，只是增加延迟，不影响程序正确性。

### 技术细节

**预测器基本结构：**

预测器由两部分组成：

1. **History Register (HR)**：n-bit shift register，记录最近 n 次访问的 row hit（1）或 row conflict（0）行为。
2. **Pattern History Table (PHT)**：2^n 个 entry，每个 entry 是一个 2-bit saturating counter。HR 的值作为 index 索引 PHT，PHT entry 的状态决定预测结果（OP 或 CPA）。

**PHT 更新机制（三种 FSM）：**

- **Non-biased**：标准 2-bit saturating counter，row hit 向 OP 方向递增，row conflict 向 CPA 方向递减。四个状态为 OP → WEAK_OP → WEAK_CP → CP。
- **Biased update (b_upd)**：在 row hit 时直接跳转到 OP 状态（无论当前处于什么状态），而 row conflict 的更新保持正常递减。这是一种向 OP 侧的激进偏置。
- **Biased PHT (b_pht)**：四个状态中有三个状态预测为 OP（仅最底部状态预测 CPA），相当于提高了预测 OP 的门槛放宽。

**预测器组织维度（HR × PHT 的组合）：**

HR 有三种粒度：Global (G)、per-Bank (B)、per-Page (P)。PHT 同样有三种：Global (g)、per-Bank (b)、per-Page (p)。论文模拟了以下组合：

| 类别 | 具体配置 |
|------|---------|
| Static PHT | BSg, PSg（含 non-biased 和 b_int 变体） |
| Dynamic PHT | BAb, PAg, PAb（含 non-biased、b_upd、b_pht 变体） |

命名规则沿用 Yeh & Patt 的记法，例如 `b_upd PAb` 表示：biased update FSM、per-Page HR (Adaptive)、per-Bank PHT。

**更新时序约束：**

一个关键的设计细节是：access N 的 row hit/conflict 行为只有在 access N+1 的地址已知后才能确定（因为需要知道后续访问是否命中同一 row）。因此，access N 的行为只能用来更新 access N-1 所索引的 PHT entry，且 N-1 和 N 必须属于同一 bank。

### 与现有工作的区别

1. **vs. Ma & Chen [5]**：[5] 预测 page 应 open 多久（时间维度），本文预测是否 precharge（二值决策），更简单也更容易用硬件实现。
2. **vs. Stankovic & Milenkovic [9]**：[9] 的"complete predictor"同时预测 precharge 决策和下一个目标 page，预测器设计更复杂。本文限制预测范围仅为 precharge 决策，降低实现复杂度。
3. **vs. Hur & Lin [4] (Adaptive History-Based Memory Schedulers)**：[4] 关注 access scheduling（改变访问顺序），本文明确约束不改变 access schedule，仅改变每次访问后的 row management policy，二者正交且可组合。

## 实验评估

### 实验设置

- **仿真平台**：修改版 SimpleScalar v3.0d，将原有 main memory model 替换为详细的 DDR SDRAM 模型和 SDRAM controller 模型
- **硬件规格配置**：
  - CPU：2.4 GHz，4-way out-of-order
  - L1 Cache：64KB I-cache + 64KB D-cache，2-way，32B cache line
  - L2 Cache：512KB，16-way，64B cache line
  - Main Memory：2GB DDR400 (PC3200)，timing 3-4-4-8，single channel，4 ranks，burst of 8
  - Write Buffer：16 entries，idle 3 cycle 后触发 writeback
- **Workload**：SPEC CPU2000，从 24 个正确执行的 benchmark 中选取 17 个（排除 DRAM 访问量极少的 benchmark，参考 Citron [2] 的方法论）
- **对比 baseline**：OP（静态 Open Page）、CPA（静态 Close Page Autoprecharge）、DYN_UPB（oracle 上界）

### 关键结果

1. **DYN_UPB 的理论收益**：相比 OP 平均提升 3.7%，相比 CPA 平均提升 19%。这划定了在不改变 access schedule 和 data placement 条件下 dynamic policy 的性能天花板。
2. **最佳短 HR 预测器（HR=5）**：b_upd PAb 相比 OP 提升 1.0%，相比 CPA 提升 16%，达到 DYN_UPB over OP 收益的 27%，DYN_UPB over CPA 收益的 89%。
3. **最佳长 HR 预测器（HR=11）**：b_int PSg 相比 OP 提升 1.6%，相比 CPA 提升 17%，达到 DYN_UPB over OP 收益的 44%。
4. **预测精度**：最佳 overall accuracy 为 PAb 达到 91%；OP preferred 访问最佳精度为 b_upd PAb 的 76%（后文 abstract 提到 87%，可能对应更长 HR）；CPA preferred 访问精度普遍较高，最高达 96%。

### 结果分析

**OP prediction accuracy 是性能的主导因素**：论文通过计算 execution time 与 OP/CPA/overall prediction accuracy 的 correlation coefficient 发现，几乎所有 benchmark 中 OP accuracy 与 execution time 呈强负相关。这解释了为何 biased 预测器（牺牲 CPA accuracy 换取 OP accuracy）反而能获得更低的 execution time。直觉上，OP misprediction 的惩罚更大（row conflict 时额外付出 `trp`），而 CPA misprediction 的惩罚较小（仅浪费了一个本可利用的 row hit）。

**HR length 的影响**：更长的 HR 普遍带来更低的 execution time。有趣的是，短 HR 时 dynamic PHT（b_upd）最优，但长 HR 时 static PHT（b_int）反超，这暗示长 HR 提供了足够的 pattern 区分度，使得 static PHT 无需在线学习就能有效覆盖常见 pattern。

**异常 benchmark**：wupwise 和 vpr 是例外——对它们而言 OP accuracy 与 execution time 无强相关性，biased predictor 可能反而有害。

## 审稿人视角

### 优点

1. **问题建模的elegance**：将 row management 决策类比为 branch prediction 是一个自然且有洞察力的映射。二者都是二值决策、都依赖历史 pattern、但 SDRAM 场景下 misprediction 无需 rollback，使得实现更简单。
2. **系统性的设计空间探索**：论文在 HR 组织 (G/B/P) × PHT 组织 (g/b/p) × PHT 更新策略 (static/dynamic) × bias 方式 (none/update/pht) 的多维空间中进行了较全面的探索，并清晰地用统一命名规则组织结果。
3. **correlation 分析有价值**：发现 OP accuracy 是性能主导因素这一 insight 对后续工作有指导意义，也合理地 motivate 了 biased predictor 的设计。

### 不足

1. **实验平台过于陈旧且配置不具代表性**：SimpleScalar + DDR400 的组合在 2009 年已经相当过时。2.4GHz 4-way OoO 处理器配 16-entry RUU 和 8-entry LSQ 极其小，这意味着 memory-level parallelism (MLP) 很低，实际 DRAM traffic pattern 与现代处理器差异巨大。单通道 4 rank 的配置也不具代表性。这些配置选择严重限制了结论的可推广性。

2. **性能收益微弱且 ceiling 很低**：即使 oracle (DYN_UPB) 相比 OP 也仅有 3.7% 的提升，最佳预测器仅 1.0%-1.6%，这本身说明在该实验配置下，dynamic row management 的收益空间极为有限。论文未讨论在更高频率 DRAM（论文仅在结论处一笔带过）或多核场景下收益是否会放大。

3. **缺少硬件开销的定量分析**：论文提到存储开销和延迟分析在硕士论文 [11] 中给出，但本文完全省略。per-Page PHT (P) 的存储开销在实际 DRAM 容量下可能非常大（每个 bank 的每个 row 都需要一个 PHT），论文未讨论这一可行性问题。per-Page HR 同样面临存储爆炸的问题。

4. **缺少与 FR-FCFS 等实际调度策略的对比**：论文的 baseline 仅为纯 OP 和纯 CPA，但实际 memory controller 中广泛使用的 FR-FCFS (First-Ready First-Come-First-Served) 调度策略已经隐含了对 row hit 的优先处理。论文完全未讨论预测器与 FR-FCFS 的交互或对比，这是一个重大的 baseline 缺失。

5. **对 write 的处理过于简化**：write buffer 在 bus idle 3 cycle 后才 writeback，这种简单策略在现代系统中不具代表性。Write-read 切换的 penalty、write drain 策略对 row buffer 状态的影响均未讨论。

6. **SPEC CPU2000 benchmark 的选择**：2009 年发表的论文使用 2000 年的 benchmark suite，SPEC CPU2006 当时已发布三年。且仅使用单线程 workload，未考虑多线程/多程序混合场景下 bank 竞争加剧后 dynamic policy 的表现。

### 疑问或值得追问的点

- per-Page predictor 的存储开销到底有多大？以 2GB DRAM、8KB page size 计算，每个 page 需要至少 `2^n × 2` bit 的 PHT 加上 n bit 的 HR，总存储量是否可接受？
- 为什么 DYN_UPB 的 upper bound 如此之低（仅 3.7% over OP）？这是否意味着在该配置下大部分 DRAM 访问本身就是 row hit（L2 cache 已经过滤掉了大部分 locality 较差的访问），dynamic policy 的发挥空间天然有限？
- 预测器的 latency 是否在 critical path 上？如果预测延迟导致 DRAM command 发射推迟 1-2 cycle，收益是否会被抵消？
- 在现代 DDR4/DDR5 配置（更多 bank/bank group、更长的 timing parameter）下，这种预测方法的收益是否会显著增大？
- biased predictor 在 streaming workload（几乎全是 row conflict）下是否会严重退化？论文仅发现 wupwise 和 vpr 是例外，但未深入分析原因。
