---
title: "Closed yet Open DRAM: Achieving Low Latency and High Performance in DRAM Memory Systems"
authors: "Lavanya Subramanian, Kaushik Vaidyanathan, Anant Nori, Sreenivas Subramoney, Tanay Karnik, Hong Wang"
venue: "DAC 2018"
year: 2018
---

# Closed yet Open DRAM: Achieving Low Latency and High Performance in DRAM Memory Systems

## 基本信息

- **发表**：DAC (Design Automation Conference), 2018
- **单位**：Intel Labs
- **DOI**：https://doi.org/10.1145/3195970.3196008

## 一句话总结

> 通过 isolation transistor + equalization transistor 的电路改动实现 sense amplifier 与 bitline 解耦，使 read 与 precharge 并行执行，同时降低 tRCD 25%。

## 问题与动机

DRAM bank 内部的 activate → read/write → precharge 操作是串行的，这导致了严重的 latency 瓶颈。Memory controller 面临经典的 open vs. close row 两难抉择：

- **Open row policy**：最大化 row hit，但 row miss 时需要串行等待 precharge 完成才能 activate 新行，miss penalty 很高。
- **Close row policy**：最小化下一次 activate 的等待，但牺牲了 row locality，降低 row hit rate。
- **Adaptive timeout**（如 [2][4]）：依赖历史信息预测 timeout，当行为偏离历史时容易预测错误，且 hill climbing 策略容易陷入 local minima。

最直接的 prior work 是 **Row Buffer Decoupling (RBD)** [ISCA 2014]，提出在 activate + restore 完成后将 sense amplifier 与 bitline 解耦，使 read 与 precharge 并行。但 RBD 存在三个关键问题：

1. **Activate latency 增加**：sense amplifier 内部节点需要在每次 activate 前单独做 precharge，增加了 tRCD——而 tRCD 恰恰是对系统性能影响最大的 timing parameter。
2. **Write 处理复杂且低效**：write 需要将数据 restore 回 bitcell，不能在 precharge 期间进行。RBD 的 activate+precharge 命令导致 write 要么等 precharge 完成后再 restore（额外延迟），要么需要 delay/preempt precharge（DRAM 内部逻辑复杂度大增）。
3. **Sense amplifier idle power 增加**：sense amplifier 一直持有数据直到下一次 activate，功耗接近 open row policy。

## 核心方法

### 关键思路

本文的核心 insight 是：通过两个精巧的电路级改动（equalization transistor + isolation transistor sizing），不仅避免了 RBD 引入的 activate latency 增加问题，反而将 tRCD **降低了 25%**。结合 memory controller 侧对 write 的差异化处理和 power-aware row management，实现了"既 open 又 close"的效果——sense amplifier 继续服务 read（open 的好处），bitline 同时 precharge 为下一次 activate 做准备（close 的好处）。

### 技术细节

SRP（Simultaneous Read and Precharge）由三个互补的组件构成：

#### 1. 电路改动：Activation Latency 降低

**问题根源**：RBD 在 sense amplifier 和 bitline 之间加了 isolation transistor 来解耦。但解耦后 sense amplifier 内部节点（bitline1sa / bitline1sa#）在下次 activate 前需要单独 precharge 到 Vdd/2。RBD 把这步放在 activate 操作开始之前（Figure 3b），直接增加了 tRCD。

**解决方案一：Equalization transistor**

在 sense amplifier 内部增加一个 equalization transistor（Figure 4a），其控制信号复用 sense-en# 的互补信号，无需额外的全局布线。关键在于：当 wordline 打开进行 charge sharing 时（activate 的第一阶段），equalization transistor 同时将 sense amplifier 内部节点均衡到 Vdd/2。也就是说，sense amplifier 内部节点的 precharge 与 charge sharing **完全重叠**（Figure 4b），不额外占用时间。

- 仅靠 isolation transistor sizing（不加 equalization transistor）：tRCD 降低 16.67%
- 两者结合：tRCD 降低 **25%**

**解决方案二：Isolation transistor 的 "resistance shielding" 效应**

这是本文最精巧的电路级 insight。tRCD 由两个阶段构成：charge sharing + sense amplification。Sense amplification 的速度取决于 bitline 的 RC load。Isolation transistor 如果 sizing 为较高的 ON-resistance（small W, reasonable L），就能将 bitline 的 RC load "屏蔽"在 isolation transistor 之后，sense amplifier 看到的有效负载大幅减小，从而加速 sense amplification，降低 tRCD。

**Trade-off**：高 ON-resistance 会导致 restore 路径阻抗增大，tRAS 略微增加（+5%）。但论文通过 sensitivity study（Figure 5）证明系统性能对 tRAS 的敏感度远低于 tRCD，因此这是一个非常划算的 trade-off。

**Sizing 约束**：
- 不能太大（large W and L）：寄生电容大，开关瞬态噪声注入 bitline 造成 crosstalk
- 不能太小（minimum dimension）：对 process variation 敏感
- 最优点：small W + reasonable L，在降低 tRCD 的同时兼顾制造可行性

**面积开销**：sense amplifier 高度增加 16%，array 级别总面积开销 2.9%。基于 55nm DDR3 2Gb 工艺 [11] 的评估。

#### 2. 架构改动：Write-aware Row Management

**问题**：RBD 对所有 activate 都发 activate+precharge 命令。Write 需要数据 restore 回 bitcell，而 precharge 会断开 sense amplifier 与 bitline 的连接，导致 write 要么延迟（等 precharge 完成再 reconnect+restore），要么需要 DRAM 内部复杂的 command preemption 逻辑。

**SRP 的策略**：根据发起 activate 的请求类型做差异化处理：

- **Read-initiated activate**：发 activate+precharge 命令。tRAS 到期后自动 isolate + precharge bitline，sense amplifier 继续服务后续 read CAS。直到下一次同 bank 的 activate 到来。
- **Write-initiated activate**：**只发 activate 命令，不发 precharge**。Write CAS 正常执行，数据正常 restore 回 bitcell。当没有更多的 write row hit 时，memory controller 才发 precharge。

**为什么对 write 有效**：Write 请求通常在 write buffer 积累到一定量后 burst drain（为避免 read-write turnaround delay [14]），因此 write 天然具有聚簇性（clustering），不立即 precharge 不会显著损失 row locality。

这个设计的美妙之处在于：**将复杂性从 DRAM 芯片内部转移到了 memory controller**，MC 本身就擅长做命令调度，而 DRAM 内部电路应保持简单。

#### 3. 架构改动：Power-aware Row Management

**问题**：sense amplifier 解耦后一直持有数据直到下一次 activate，idle power 接近 open row policy。

**观察**：大部分 row hit 集中在 activate 后的一段时间内。超过这段时间后，继续保持 sense amplifier active 的边际收益递减。

**机制**（Figure 6）：
- 在 memory controller 中维护一个 **row open time table**，记录不同 row open duration 下的请求数量
- 对每个到达的请求，计算其到达时间与上一次 activate 时间的差值，更新对应条目
- 周期性地计算：使得 **hitthreshold** 比例的请求命中 row buffer 的最小 row open time
- 当 row open time 达到该值后，关闭 sense amplifier

论文选择 **hitthreshold = 95%**，实现了 sense amplifier idle power 降低约 30%（相比 RBD），性能损失可忽略。

### 与现有工作的区别

| 特性 | Baseline (Open/Close/Timeout) | RBD [ISCA 2014] | SRP (本文) |
|---|---|---|---|
| tRCD | 标准 | 增加（内部节点 precharge） | **降低 25%** |
| tRAS | 标准 | 标准 (32ns) | 略增 5% (34ns) |
| Read-Precharge 并行 | 不支持 | 支持 | 支持 |
| Write 处理 | 标准 | 复杂：reconnect-restore-disconnect | 简单：不发 precharge 直到无 write hit |
| Power awareness | 无 | 无 | 有：hitthreshold-based |
| 面积开销 | 0 | isolation transistor only | isolation + equalization，array 级 2.9% |
| 平均性能提升 (vs baseline) | - | ~4% | **~8.6%** |

与 **TL-DRAM** [HPCA 2013]：Lee et al. 也使用 isolation transistor，但用途完全不同——TL-DRAM 将 subarray 分为 near/far segment 来降低 near segment 的 latency，是 intra-array 的分段策略。SRP 的 isolation 是 sense amplifier 与 bitline 之间的解耦，目的是 enable read-precharge parallelism。两者互补。

与 **SALP** [ISCA 2012]、**Q-DRAM** [IEEE TC 2016]：SALP 解决 subarray-level parallelism（减少 bank conflict），Q-DRAM 通过额外 row buffer 将 restoration 移出 critical path。SRP 与它们正交，可以叠加使用。

## 实验评估

### 实验设置

- **仿真平台**：Intel 内部 cycle-accurate simulator，详细建模 DDR3-SDRAM memory system、processor cores、caches、interconnect、prefetchers
- **电路仿真**：SPICE，使用公开的 55nm DDR3 2Gb 工艺文件 [11]（Rambus DRAM power model）和 [9] 的晶体管模型
- **硬件配置**：
  - 1-4 cores, 3.2 GHz, 192-entry window
  - L1: 64KB/core, 8-way; L2: 512KB/core, 16-way; LLC: 2.5MB/core, 11-way
  - DDR3-1867, 12-12-12 timing, 2 channels, 1 rank/channel, 8 banks/rank
  - FRFCFS scheduling
  - Sub-row interleaved address mapping（4 cache blocks 跨 channel 交织）
- **Workload**：80 个单核应用（SPEC, SysMark, 3DMark, MLBench, database, animation, photo editing 等）；20 个 2-core 和 20 个 4-core 混合负载。每个应用执行 50M instructions。
- **对比 baseline**：Closed row, Open row, Static timeout, RBD [ISCA 2014]
- **SRP 参数**：tRCD = 9ns（降 25%），tRAS = 34ns（增 5%），hitthreshold = 95%
- **RBD 参数**：tRCD = 13ns，tRAS = 32ns（来自原文 [10]）

### 关键结果

1. **单核性能**：SRP 在 80 个 workload 上平均性能提升 **8.6%**（vs baseline），RBD 为 ~4%。SRP-Power-Aware 与 SRP 性能几乎持平。
2. **多核性能**：2-core 和 4-core workload 上 SRP 提升约 **8%**（vs baseline），RBD 仅约 2%。随着核数增加、memory contention 增大，activation latency 和 write latency 的降低效果更加显著。
3. **Sense amplifier idle power**：SRP-Power-Aware 相比 RBD 和 SRP（无 power awareness）降低 sense amplifier idle power 约 **30%**；甚至比 closed row policy 还低 15%（closed row 是所有 baseline 中 idle power 最低的）。
4. **Latency 分解**：activation latency reduction 和 write management 对 SRP 超越 RBD 的性能增益贡献**大致相等**。
5. **电路评估**：tRCD 降低 25%，tRAS 增加 5%。面积开销 2.9%（array level）。Refresh 行为不受影响。

### 结果分析

- 系统对 tRCD 的敏感度远高于 tRAS（Figure 5），这为 SRP 的"用 tRAS 换 tRCD"策略提供了坚实依据。
- Open row policy 的 sense amplifier idle power 是 closed row 的 ~8.3x。RBD/SRP（无 power awareness）虽然 sense amplifier 持有数据的 cycle 数类似 open row，但因为 sense amplifier 与 bitline 断开、负载更低，每 cycle 功耗为 baseline 的 1/7，总体 idle power 增加约 14-18%。
- hitthreshold = 95% 是 power-performance 的甜点：牺牲极少性能（<5% 的 row hit 被放弃），换取 30% 的 idle power 降低。
- 论文还在 ATLAS [HPCA 2010] 和 BLISS [ICCD 2014] scheduling 上验证，结论与 FRFCFS 一致，说明 SRP 与不同 memory scheduler 的组合是鲁棒的。

## 审稿人视角

### 优点

1. **电路与架构协同设计的典范**。Resistance shielding 降低 tRCD 的 insight 非常精彩——通常人们认为在 bitline 上加 transistor 只会增加延迟，本文反其道而行，利用高 ON-resistance 屏蔽 bitline RC load 加速 sense amplification。equalization transistor 复用 sense-en# 信号的设计也非常优雅，zero routing overhead。

2. **问题分解清晰，逐个击破**。将 RBD 的三个问题（activate latency、write complexity、idle power）独立分析、分别解决，且三个解决方案互不冲突、互相增强。论文结构和叙事逻辑非常清晰。

3. **Write 处理策略简洁高效**。不在 DRAM 内部增加复杂逻辑，而是利用 write burst drain 的天然特性在 MC 端做差异化处理，体现了对 DRAM-MC 接口设计哲学的深刻理解。

4. **评估全面**：电路级 SPICE 仿真验证 timing，系统级 cycle-accurate 仿真验证性能，80 个单核 + 40 个多核 workload，覆盖面广。同时评估了 performance、power、area 三个维度。

### 不足

1. **工艺节点过时，可扩展性存疑**。电路仿真基于 55nm DDR3 工艺，而 2018 年发表时主流已是 DDR4/1x-nm 节点。Isolation transistor 的 resistance shielding 效应在更小节点（leakage 更大、process variation 更严重）是否仍然成立？tRCD 25% 的改善是否可以直接 scale 到先进节点？论文未讨论。

2. **In-house simulator 不可复现**。使用 Intel 内部 cycle-accurate simulator，外部研究者无法验证结果。虽然这在工业界论文中常见，但对学术评审仍是一个减分项。对比之下，RBD 原文使用的是什么 simulator 也没有明确说明是否对齐。

3. **Workload 描述模糊**。80 个 workload 来自 SPEC、SysMark、3DMark 等多个 suite，但没有列出具体哪些 benchmark、哪些 input set。Figure 9 的性能图只展示了部分 workload 类别的聚合结果（如 "spark", "mlbench-logreg" 等），不是每个 benchmark 的独立数据。缺乏对 memory-intensive vs. compute-intensive workload 的分类分析。

4. **Power 评估不够完整**。只评估了 sense amplifier idle power，没有评估 isolation/equalization transistor 自身的 switching power 和 leakage，也没有给出系统级总功耗数据。30% idle power reduction 的绝对值在总 DRAM 功耗中占比多少？不清楚。

5. **缺少与 DRAM vendor 的交互验证**。论文声称 isolation transistor 在 folded bitline architecture 中已有产品先例，但没有引用具体产品或给出 DRAM vendor 的可行性评估。2.9% 的 array area overhead 在 DRAM die 的极端面积优化环境下是否可接受，需要更强的论证。

6. **Row open time table 的实现细节不足**。该表的条目数（论文中只展示了 4 个 bucket：256/512/1024/2048 cycles）、更新周期、硬件开销（计数器、比较器等）缺乏量化评估。在多 bank 场景下，每个 bank 是否需要独立的 table？

7. **未讨论 DDR4/DDR5 的适用性**。DDR4 引入了 bank group 概念，DDR5 进一步变化（same bank refresh、on-die ECC 等），这些对 SRP 的 row management 策略可能有影响。作为 2018 年的工作，缺少前瞻性讨论。

### 疑问或值得追问的点

- Resistance shielding 效应的量化模型是什么？论文给出了 SPICE 仿真结果，但缺少解析模型来帮助理解 tRCD reduction 与 isolation transistor sizing 之间的定量关系。
- hitthreshold = 95% 是否在所有 workload phase 都是最优？论文提到 row open time "varies across different workloads and phases"，但 hitthreshold 是全局静态的，是否应该做 per-phase adaptive？
- Equalization transistor 的 equalization 速度是否足够？如果 charge sharing phase 很短（在 aggressive timing 下），equalization 可能来不及完成，论文是否做了 corner case 分析？
- 在 multi-rank 场景下（论文只用了 1 rank/channel），rank-to-rank switching 与 SRP 的交互如何？SRP 是否在 multi-rank 配置下仍然有效？
- Write-aware management 在 write-intensive workload（如 STREAM write）下表现如何？论文的 workload mix 中是否有此类 stress test？
