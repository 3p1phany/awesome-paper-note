---
title: "A Study of Performance Impact of Memory Controller Features in Multi-Processor Server Environment"
authors: "Chitra Natarajan, Bruce Christenson, Fayé Briggs"
venue: "WMPI '04 (Workshop on Memory Performance Issues), Munich, Germany"
year: 2004
---

# A Study of Performance Impact of Memory Controller Features in Multi-Processor Server Environment

## 基本信息

- **发表**：WMPI '04 (Workshop on Memory Performance Issues, held in conjunction with ISCA), 2004
- **作者单位**：Intel Corporation, Santa Clara
- **关键词**：Memory controller, Memory transaction scheduling, Multi-processor server, DDR/DDR2

## 一句话总结

> 系统性评估了 MP server 环境下 page policy、command scheduling、rank interleaving、read/write switching 及 request reordering 等 memory controller 特性对 loaded latency-throughput 性能的影响。

## 问题与动机

**核心问题**：在 multi-processor server 环境下，memory controller 的各项微架构特性（page policy、command scheduling overlap、rank interleaving、read-write switching、request reordering）对系统内存性能（loaded latency vs. delivered throughput）的量化影响是什么？

**重要性**：2004 年前后，processor 性能以约 55%/年增长，而 DRAM latency 仅以约 7%/年下降、bandwidth 以约 20%/年增长（引自 Hennessy & Patterson），memory wall 问题日益严重。processor 侧的 prefetching、OoO execution、speculation 等技术虽然能容忍部分 memory latency，但同时也增加了对 memory bandwidth 的需求。因此，memory controller 端的优化对于提升系统实际可达性能至关重要。

**现有工作不足**：此前的研究（Alakarhu & Niittylahti 2002, Cuppu & Jacob 2001 等）主要关注 uni-processor desktop/workstation 环境，使用 SPECint/SPECfp benchmark，而 MP server 环境下的 memory access pattern 与 desktop 有本质不同——多个 processor 和 IO 设备同时访问内存产生 "localized random access pattern"，desktop 场景下的最优 MC 策略在 server 场景下可能严重失效。

## 核心方法

### 关键思路

论文的核心方法论是**渐进式特性叠加分析**（incremental feature stacking）：从一个简单的 desktop-origin memory controller (MC-A) 出发，逐步叠加 rank interleaving、page-close + overlapped scheduling、intelligent read-to-write switching、OoO request reordering、delayed write scheduling 等特性，通过 latency-vs.-throughput 曲线量化每项特性的边际收益。

关键洞察：MP server workload 产生的 "localized random" access pattern 使得 desktop 环境下有效的 page-open policy 在 server 场景下因高 page-miss rate（43%）而严重低效；需要从根本上切换到 page-close + overlapped scheduling 的策略组合。

### 技术细节

论文通过 4 个递进式 study 展开分析：

**Study #1：基础特性对比（MC-A → MC-B）**

起点 MC-A 是一个从 desktop 移植到 DP server 的简单 memory controller，具有以下特征：
- **Page-open policy**：Open-policy1（保持 page open 直到 conflict）或 Open-policy2（idle n cycles 后关闭，n 可配置为 0/8/16/64）。实际 server benchmark 测试中 Open-policy1 性能极差，Open-policy2 with n=0 最优。
- **Non-overlapped scheduling**：假设 page hit 为主，简化实现。page hit 场景下命令序列为 RD-NOP-RD-NOP-RD；但 page empty 场景下退化为 ACT1-NOP-RD1-NOP-ACT2-NOP-RD2...，而 overlapped scheduling 可实现 ACT1-ACT2-RD1-ACT3-RD2... 的高效序列。
- **No rank interleaving**：高位地址映射到不同 rank，连续地址映射到同一 page/bank/rank，遍历完一个 rank 才到下一个。
- **In-order request processing**：read 优先，write queue 接近满或无 read 时 burst 4 个 write。

逐步叠加的特性：
1. **MC-A + rank interleaving**：用低位 bit（如 bit 12 & 13）映射 rank，将小区域内存分布到多 rank 多 bank。
2. **MC-A + page-close + overlapped scheduling**：使用 auto pre-charge 关闭 page，overlapped ACT 和 RD/WR command 到独立 bank。
3. **MC-B = page-close + overlapped + rank interleaving**：三项特性的组合，request processing 仍为 in-order。

关键数据：MC-A 在 server workload 下观察到 53% page-hit、43% page-miss、4% page-empty。加入 rank interleaving 后变为 53% hit、29% miss、18% empty——大量 page-miss 转化为 page-empty，这是性能提升的直接原因。

**Study #2：Intelligent Read-to-Write Switching**

基于 MC-B 改进 in-order request processing。传统策略仅在 write queue 接近满或 read queue 为空时切换到 write。新策略增加一个触发条件：**当 read queue 的 head-of-queue 因 bank conflict 而 blocked 时，opportunistically 切换到 write**。

核心设计权衡：bank conflict 造成的 idle cycle 远大于 read-write turnaround penalty，因此利用 bank conflict 的等待时间处理 write 是净收益。

**Study #3：OoO Memory Controller（Request Reordering）**

在 Intel 870 平台（O-o-O IPF FSB）上评估 OoO MC vs. in-order MC。OoO MC 的关键设计：
- Read queue 中 look-ahead 最多 4 个 request，寻找 conflict-free 的 request 来调度。
- Write 切换策略：(1) read queue 空且有 ≥4 outstanding write 时 opportunistically burst ≥4 writes；(2) write queue 接近满时 burst ≥16 writes。
- 由于 request reordering 大幅降低了整个 read queue 被 block 的概率，intelligent read-to-write switching 变得不再必要。
- Write queue 同样做 request reordering 以减少 bank conflict 造成的 channel idle。

**Study #4：Delayed Write Scheduling**

进一步优化 OoO MC 的 read latency。问题：当 read queue 为空时，MC 立即切换到 write 并发起 ACT 操作保持 channel busy，但如果此后很快有 read 到来，这些 read 被不必要地 block。

解决方案：MC 计算从当前 cycle 到 "必须发起 write ACT 以避免 data channel 出现不必要 idle" 的最后时刻（last possible cycle），在此之前等待。若等待期间有 read 到来则优先处理 read；若无 read 到来则在最后时刻发起 write ACT，不浪费 bandwidth。

这是一个经典的 **latency-bandwidth trade-off**：传统策略以最大化 channel utilization 为目标（bandwidth-oriented），delayed write scheduling 则牺牲了一点 write 的及时性来降低 read latency。

### 与现有工作的区别

1. **Rixner et al. (ISCA 2000)**：最早提出 memory access scheduling 的概念，但针对 Imagine Stream Processor（media processing），而非 MP server 环境。本文报告了 MP server 中类似量级的 bandwidth improvement（12%-30%），呼应了 Rixner 的结论在不同场景下的普适性。

2. **Cuppu & Jacob (ISCA 2001)**：研究了 concurrency、latency、system overhead 对 uni-processor DRAM 性能的影响，但局限于单核环境。本文将分析扩展到 MP server，发现 desktop 最优策略在 server 场景下失效。

3. **Alakarhu & Niittylahti (2002)**：对比了 pre-charge policy 在现代 DRAM 架构上的表现，但同样是 desktop 环境。本文的 Study #1 直接展示了 page-open policy 在 MP server workload 下因 43% page-miss rate 而严重低效。

## 实验评估

### 实验设置

- **仿真平台**：Intel 内部开发的 near cycle-accurate C++ simulation models，包含 FSB model、MCH model（含 memory controller）、memory channel model、IO bus model。非公开工具，不是 gem5/DRAMSim3 等学术模拟器。
- **硬件配置**：
  - Study #1：IA32 in-order FSB 133MHz/533MT/s，2× lockstep DDR266A (2-3-3) channels，4 single-rank DIMMs per channel
  - Study #2：IA32 in-order FSB 167MHz/667MT/s，2× lockstep DDR266 (2-2-2) channels，1/2/4 single-rank DIMMs per channel
  - Study #3：IPF O-o-O FSB 200MHz/400MT/s (6.4GB/s peak)，4× lockstep 1.6GB/s memory channels → DMH → 2× independent DDR200 channels，4 DIMMs per DDR channel
  - Study #4：O-o-O server，2× lockstep DDR2-400 channels，4 ranks per channel
- **Workload**：
  - Study #1：MP server FSB trace（来自 web-server benchmark 的真实 trace）
  - Study #2/3/4：统计生成的 random address traffic，2:1 read-to-write mix（representative of many server workloads），均匀随机地址分布，指数分布的 inter-arrival time
- **对比 baseline**：逐步对比方式——MC-A（desktop origin）→ MC-A + rank interleaving → MC-A + page-close + overlapped → MC-B → MC-B + intelligent r/w switch → OoO MC → OoO MC + delayed write

### 关键结果

1. **Page-close + overlapped scheduling vs. page-open + non-overlapped**：在 server workload 下，page-close + overlapped scheduling 在整个 throughput 范围内大幅降低 loaded read latency（从 Fig. 2 看，在相同 throughput 下 latency 降低约 50-100ns），且 max sustainable throughput 显著提升。

2. **Rank interleaving 的效果**：将 page-miss 比例从 43% 降至 29%，page-empty 从 4% 升至 18%，且其收益与 page-close + overlapped scheduling 是**叠加的**（additive）。

3. **Intelligent read-to-write switching 的性能提升与 doubling interleaved ranks 同量级**：在 in-order FSB 系统中，该特性的 latency-throughput 改善相当于将 interleaved rank 数量翻倍。

4. **OoO MC vs. in-order MC**：在 Intel 870 平台上，OoO MC 在 delivered FSB BW > 2GB/s 时降低 loaded read latency > 10ns；max sustained bandwidth 提升 12%-30%（取决于 read-write mix）。

5. **Delayed write scheduling**：显著降低 loaded read latency，使 latency-vs.-throughput 曲线在更宽的 throughput 范围内保持较低且平坦（从 Fig. 6 看，在接近 max BW 时 latency 差异可达 ~80-100ns）。

### 结果分析

论文对结果的解释围绕以下几个维度：

- **Page hit/miss/empty 分布变化**：rank interleaving 通过将地址分散到更多 independent bank 来降低 page-miss 概率，这是定量可解释的。
- **Channel utilization**：overlapped scheduling 和 request reordering 通过减少 bank conflict 造成的 idle cycle 来提升 channel 有效利用率。
- **Workload demand curve 分析**：论文引入了 workload demand curve 的概念——server benchmark 中 latency 降低会导致 demanded bandwidth 增加，形成正反馈。这使得 memory controller 优化的实际收益需要结合 demand curve 交点来评估。特别地，CPU prefetching 在 MC-A 上因增加了过多 load 导致性能反而下降，而 MC-B 的更好 latency-throughput 曲线使得 prefetching 才能发挥正面作用。

论文没有做严格的 sensitivity analysis（如参数扫描），但通过不同 DIMM 数量（1/2/4 DIMMs per channel）和两种 CPU type 的 workload demand curve 提供了部分参数敏感性信息。

## 审稿人视角

### 优点

1. **工业界视角的系统性评估**：论文的最大价值在于提供了一个来自 Intel 内部的、基于近 cycle-accurate 模拟器和真实 server trace 的系统性评估。每项 MC 特性的边际收益被清晰量化，且渐进式叠加的实验方法论非常清晰，便于工程决策。

2. **实际工程洞察丰富**：多个发现具有直接工程指导意义——例如 desktop MC 直接用于 server 场景时 page-open policy 严重失效、intelligent read-to-write switching 的收益等价于翻倍 rank 数（这对硬件 BOM cost 决策非常有价值）、CPU prefetching 在弱 MC 上反而有害等。

3. **Delayed write scheduling 的设计思路**：Study #4 提出的 "计算 last possible cycle 再发起 write ACT" 的思路虽然简单，但很优雅地平衡了 read latency 和 channel utilization，这个 idea 在后来的 MC 设计中被广泛采用。

4. **Workload demand curve 的引入**：将 memory 性能改善与 workload 行为变化的反馈效应纳入分析框架，这比单纯看 latency-throughput 曲线更贴近真实 benchmark 性能。

### 不足

1. **Workload 代表性不足**：Study #1 使用了一条 web-server benchmark trace，Study #2-4 使用统计生成的 uniform random traffic。虽然论文声称在 rank interleaving + page-close 条件下统计 traffic 与 trace-driven 结果相近，但缺乏对更多实际 server workload（如 OLTP、HPC、数据库）的验证。Uniform random address 假设过于理想，无法捕捉真实 workload 中的 row buffer locality 变化。

2. **仿真工具不可复现**：使用 Intel 内部非公开的 simulation model，外部研究者无法验证或复现结果。虽然 2004 年的 workshop paper 对此要求不如顶会严格，但从学术角度看这是明显短板。

3. **缺乏对 fairness 和 per-source 性能的分析**：在 MP 环境下，memory controller 的 scheduling 策略不仅影响整体 throughput 和平均 latency，还涉及多个 processor/IO 之间的 fairness 问题。论文完全没有讨论 request reordering 对不同 source 的 starvation/fairness 影响，这对 server workload 尤为关键。

4. **OoO MC 的 look-ahead window 仅为 4**：Study #3 中 OoO MC 只 look ahead 4 个 read request，但没有做 window size 的 sensitivity analysis。后来的研究（如 Rixner 的 FR-FCFS 及其变体）表明 scheduling window size 对性能有显著影响，论文遗漏了这一分析维度。

5. **缺乏面积/功耗/复杂度的讨论**：论文完全聚焦于性能收益，没有讨论各项特性带来的硬件实现开销（如 OoO scheduling 的 comparator 逻辑、delayed write 的 timing 计算逻辑等），对 MC 设计者而言不够完整。

6. **实验年代局限性**：DDR/DDR2 时代的 timing 参数和 system topology（shared FSB、MCH 集成）与现代 DDR4/DDR5/HBM 架构差异巨大。虽然基本原理（如 page policy 选择、command overlap、request reordering）仍然适用，但具体的量化结论难以直接推广。

### 疑问或值得追问的点

- 论文提到 rank interleaving 使 page-miss 从 43% 降至 29%、page-empty 从 4% 升至 18%，但 page-hit 保持 53% 不变。这意味着 rank interleaving 主要将原来的跨 page conflict 转化为了 bank idle 状态（empty），而没有增加 locality。是否有更 aggressive 的 address mapping 方案能同时提升 page-hit rate？
- Delayed write scheduling 中 "last possible cycle" 的计算依赖于 write request 对应 bank 的当前状态和 timing constraint。论文没有详述这个计算的具体实现——在多 bank、多 rank 的情况下，是否需要对所有 pending write 都计算一遍？这个计算的硬件延迟是否会成为 critical path？
- Study #2 声称 intelligent read-to-write switching 的收益等价于翻倍 rank 数量，但这个等价关系是否依赖于 read-write mix ratio？在 write-heavy workload 下这个结论是否仍然成立？
- 论文的 workload demand curve 是 "estimated" 的，具体如何从 latency 变化估算 demanded bandwidth 的变化没有说明。这个 feedback model 的精度对结论的影响值得进一步验证。
