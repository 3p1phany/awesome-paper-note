---
title: "The Blacklisting Memory Scheduler: Balancing Performance, Fairness and Complexity"
authors: "Lavanya Subramanian, Donghyuk Lee, Vivek Seshadri, Harsha Rastogi, Onur Mutlu"
venue: "SAFARI Technical Report No. 2015-004 (arXiv:1504.00390)"
year: 2015
---

# The Blacklisting Memory Scheduler: Balancing Performance, Fairness and Complexity

## 基本信息

- **发表**：SAFARI Technical Report No. 2015-004, March 2015 (arXiv)
- **单位**：Carnegie Mellon University
- **通讯作者**：Onur Mutlu

## 一句话总结

> 通过将应用二分组（而非全序排列）并以连续请求计数实现分组，以极低硬件开销同时提升性能与公平性。

## 问题与动机

在多核系统中，运行在不同核心上的应用共享 off-chip DRAM 主存，导致 inter-application interference。这种干扰会严重降低系统整体性能（system performance）并造成不公平的应用减速（unfair slowdowns）。随着核心数量增长，该问题将愈发严重。

先前的 application-aware memory scheduler（如 PARBS、ATLAS、TCM）试图通过监测应用的 memory access characteristics 并对所有应用建立 total rank order 来缓解干扰。但这类方案存在两个核心缺陷：

1. **硬件复杂度过高**：基于 total rank order 的调度需要大量比较逻辑和存储开销。以 TCM 为例，其 critical path latency 是 FRFCFS 的 8 倍，面积是 1.8 倍。高 critical path latency 使调度器无法满足 DDR3/DDR4 的严格时序要求（如 tCCD = 4 cycles 的 worst-case timing constraint），严重阻碍实际部署。
2. **排名底部应用遭受不公平减速**：total rank order 导致排名靠后的应用被严重去优先化。即使 TCM 周期性地 shuffle 排名，应用在大部分 shuffle interval 内仍处于低排名状态，导致高 memory-intensity 应用经历极高的 slowdown，降低系统公平性。

## 核心方法

### 关键思路

论文基于两个关键观察提出 BLISS：

**Observation 1**：为缓解 inter-application interference，不需要对所有应用建立 total rank order，只需将应用分为两组——interference-causing 组和 vulnerable-to-interference 组，优先调度后者即可。这种二分组策略不仅降低硬件复杂度，还避免了 total rank order 中底部应用被严重去优先化的问题。

**Observation 2**：应用的分组可以通过一个极简方法实现——监测 memory controller 处每个应用的 consecutive requests served 数量。当某应用连续被服务的请求数超过阈值时，将其归类为 interference-causing（即 "blacklist" 该应用）。其直觉是：当大量连续请求来自同一应用时，其他应用的请求必然被延迟，导致其他应用 stall。

### 技术细节

#### Blacklisting Mechanism

BLISS 的 blacklisting 机制仅需维护三个量：

- **Application ID**：最后一个被调度请求的 application（hardware context）ID，一个寄存器。
- **\#Requests Served**：来自同一应用的连续请求计数器。
- **Blacklist bit vector**：每个 hardware context 一个 bit，标记其是否被 blacklist。

工作流程如下：当 memory controller 即将发射一个请求时，比较该请求的 application ID 与上一个被调度请求的 Application ID：

- 若相同：`#Requests Served` 递增。
- 若不同：`#Requests Served` 重置为零，更新 Application ID。

当 `#Requests Served` 超过 **Blacklisting Threshold**（默认为 4）时：

- 对应应用被 blacklist。
- `#Requests Served` 重置为零。

Blacklist 信息每隔一个 **Clearing Interval**（默认 10000 cycles）全部清除，确保不会因长时间去优先化某个应用而引发 starvation 或不公平。

#### Blacklist-Based Memory Scheduling

请求调度优先级（从高到低）：

1. Non-blacklisted 应用的请求（缓解干扰）
2. Row-buffer hit 请求（优化 DRAM bandwidth utilization）
3. Older 请求（保证 forward progress）

#### 硬件开销分析

存储开销极小：

- 1 个 Application ID 寄存器（$\log_2 N_{cores}$ bits）
- 1 个 `#Requests Served` 计数器
- 1 个 Blacklisting Threshold 寄存器
- 1 个 N-bit blacklist bit vector（N = hardware context 数）
- 每个 request queue entry 需存储其 application ID

逻辑开销仅包含：比较 `#Requests Served` 与 Blacklisting Threshold、查找 blacklist bit vector 以确定请求优先级。

### 关键设计权衡

1. **Blacklisting Threshold 的选择**：阈值过小（如 2）导致过多应用被 blacklist，干扰缓解效果差；阈值过大（如 8）导致 high/low memory-intensity 应用区分不充分，公平性下降。阈值 4 在性能和公平性之间取得最佳平衡。

2. **Clearing Interval 的选择**：清除过于频繁（1000 cycles）使得 interference-causing 应用未被去优先化足够长时间，性能较低；清除过于罕见（100000 cycles）导致应用被去优先化过久，公平性下降。10000 cycles 是最优折中点。

3. **同步 vs. 异步清除**：论文对比了全局同步清除（所有应用同时清除 blacklist）与异步清除（每个应用在被 blacklist 后经过 Clearing Interval 个 cycles 后独立清除）。两种方案性能和公平性相近（WS: 9.18 vs. 9.12, MS: 6.54 vs. 6.60），但同步清除更简单，无需为每个应用维护额外计数器。

4. **仅 blacklist row-buffer hit 的局限**：FRFCFS-Cap-Blacklisting（仅 blacklist 高 row-buffer locality 应用）虽性能接近 BLISS，但公平性更差（high memory-intensity but low row-buffer locality 的应用未被约束），且需要 per-bank counter，硬件成本更高。BLISS 仅需 per-channel 一个计数器。

### 与现有工作的区别

| 特性                               | TCM                             | ATLAS     | PARBS       | BLISS                             |
| -------------------------------- | ------------------------------- | --------- | ----------- | --------------------------------- |
| 排序方式                             | 全序排名（cluster 内）                 | 全序排名      | batch 内全序排名 | 仅二分组，无排名                          |
| 分组粒度                             | 百万 cycles 级 memory intensity 聚类 | 无分组       | 无分组         | 千 cycles 级 consecutive request 监测 |
| Critical path latency（相对 FRFCFS） | 8.1x                            | 5.3x      | 11x         | 1.7x                              |
| 面积开销（相对 FRFCFS）                  | 1.8x                            | 1.7x      | 2.4x        | 1.032x                            |
| 满足的最高 DDR 频率                     | DDR3-800                        | DDR3-1333 | 低于 DDR3-800 | DDR4-3200                         |

核心区别：

- **与 TCM 的区别**：TCM 虽有 clustering 但在 cluster 内仍进行 per-application ranking；BLISS 完全避免 per-application ranking。TCM 的 clustering 基于 coarse-grained memory intensity 测量（百万 cycles 级别），对行为动态变化的应用适应性差；BLISS 基于 fine-grained consecutive request 监测（千 cycles 级别）。
- **与 PARBS 的区别**：PARBS 通过 batching 约束公平性，但 batch 内仍进行 per-application ranking。PARBS 公平性最好（MS 比 BLISS 低 10%），但系统性能低 8%，且 critical path 是 BLISS 的 6.5 倍。
- **与 FRFCFS-Cap 的区别**：FRFCFS-Cap 仅约束当前 ongoing row-buffer hit streak，不 blacklist 应用。因此对 high memory-intensity 但 low row-buffer locality 的应用无效。BLISS 通过 blacklist 机制持续去优先化干扰应用直到 clearing interval 结束。

## 实验评估

### 实验设置

- **仿真平台**：cycle-level in-house DDR3-SDRAM simulator（经 Micron behavioral Verilog model 和 DRAMSim2 验证），集成 cycle-level in-house out-of-order execution core simulator，前端由 Pin 驱动。
- **RTL 综合**：Verilog RTL 实现 + Synopsys Design Compiler + 商业 32nm 标准单元库。
- **硬件配置**：
  - 主配置：24 cores, 4 channels, 1 rank/channel, 8 banks/rank, 8KB row-buffer
  - 处理器：5.3GHz, 3-wide issue, 8 MSHRs, 128-entry instruction window
  - L2 cache：512KB per core private, 64B cache-line, 16-way
  - Memory：DDR3-1066 (8-8-8), 128-entry read/write request queue per controller
- **Workload**：SPEC CPU2006 + TPC-C + Matlab + NAS parallel benchmarks，均为 single-threaded。按 MPKI > 5 分类为 memory-intensive，构造 25%/50%/75%/100% memory-intensive 比例的四类 workloads，每类 20 个，共 80 个 workloads。每个 workload 仿真 100M representative cycles。
- **对比 baseline**：FRFCFS, FRFCFS-Cap, PARBS, ATLAS, TCM, Crit-MaxStall, Crit-TotalStall
- **评价指标**：Weighted Speedup（性能）, Maximum Slowdown（公平性）, Harmonic Speedup（性能-公平性平衡）, Critical Path Latency (ns), Area (μm²)

### 关键结果

1. **性能与公平性**：BLISS 比最佳先前调度器 TCM 实现 **5% 更高的 weighted speedup** 和 **25% 更低的 maximum slowdown**，harmonic speedup 提升 19%。
2. **硬件复杂度**：BLISS 将 memory scheduler 的 **critical path latency 降低 79%**、**面积降低 43%**（相比 TCM）。BLISS 的 critical path 仅为 FRFCFS 的 1.7 倍，可满足 DDR4-3200 的 worst-case timing（2.5 ns），而 TCM 仅能满足 DDR3-800（10 ns）。
3. **与 TCM-Cluster（去除 TCM 内排名）对比**：性能相近，但 BLISS 公平性显著更好，因为 TCM-Cluster 在百万 cycles 粒度上分组，对行为变化的应用适应性差。
4. **Sensitivity**：BLISS 在 16-64 核、1-8 通道、512KB-2MB cache、shared cache（32MB）、cache block interleaving、sub-row interleaving 等配置下均表现出一致的优势。性能收益在系统 bandwidth 越受限时越大（高核数、低通道数）。

### 结果分析

**Streak length distribution 分析**（Section 7.4）是理解 BLISS 有效性的关键。BLISS 将 interference-causing 应用（如 libquantum: MPKI 52, RBH 99%）的 streak length 分布向左移动（打断长连续请求），同时将 vulnerable 应用（如 calculix: MPKI 0.1, RBH 85%）的 streak length 分布向右移动（使其获得更多连续服务机会）。

**Average request latency 不是好的性能/公平性指标**：FRFCFS 的平均请求延迟最低（因为最大化 row-buffer hits），但系统性能和公平性最差。提供最好公平性的调度器（PARBS, FRFCFS-Cap, BLISS）反而有较高的平均延迟，因为它们避免让 low-memory-intensity 应用独占低延迟路径。

**与 interleaving policy 的交互**：在 cache block interleaving 和 sub-row interleaving 下，FRFCFS 本身性能已大幅提升（并行度提高减少干扰），但 BLISS 仍能在此基础上提供接近最优性能，同时保持公平性优势。值得注意的是，sub-row interleaving + BLISS 可能因 capping decision 叠加导致 high-row-buffer-locality 应用被过度约束，unfairness 略高于 FRFCFS——作者指出 scheduling-interleaving co-design 是重要的 future work。

## 审稿人视角

### 优点

1. **问题定义精准，motivation 有力**：论文清晰地指出了 prior art 在 performance-fairness-complexity 三维度上的 trade-off 困境，并通过 Figure 1 的三维 radar plot 和 RTL 综合数据给出了非常有说服力的量化 motivation。TCM 的 8x critical path latency 和无法满足 DDR3-800 以上频率的 timing constraint 这一点，对实际部署是致命的。

2. **设计极为简洁，insight 深刻**：两个 observation 抓住了问题的本质——二分组足够、consecutive request counting 足够。整个机制仅需一个计数器、一个寄存器、一个 bit vector，逻辑极简。这种"less is more"的设计哲学非常适合 memory controller 的严格时序约束。

3. **评估极为全面**：80 个 workloads、6 个 baseline、RTL 综合、核数/通道数/cache size 的 sensitivity analysis、与不同 interleaving policy 的交互分析、参数 sensitivity、多种变体对比（TCM-Cluster、FRFCFS-Cap-Blacklisting、async clearing）——几乎穷尽了审稿人可能提出的实验需求。

4. **RTL-level complexity analysis 是重要贡献**：此前的 memory scheduling 论文多停留在 cycle-level simulation，BLISS 通过 Verilog RTL + 32nm 综合给出了 critical path 和 area 的精确对比，直接回答了"能否真正部署"这一关键问题。

### 不足

1. **仿真平台为 in-house simulator，非开源工具**：虽然声明经 Micron Verilog model 和 DRAMSim2 验证，但 in-house simulator 的可复现性较差。使用 gem5 + DRAMSim2/Ramulator 等公开工具会增强结果可信度。

2. **Workload 构成偏传统**：全部使用 single-threaded SPEC CPU2006 + 少量 TPC-C/Matlab/NAS，缺乏对 multi-threaded 应用（如 PARSEC）、GPGPU workloads、以及 data-intensive workloads（如 graph processing, ML inference）的评估。在当今异构计算场景下，这些 workloads 的 memory access pattern 可能对 BLISS 的 consecutive request counting 策略构成新的挑战。

3. **Blacklisting Threshold 和 Clearing Interval 为静态参数**：虽然 sensitivity study 表明 Threshold=4, Interval=10000 是好的默认值，但不同 workload phase 和不同系统配置可能需要动态调整。论文未探讨自适应参数调整机制。

4. **对 write traffic 的讨论不足**：论文提到 128-entry read/write request queue，但 blacklisting 机制似乎未区分 read 和 write 请求。在 write-intensive workloads（如 persistent memory 系统）中，write drain 行为可能导致大量连续 write 请求被发射，这些是 memory controller 内部行为而非真正的 inter-application interference。BLISS 是否会错误地 blacklist 这些应用？

5. **缺乏功耗评估**：虽然面积开销很小，但未报告功耗数据。对于 power-constrained 设计（如移动端 SoC），功耗是同样重要的约束维度。

6. **公平性指标单一**：仅使用 maximum slowdown 衡量公平性。Maximum slowdown 对 outlier 敏感，建议补充 average slowdown、slowdown variance 或 Jain's fairness index 等指标以更全面地评估公平性。

### 疑问或值得追问的点

- **Blacklisting 的 per-channel 独立性**：论文提到 blacklisting 在每个 channel 独立进行。若某应用在 channel 0 被 blacklist 但在 channel 1 未被 blacklist，其跨 channel 的请求分布是否会导致策略失效？尤其在 row-interleaved 系统中，同一应用的请求可能集中在某些 channel。
- **与 DRAM refresh 的交互**：DRAM refresh 会周期性地阻塞 bank，导致请求被延迟。在 refresh 密集的场景下（如高容量 DRAM），consecutive request counting 是否会受到 refresh-induced gap 的干扰？
- **扩展到 HBM/LPDDR 等新型存储接口的适用性**：HBM 的 pseudo-channel 架构和 LPDDR 的不同 timing 参数是否会影响 BLISS 的 Threshold/Interval 最优值？
- **Clearing Interval 的粒度与 DDR timing 的对齐**：10000 cycles 的 clearing interval 是否应与 DRAM 的 tREFI 或其他周期性事件对齐，以避免与 refresh 操作的冲突？
