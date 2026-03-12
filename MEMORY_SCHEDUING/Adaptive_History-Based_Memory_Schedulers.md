---
title: "Adaptive History-Based Memory Schedulers"
authors: "Ibrahim Hur, Calvin Lin"
venue: "MICRO 2004 (推测，论文未明确标注会议名，但从内容和引用风格判断为体系结构顶会)"
year: 2004
---

# Adaptive History-Based Memory Schedulers

## 基本信息
- **作者**：Ibrahim Hur (UT Austin ECE / IBM), Calvin Lin (UT Austin CS)
- **发表**：2004年（具体会议未在文中明确标注）
- **平台**：IBM Power5

## 一句话总结
> 利用最近已调度命令的历史序列构建FSM，自适应地同时优化DRAM command latency和Read/Write比例匹配，显著提升内存带宽。

## 问题与动机

论文要解决的核心问题是：**通用处理器（general-purpose processor）的 memory controller 中，如何智能调度发往 DRAM 的 memory command 以最大化带宽利用率**。

在2004年的背景下，这个问题的重要性体现在：

1. **Processor-Memory速度差持续扩大**：CPU频率快速增长，DRAM带宽增长滞后，memory wall问题日趋严重。
2. **现有工作局限于 streaming processor**：Rixner et al. (ISCA 2000) 的经典工作是在 cacheless 的 Imagine stream processor 上做的，缺乏对带有多级cache hierarchy的通用处理器的研究。
3. **现有调度策略的两个根本缺陷**：
   - **短视（greedy）**：已有策略（如避免 bank conflict 的 oldest-first）本质上是贪心的，只看当前cycle能否避免冲突，不考虑当前决策对后续cycle的连锁影响。例如，当前避免了bank conflict，但可能导致下一个cycle产生不可避免的conflict。
   - **忽略应用行为**：只考虑硬件约束（bank/rank/port conflict），不考虑应用本身的 Read/Write 比例特征。以 daxpy kernel 为例，其 Read:Write = 2:1，如果 arbiter 不按此比例调度，Read Queue 或 Write Queue 会饱和，形成瓶颈。

## 核心方法

### 关键思路

核心 insight 有两个：（1）通过维护**最近已发射命令的历史（command history）**，arbiter 可以推理当前调度决策将引入的延迟代价，从而做出更优的非贪心决策；（2）通过跟踪历史中的 Read/Write 比例，arbiter 可以调度命令以**匹配应用的 Read/Write pattern**，避免 reorder queue 饱和导致的瓶颈。这两个目标通过 FSM 编码并概率性组合，再通过运行时自适应选择机制支持多种 workload。

### 技术细节

#### 1. History-Based Arbiter 的 FSM 编码

每个 history-based arbiter 维护一个长度为 $n$ 的命令历史窗口。假设命令类型有 $m$ 种（本文中 $m=4$：Read Port0, Read Port1, Write Port0, Write Port1），则 FSM 共有 $m^n$ 个状态。每个状态编码了最近 $n$ 条命令的类型序列。

在每个状态下，FSM 为所有可能的 next command 定义了一个**优先级排序**。当 arbiter 需要选择下一条命令时，它按优先级顺序检查 reorder queue 中是否有对应类型的命令可用，选择第一个可用的。选择后，FSM 转移到新状态（history 窗口滑动）。

关键设计参数：本文使用 **history length = 2**，4种命令类型，因此每个 FSM 有 $4^2 = 16$ 个状态。

#### 2. 两个优化目标及其算法

**目标一：Command Pattern Matching（Algorithm 1）**

对每个 FSM 状态，计算当前历史的 Read/Write ratio，然后对所有可能的 next command，计算选择该命令后新 ratio 与目标 ratio 的偏差。优先选择使 ratio 更接近目标值的命令。平局时用 expected latency 作为 secondary criterion。

**目标二：Expected Latency Minimization（Algorithm 2）**

定义 $k$ 个 cost function $f_{1..k}(c_x, c_y)$，每个代表一种硬件约束下两条命令之间的 mandatory delay（例如"同 port 不同 rank 的 Read 之间的延迟"、"Read 之后不同 bank 的 Write 的延迟"等）。

关键假设：arbiter **不跟踪历史命令发射后经过的cycle数**，而是假设历史中的命令是**连续发射的（one cycle apart）**。这是一个重要的简化，使得延迟计算可以纯粹基于命令类型序列进行，不需要时间戳。

对每条历史命令 $c_i$ 和候选新命令 $c_{new}$，计算：
$$fcost_i(c_{new}) = \max(f_j(c_i, c_{new})) - (i-1)$$
其中 $(i-1)$ 是 $c_i$ 在 $c_{new}$ 之前已经经过的cycle数。最终延迟：
$$T_{delay}(c_{new}) = \max(fcost_{1..n}(c_{new}))$$

按 $T_{delay}$ 排序优先级，平局时用 command pattern 作为 secondary criterion。

#### 3. 概率性组合（Algorithm 3）

两个目标可能冲突，因此通过一个 threshold 参数概率性地在两个 FSM 之间切换：每隔若干 cycle 生成随机数，若小于 threshold 则使用 pattern arbiter，否则使用 latency arbiter。Threshold 是系统相关的经验参数。

#### 4. Adaptive Selection 机制

单个 history-based arbiter 只为一种特定的 Read/Write ratio 优化。为支持不同 workload，系统同时维护**三个** history-based arbiter：
- **1R2W**：优化 Read:Write = 1:2 的场景
- **1R1W**：优化 Read:Write = 1:1 的场景
- **2R1W**：优化 Read:Write = 2:1 的场景

Memory controller 维护两个计数器 $Rcnt$ 和 $Wcnt$ 跟踪从 processor 收到的 Read/Write 数量，以及一个周期计数器 $Ccnt$。每 $Ccnt$（= 10000 processor cycles）周期，根据 $Rcnt/Wcnt$ 的值选择最合适的 arbiter：
- ratio > 1.2 → 选 2R1W
- 0.8 ≤ ratio ≤ 1.2 → 选 1R1W
- ratio < 0.8 → 选 1R2W

为防止重试命令（retried commands）干扰计数，只统计新命令。

#### 5. 与 Bank Conflict 处理的分工

一个重要的设计选择：**bank conflict 仍然使用 Rixner et al. 的简单避让策略**（选择不与 DRAM 中命令冲突的命令），history-based 方法只处理 **rank conflict 和 port conflict**。原因是 bank conflict 的延迟远大于 rank/port conflict（75ns vs 30ns），要用 history-based 方法有效处理 bank conflict 需要远超2的 history length，状态空间会爆炸。

### 与现有工作的区别

| 维度 | Rixner et al. (ISCA 2000) | In-Order (FIFO) | 本文 Adaptive History-Based |
|------|--------------------------|-----------------|---------------------------|
| 目标平台 | Cacheless stream processor (Imagine) | 通用处理器 | 通用处理器 (IBM Power5，含多级 cache) |
| 调度策略 | 基于当前硬件状态的 greedy reordering | 严格 FIFO | 基于命令历史的 FSM + 自适应选择 |
| 是否考虑历史 | 否（memoryless） | 否 | 是（history length = 2） |
| 是否考虑应用行为 | 否 | 否 | 是（R/W ratio matching） |
| 多目标优化 | 否 | 否 | 是（latency + pattern，概率性组合） |

## 实验评估

### 实验设置

- **仿真平台**：IBM Power5 cycle-accurate simulator（经过与实际硬件 1% 以内精度验证的内部模拟器）。两种模式：(1) 使用 execution trace 模拟单处理器；(2) 使用 microbenchmark 描述模拟双处理器共享 memory controller。
- **处理器配置**：Power5 @ 1.6GHz，双核 SMT；L1D 64KB 4-way，L1I 128KB 2-way，L2 3640KB 10-way（128B line），off-chip L3 36MB；硬件 data prefetcher（L2→L1, Memory→L2）。
- **DRAM 配置**：DDR2-266 @ 266MHz，2 ports，4 ranks/port，4 banks/rank（5D 结构：port-rank-bank-row-column）。Bank conflict = 75ns，rank conflict = 30ns。
- **Memory Controller**：Read Reorder Queue 和 Write Reorder Queue 各 8 entries，CAQ (Central Arbiter Queue) 4 entries FIFO，可跟踪最近 12 条已发射命令。
- **Workload**：
  - **Stream benchmarks**（Copy, Scale, Sum, Triad）：衡量可持续内存带宽
  - **NAS benchmarks**（BT, CG, EP, FT, IS, LU, MG, SP）：科学计算
  - **14 个 microbenchmarks**（从 4R0W 到 0R4W）：覆盖不同 R/W ratio 的数据流模式
- **采样方法**：NAS/Stream 使用 uniform sampling，50 个均匀样本，每个 2M 指令；microbenchmarks 完整模拟。
- **对比 baseline**：
  1. **In-Order**：FIFO 调度（Power5 默认）
  2. **Memoryless**：Rixner et al. 的策略（避免 bank conflict + oldest-first）

### 关键结果

1. **Stream benchmarks**：adaptive history-based 方法相比 in-order 提升执行时间 **65-70%**，相比 memoryless 提升 **18-20%**。
2. **NAS benchmarks（单核）**：相比 in-order 提升 IPC **6.6%-21.4%（几何均值 10.9%）**；相比 memoryless 提升 **2.4%-9.7%（几何均值 5.1%）**。
3. **4倍CPU频率下的 NAS benchmarks**：相比 in-order 几何均值提升 **14.9%**，相比 memoryless 提升 **8.4%**——说明随着 CPU 变快，memory scheduling 的重要性增加。
4. **与 Perfect DRAM 对比**：本文方案在所有 benchmark 上达到了无任何硬件 hazard 的理想 DRAM 性能的 **95-98%**，表明进一步优化空间极为有限。
5. **硬件开销**：adaptive history-based arbiter 使 memory controller 面积增加 2.38%，占整个 Power5 芯片面积的 **0.038%**。

### 结果分析

**瓶颈转移分析**（Section 5.7 是本文最有价值的分析之一）：

论文深入分析了 memory controller 内部四个潜在瓶颈的变化：

1. **Reorder Queue 满（retry）**：history-based 方法相比 in-order 总是减少 retry，但相比 memoryless 有时反而增加——这看似矛盾，但论文解释这是因为 history-based 方法更高效地利用了 DRAM 带宽，导致 CAQ 更容易满，反压到 reorder queue。
2. **Reorder Queue 中所有命令被 bank conflict 阻塞**：history-based 方法大幅减少此类情况（Figure 12），说明调度质量提升。
3. **Reorder Queue 空**：history-based 方法显著减少空队列出现频率（Figure 13），说明队列占用率提高。
4. **CAQ 满**：history-based 方法**大幅增加** CAQ 满的情况（Figure 14，增加 100-500%）。

论文的核心洞察：**好的调度器将瓶颈从 controller 外部（arbiter 无法帮助的地方）转移到了 controller 内部的管线末端（CAQ），而 CAQ 的反压增加了 reorder queue 的占用率，给 arbiter 提供了更大的调度窗口，形成了良性循环。**

验证实验：增大 CAQ 长度后，CAQ 瓶颈减少，但 reorder queue 占用率也下降，整体性能反而下降——证实了上述理论。

**Sensitivity 分析**：
- 关闭 prefetch unit 后收益显著下降（Figure 7 vs Figure 6），因为内存流量降低，调度压力减小。
- 从双核改为单核（关闭 prefetch）后收益进一步下降（Figure 8）。
- Adaptive selection 机制在除 3R2W 外的所有 microbenchmark 上都选择了最优的 history-based arbiter（Figure 9），3R2W 上偏差仅 1%。
- 数据对齐敏感性降低（Figure 15），表明更好的调度可以 mitigate alignment 带来的性能差异。
- 适应周期（$Ccnt$）只要大于约100 cycle，结果对其不敏感。

## 审稿人视角

### 优点

1. **问题定义清晰，motivation 扎实**：从 greedy scheduling 的两个根本缺陷出发，提出 history-based 的思路，逻辑链条非常清楚。daxpy kernel 的 R/W ratio 例子简洁有力地说明了为什么需要考虑应用行为。

2. **实验平台可信度极高**：使用经过 1% 精度验证的 IBM Power5 cycle-accurate 内部模拟器，这在学术论文中罕见。相比使用简化模型的研究，本文结果的工业参考价值很高。

3. **Section 5.7 的瓶颈转移分析极为深刻**：不只是报告性能数字，而是深入 memory controller 内部，分析四种瓶颈的此消彼长，得出"将瓶颈转移到管线末端从而提升 reorder queue 占用率"的洞察。这种分析对后续 memory controller 设计者有很强的指导意义。增大 CAQ 反而降低性能的验证实验也很有说服力。

4. **硬件开销极低**：0.038% 的芯片面积增加和 95-98% 的 perfect DRAM 性能达成率，cost-effectiveness 非常好。这使得方案具有实际部署可行性。

5. **自适应机制简洁有效**：三个 arbiter + 基于 R/W ratio 的选择逻辑，设计简单但效果好，并且论文诚实地展示了在 3R2W 上选错的 case（仅损失 1%）。

### 不足

1. **评估仅限单核（NAS benchmarks）且 workload 覆盖有限**：Power5 是双核 SMT 系统，但 NAS 和 Stream benchmarks 只在单核上运行。论文自己也承认结果是"conservative"的，但未能提供更有代表性的多线程/多核评估。Microbenchmarks 虽然覆盖了不同 R/W ratio，但都是规则的 streaming pattern，缺乏不规则访问模式（如 pointer-chasing、graph 类 workload）的评估。

2. **Bank conflict 处理退化为与 baseline 相同的策略**：论文明确指出 bank conflict（75ns，远大于 rank/port conflict 的 30ns）仍使用 Rixner 的简单避让方法，history-based 只处理 rank/port conflict。换言之，**延迟最大的冲突类型恰恰没有被 history-based 方法覆盖**。论文解释说需要更长的 history length，但未给出任何量化分析或未来方向。这显著削弱了方法的通用性。

3. **History length = 2 的 justification 不够充分**：论文声称 history length = 2 效果"surprisingly good"，但未展示不同 history length（如 1, 3, 4）的对比实验。没有 ablation study 来验证这个关键设计参数的选择。95-98% perfect DRAM 的结果间接说明了问题，但直接对比不同 history length 的数据会更有说服力。

4. **概率性组合的 threshold 参数缺乏系统性分析**：Algorithm 3 中 latency arbiter 和 pattern arbiter 之间的 threshold 被描述为"system dependent, determined experimentally"，但论文未给出具体值，也未展示 threshold 对性能的 sensitivity analysis。

5. **适用性讨论不足**：论文的实验完全基于 Power5 的特定微架构（2 ports, 4 ranks, 4 banks, 特定的 queue 结构）。FSM 的状态编码和 cost function 都是针对此结构设计的。缺乏对方法如何推广到不同 DRAM 组织（如更多 bank、channel、不同 timing 参数）的讨论。

6. **DDR2-266 的时代局限**：从今天的视角看，DDR2-266 的频率极低，bank 数量只有4，timing constraint 相对简单。现代 DDR5 系统有 32+ bank groups、更复杂的 timing 关系（tFAW, tRRD_S/L 等），history length = 2 的 FSM 是否仍然有效值得怀疑。

### 疑问或值得追问的点

- history length = 2 在 Power5 上工作良好，但文中的 cost model 假设历史命令是连续发射的（one cycle apart）。在实际系统中命令间可能有 idle cycle，这个假设引入的误差有多大？论文未给出量化分析。
- Adaptive selection 每 10000 cycle 切换一次，对于 phase behavior 变化更频繁的 workload（如多线程竞争），这个粒度是否太粗？
- 论文中 reorder queue 只有 8 entries，在这种小窗口下 scheduling 的收益是否会被队列大小本身限制？论文提到增大 reorder queue 效果不明显但未详细分析原因。
- Perfect DRAM 作为 upper bound 是否合理？它消除了所有硬件 hazard，但现实中即使调度完美也无法消除所有结构性冲突（如 tFAW 等 DRAM 固有 timing 约束），这个 bound 可能过于乐观。
- 三个 arbiter 的 R/W ratio 设定（1:2, 1:1, 2:1）和切换阈值（0.8, 1.2）是否是最优的？是否需要更多 arbiter 或更细粒度的 ratio 覆盖？论文未做 sensitivity 分析。
