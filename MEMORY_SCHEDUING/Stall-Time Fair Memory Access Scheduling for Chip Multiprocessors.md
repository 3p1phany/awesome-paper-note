---
title: "Stall-Time Fair Memory Access Scheduling for Chip Multiprocessors"
authors: "Onur Mutlu, Thomas Moscibroda"
venue: "MICRO 2007 (40th IEEE/ACM International Symposium on Microarchitecture)"
year: 2007
---

# Stall-Time Fair Memory Access Scheduling for Chip Multiprocessors

## 基本信息

- **作者**：Onur Mutlu, Thomas Moscibroda (Microsoft Research)
- **发表**：MICRO 2007（第40届 IEEE/ACM International Symposium on Microarchitecture）

## 一句话总结

> 提出基于 stall-time slowdown 均衡化的 DRAM 公平调度策略 STFM，在不损失吞吐量的前提下显著降低多核系统中线程间的内存访问不公平性。

## 问题与动机

### 核心问题

在 CMP（Chip Multiprocessor）系统中，多个核心共享 DRAM 内存子系统。传统的高性能内存调度器（如 FR-FCFS）以最大化 DRAM 吞吐量为目标，**完全不考虑线程间的干扰**，导致不同线程经历的 memory-related slowdown 差异极大。

论文用具体数据说明问题的严重性：

**4 核系统**：hmmer (Core 1), libquantum (Core 2), h264ref (Core 3), omnetpp (Core 4)

**8 核系统**：mcf (Core 1), hmmer (Core 2), GemsFDTD (Core 3), libquantum (Core 4), omnetpp (Core 5), astar (Core 6), sphinx3 (Core 7), dealII (Core 8)

在 4 核系统中，omnetpp 的 memory slowdown 高达 7.74X，而 libquantum 几乎没有减速（1.04X）；在 8 核系统中，dealII 的 slowdown 高达 11.35X，而 libquantum 仅为 1.09X。

### 为什么重要

论文指出 unfairness 带来四个层面的问题：

1. **OS 调度失效**：硬件层面的不公平使得操作系统基于优先级的线程调度策略形同虚设。
2. **安全威胁**：恶意程序可以利用不公平性对其他线程实施 denial of service 攻击（作者在另一篇 USENIX Security 2007 论文中详细论述）。
3. **性能不可预测**：应用的性能严重依赖于同时运行的其他应用的特征，难以分析和优化。
4. **计费不公**：在 grid computing 等按 CPU 时间计费的商业系统中，用户体验到的性能与支付的费用不成正比。

### 现有工作的不足

FR-FCFS 的不公平性源于两个子策略：

- **Column-first**：优先调度 row-buffer hit 请求，使高 row-buffer locality 的线程（如 libquantum，98.4% hit rate）持续获得优先服务。以 2KB row-buffer size、8 chips/DIMM、64B cache line 为例，一个 streaming thread 可以在另一线程的 row-conflict 请求之前插入多达 256 个 row-hit 请求。
- **Oldest-first**：隐式地优先服务 memory-intensive 线程，因为这些线程能更快地产生请求填满 request buffer，其请求总是"更老"。

## 核心方法

### 关键思路

**Key Observation**：不同线程在 DRAM 系统中经历的性能降级应该用 **memory-related slowdown**（即共享运行时的 memory stall time 与单独运行时的 memory stall time 之比）来刻画，而非简单地均分带宽。

**核心 Idea**：调度器为每个线程维护 $T_{shared}$（共享时的 memory stall time）和 $T_{alone}$（估计的单独运行时的 stall time），计算 slowdown $S = T_{shared}/T_{alone}$。当线程间的 slowdown 差异超过阈值时，优先调度 slowdown 最大的线程的请求。

这一思路的根本洞察在于：**DRAM fairness 不是一个纯粹的 bandwidth allocation 问题**，因为 DRAM 系统有 state（row buffer）和 parallelism（multiple banks），带宽分配无法直接映射到性能。

### 技术细节

#### 1. STFM 调度策略

STFM 的调度决策流程如下：

**Step 1 — 判断 Unfairness**：在所有有 ready request 的线程中，找到 slowdown 最大的线程（$S_{max}$）和最小的线程（$S_{min}$）。

**Step 2a — FR-FCFS Rule**：若 $S_{max}/S_{min} \leq \alpha$（阈值，默认 $\alpha = 1.10$），则 unfairness 在可接受范围内，使用标准 FR-FCFS 策略以优化吞吐量。

**Step 2b — Fairness Rule**：若 $S_{max}/S_{min} > \alpha$，则启用公平调度，优先级顺序为：

- **$T_{max}$-first**：slowdown 最大的线程的 ready commands 最优先
- **Column-first**：在同一线程内部，ready column accesses 优先于 row accesses
- **Oldest-first**：更早到达的请求优先

这是一个**两模式切换**的设计：通常使用 FR-FCFS 最大化吞吐量，只在 unfairness 超标时切换到 fairness 模式。

#### 2. $T_{alone}$ 的估算机制

$T_{alone}$ 的准确估算是 STFM 的核心难点。论文将其转化为：

$T_{alone} = T_{shared} - T_{Interference} $

因此只需估算 $T_{Interference}$（由其他线程的干扰造成的额外 stall time）。

**更新其他线程的 $T_{Interference}$**：当线程 C 的请求 R 被调度时，对其他线程 C' 的 $T_{Interference}$ 从两个维度更新：

**(a) DRAM Bus 干扰**：R 的 read/write command 占用数据总线 $t_{bus}$ 周期（DDR2 下 $t_{bus} = BL/2$），期间其他有 ready read/write command 的线程 C' 的 $T_{Interference}$ 增加 $t_{bus}$。

**(b) DRAM Bank 干扰**：对于与 R 在同一 bank 有 ready command 的线程 C'，其额外 stall time 为 R 的 service latency 除以 C' 的 **BankWaitingParallelism**（C' 在多少个 bank 有等待中的请求）：

$T_{Interference}^{new}(C') = T_{Interference}^{old}(C') + \frac{Latency(R)}{\gamma \cdot BankWaitingParallelism(C')}$

其中 $\gamma = 1/2$ 是一个 scaling factor，用来校正 bank parallelism 估算的不精确性（乘以 $1/\gamma = 2$ 相当于左移一位，硬件实现极为简单）。

**更新自身线程的 $T_{Interference}$**：线程 C 自身的请求也可能因干扰而遭受额外延迟。关键场景是：两个连续请求 R1、R2 访问同一 bank 的同一 row，单独运行时 R2 会是 row-hit（latency = $t_{CL}$），但共享运行时其他线程的请求可能插入导致 R2 变成 row-conflict（latency = $t_{RP} + t_{RCD} + t_{CL}$）。

STFM 通过维护每个线程在每个 bank 的 **LastRowAddress** 来判断这种情况。若发生上述"本应 row-hit 变为 row-conflict"，则：

$T_{Interference}^{new}(C) = T_{Interference}^{old}(C) + \frac{ExtraLatency}{BankAccessParallelism(C)}$

其中 $ExtraLatency = t_{RP} + t_{RCD}$，$BankAccessParallelism$ 是当前正在 DRAM banks 中被服务的该线程请求数。注意 $ExtraLatency$ 也可以为负值（共享时反而因其他线程打开了有利的 row 而获益）。

#### 3. 硬件实现

STFM 在 baseline FR-FCFS 控制器基础上增加侧路逻辑，不改变基本的 request buffer 和 priority encoder 结构：

| 寄存器                                  | 功能                                           | 位宽                   |
| ------------------------------------ | -------------------------------------------- | -------------------- |
| $T_{shared}$ (per-thread)            | 核心提供的 memory stall cycles                    | 24 bits              |
| $T_{Interference}$ (per-thread)      | 干扰造成的额外 stall cycles                         | 24 bits              |
| Slowdown (per-thread)                | $T_{shared}/(T_{shared} - T_{Interference})$ | 8 bits (fixed point) |
| BankWaitingParallelism (per-thread)  | 有等待请求的 bank 数                                | 3 bits               |
| BankAccessParallelism (per-thread)   | 正在服务请求的 bank 数                               | 3 bits               |
| LastRowAddress (per-thread per-bank) | 每个线程在每个 bank 最后访问的 row 地址                    | 14 bits              |
| ThreadID (per-request)               | 请求所属线程 ID                                    | 3 bits               |

对于 8 线程、8 bank、$2^{14}$ rows/bank、128-entry request buffer 的配置，**额外硬件开销仅为 1808 bits**。

所有寄存器每隔 IntervalLength（$2^{24}$）周期或 context switch 时重置，以适应线程的 phase behavior。

#### 4. 系统软件支持

- **$\alpha$ 参数可配置**：OS 可通过 privileged instruction 设置 $\alpha$；若不需要硬件 fairness，设置极大的 $\alpha$ 即可退化为 FR-FCFS。
- **Thread Weights**：支持线程权重机制。weighted slowdown 计算为 $S = 1 + (S - 1) \times Weight$，高权重线程被解释为经历了更大的 slowdown 从而获得优先服务。

### 与现有工作的区别

#### vs. FR-FCFS

FR-FCFS 是 thread-unaware 的，其 column-first 和 oldest-first 策略系统性地偏向高 row-buffer locality 和高 memory-intensity 的线程。STFM 通过估算每个线程的 slowdown 来感知这种不公平并动态纠正。

#### vs. FCFS

纯 FCFS 消除了 row-buffer locality exploitation 带来的不公平，但仍然隐式偏向 memory-intensive 线程（因为它们的请求更早到达），且完全放弃了 row-buffer locality 优化，导致吞吐量显著下降。

#### vs. Network Fair Queueing (NFQ) [Nesbit et al., MICRO 2006]

NFQ 借鉴网络 fair queueing 的思想，为每个线程在每个 bank 维护 virtual deadline，试图均分 DRAM 带宽。论文指出 NFQ 有两个根本性问题：

1. **Idleness Problem**：非 bursty 的线程在没有竞争时正常使用 DRAM，其 virtual deadline 不断推进。当 bursty 线程突然开始发请求时（virtual deadline 为 0），它们会完全抢占非 bursty 线程的带宽，导致后者被 starve。
2. **Access Balance Problem**：bank 访问不均衡的线程在其集中访问的少数 bank 上 virtual deadline 增长过快，被不公平地降低优先级。

STFM 通过直接度量 slowdown（而非管理带宽分配）来避免这些问题，因为 slowdown 天然地反映了所有内在内存特征（row-buffer locality、bank parallelism、memory intensity、access balance）的综合效果。

#### vs. FR-FCFS+Cap（论文新提出的 baseline）

对 FR-FCFS 的 column-over-row reordering 加一个 cap（限制最多 4 个 younger column access 可以 bypass 一个 older row access）。缓解了 row-buffer locality 不公平但未解决 FCFS 固有的对 non-intensive 线程的惩罚问题。

## 实验评估

### 实验设置

- **仿真平台**：Cycle-accurate x86 CMP simulator，functional front-end 基于 Pin，DRAM model 基于 DRAMsim，performance model 基于 Intel Pentium M。
- **处理器配置**：4 GHz，128-entry instruction window，12-stage pipeline，3-wide fetch/exec/commit；每核 32KB L1（4-way），512KB L2（8-way，64 MSHRs）。
- **DRAM 配置**：DDR2-800（$t_{CL}=t_{RCD}=t_{RP}=15$ns，$BL/2=10$ns），8 banks，2KB row-buffer/bank，128-entry request buffer，open-page policy，XOR-based address-to-bank mapping。DRAM channel 数随核数线性增长（2核/4核=1 channel，8核=2 channels，16核=4 channels）。
- **Uncontended round-trip L2 miss latency**：row-hit 140 cycles，row-closed 200 cycles，row-conflict 280 cycles。
- **Workload**：SPEC CPU2006（26 benchmarks，按 memory intensity 和 row-buffer locality 分为 4 类）+ Windows 桌面应用。每个 benchmark 运行 SimPoint 选定的 100M 指令。2核评估25组、4核评估256组、8核评估32组、16核评估3组 workload 组合。
- **对比 Baselines**：FR-FCFS、FCFS、FR-FCFS+Cap（新提出）、NFQ（Nesbit et al. MICRO 2006 的 FQ-VFTF 方案）。

### 评价指标

- **Unfairness Index**：$\frac{\max_i MemSlowdown_i}{\min_i MemSlowdown_i}$（越小越好，理想值为 1）
- **Weighted Speedup**：$\sum_i \frac{IPC_i^{shared}}{IPC_i^{alone}}$（衡量系统吞吐量）
- **Hmean Speedup**：各线程相对 IPC 的调和平均（平衡 fairness 和 throughput）
- **Sum of IPCs**：纯吞吐量指标（论文指出此指标应谨慎使用）

### 关键结果

**1. 2 核系统（mcf 与其他 25 个 benchmark 配对）**：

- STFM 将平均 unfairness 从 2.02（FR-FCFS）降至 1.24，降低 76%；最大 unfairness 仅为 1.74。
- Weighted speedup 提升 1%，hmean speedup 提升 6.5%。

**2. 4 核系统（256 个 workload 组合平均）**：

- 平均 unfairness：FR-FCFS=5.31, FCFS=1.80, FR-FCFS+Cap=1.65, NFQ=1.58, **STFM=1.24**。
- STFM 比 NFQ 改善 weighted speedup 5.8%，hmean speedup 10.8%。

**3. 8 核系统（32 个 workload 组合平均）**：

- 平均 unfairness：FR-FCFS=5.26, FR-FCFS+Cap=2.64, NFQ=2.53, **STFM=1.40**。
- FR-FCFS 的 unfairness 从 4 核的 5.31 增长到 8 核的 5.26（channel 数同步增加），其他方案的 unfairness 也随核数增加而恶化，而 STFM 仅从 1.24 增至 1.40。
- STFM 比 NFQ 改善 weighted speedup 7.6%。

**4. 16 核系统**：

- NFQ 的 fairness 严重恶化（idleness 和 access balance 问题加剧），甚至不如 FCFS 和 FR-FCFS+Cap。
- STFM 将平均 unfairness 从 2.23（FCFS，此时 second-best）降至 1.75，weighted speedup 和 hmean speedup 分别比 NFQ 提升 4.6% 和 15%。

### 结果分析

#### Case Study 深度分析

论文通过三个 4 核 case study 揭示了不同调度器的行为差异：

- **Memory-intensive workload**（mcf, libquantum, GemsFDTD, astar）：NFQ 因 idleness problem 严重惩罚持续发请求的 mcf（3.4X slowdown），因 access balance problem 严重惩罚 bank 访问集中的 astar（3.3X）。STFM 将 unfairness 从 1.87（NFQ）降至 1.27。
- **Mixed workload**（mcf, leslie3d, h264ref, bzip2）：FR-FCFS 反而比 FCFS/FR-FCFS+Cap 更 fair（因为这组 benchmark 的 row-buffer locality 差异不大），NFQ 反而增加了 unfairness。这说明 **其他方案的 fairness 表现高度依赖 workload 特征**，而 STFM 始终表现最优。
- **Non-intensive workload**（libquantum, omnetpp, hmmer, h264ref）：NFQ 对 omnetpp 产生 3.47X slowdown，原因是 h264ref 的 bursty 访问在共同 bank 上抢占了 omnetpp 的请求，降低了 omnetpp 的 bank parallelism。

#### Sensitivity Analysis

- **DRAM bank 数**（4/8/16）：FR-FCFS 的 unfairness 随 bank 增加略有下降（bank conflict 减少），STFM 在所有配置下保持 ~1.40 的 unfairness，且 weighted speedup 改善随 bank 数增加而增大（4 banks: 5.4%, 8 banks: 7.6%, 16 banks: 11.1%），因为更多 bank 给了 STFM 更大的调度灵活性。
- **Row-buffer size**（1KB/2KB/4KB）：FR-FCFS 的 unfairness 随 row-buffer 增大而增加（row-buffer locality exploitation 更严重），STFM 始终保持 ~1.38-1.40 的 unfairness。
- **$\alpha$ 参数**：$\alpha = 1.10$ 是最佳 trade-off 点。$\alpha = 1.0$ 导致 fairness rule 过于频繁触发，损失吞吐量；$\alpha = 1.05$ 因为 slowdown 估算不够精确也稍有不足。

#### 桌面应用评估

在 xml-parser + matlab（memory-intensive，高 row-buffer locality）+ iexplorer + instant-messenger（non-intensive）的 4 核场景中：FR-FCFS 的 unfairness 高达 8.88，NFQ 降至 1.75，STFM 进一步降至 1.37，且 weighted speedup 提升 5.4%，hmean speedup 提升 10.7%。

## 审稿人视角

### 优点

1. **问题定义精准，fairness metric 设计有深度**：论文提出的 stall-time fairness 定义抓住了 DRAM 系统有 state（row buffer）和 parallelism（multiple banks）的本质特征，很有说服力地论证了为什么简单的 bandwidth-based fairness（如 NFQ）在 DRAM 场景下是不充分的。对 NFQ 的 idleness problem 和 access balance problem 的分析尤其深刻。

2. **方法设计兼顾 fairness 和 throughput**：通过 $\alpha$ 阈值实现两模式切换，在 unfairness 可接受时不干扰 FR-FCFS 的吞吐量优化，这是一个非常实用的工程设计。实验也证实 STFM 不仅改善 fairness，还同时提升了 weighted speedup，这是一个很强的结果。

3. **硬件开销极低，实现可行性强**：额外仅需 1808 bits 的存储开销，运算逻辑只涉及加减法、移位（$\gamma = 1/2$ 的巧妙选择），且 memory controller 不在处理器关键路径上，有充裕的时间余量。

4. **实验评估极为全面**：256 个 4 核 workload 组合、32 个 8 核组合、跨越 2/4/8/16 核的可扩展性评估、对 bank 数和 row-buffer size 的 sensitivity analysis、桌面应用 case study、$\alpha$ 参数扫描、thread weight 功能验证——评估的广度和深度在同期工作中属于上乘。

5. **系统软件接口设计周到**：$\alpha$ 和 thread weight 可由 OS 配置，使得 STFM 可以作为一个灵活的 fairness substrate 而非僵化的策略。

### 不足

1. **$T_{alone}$ 估算精度存疑，缺乏误差分析**：$T_{Interference}$ 的估算依赖多个近似（BankWaitingParallelism 的均摊、$\gamma$ 因子、是否在 critical path 等），论文承认估算不总是准确（Section 7.2.1 提到对 libquantum 的 slowdown 略有低估），但缺乏系统性的误差量化分析。不同访问模式下估算精度可能差异很大。

2. **只考虑了 open-page policy**：实验全部基于 open-page policy，而 close-page policy 在服务器场景中也很常见。在 close-page 下 row-buffer locality 不再是 unfairness 的主要来源，STFM 的改善幅度可能会显著不同。

3. **Memory-level parallelism (MLP) 的建模过于简化**：用 BankWaitingParallelism 和 BankAccessParallelism 来近似 MLP 是一个相当粗糙的启发式函数。实际的 MLP 取决于请求间的依赖关系、instruction window 的状态等，远比"有多少 bank 有等待请求"复杂。论文在脚注 8 中承认这一点，但未做深入探讨。

4. **Workload 规模和多样性的局限**：虽然评估了大量组合，但全部是 multiprogrammed workload（无 shared memory 的多线程程序）。对于 multi-threaded workload（如 PARSEC），线程间的内存访问可能有 sharing 和同步模式，STFM 的效果未知。

5. **未考虑 write 请求的复杂性**：论文提到 reads 优先于 writes，但对 read-write 切换开销（bus turnaround penalty）、write drain 模式等对 fairness 的影响缺乏讨论。

6. **Scalability 分析不够充分**：16 核只评估了 3 个 workload 组合，且 STFM 的 unfairness 已从 4 核的 1.24 上升到 16 核的 1.75。随着核数进一步增加（32 核、64 核），$T_{Interference}$ 的估算复杂度和精度都会面临更大挑战，论文对此未做预测性分析。

7. **IntervalLength 的选择缺乏理论依据**：$2^{24}$ 周期的重置间隔是经验性的，论文只提到小于 $2^{18}$ 时效果变差。对于 phase behavior 剧烈变化的程序，这个固定间隔可能不够自适应。

### 疑问或值得追问的点

- STFM 的 fairness rule 被触发的频率如何？在典型 workload 中，fairness mode 和 FR-FCFS mode 的时间占比各是多少？这直接影响了 throughput 的 trade-off。
- 当所有线程都是 memory-intensive 且 row-buffer locality 相似时，STFM 是否退化为接近 FR-FCFS 的行为？此时 unfairness 本身就较低，STFM 的额外硬件是否成为纯粹的 overhead？
- $T_{Interference}$ 的估算只考虑了 bank-level 和 bus-level 的干扰，未考虑 rank-level timing constraints（如 $t_{FAW}$、$t_{RRD}$）和 refresh 造成的干扰。这些在高 bank 利用率下可能是显著的 fairness 影响因素。
- 论文的 $T_{shared}$ 定义是"核心因 L2 miss 无法 commit 的周期数"，这实际上包含了 memory controller queueing delay + DRAM access latency + 网络传输延迟。在 NUMA 或有 shared cache 的系统中，这个度量是否还合适？
- 从后续研究的视角看，STFM 开创了"slowdown-based fairness"的思路，直接启发了 Mutlu & Moscibroda 2008 年 ISCA 的 PARBS 以及后续的 ATLAS（HPCA 2010）、TCM（MICRO 2010）等一系列重要工作。这条研究线索的演进方向值得关注。
