---
title: "Thread Cluster Memory Scheduling: Exploiting Differences in Memory Access Behavior"
authors: "Yoongu Kim, Michael Papamichael, Onur Mutlu, Mor Harchol-Balter"
venue: "MICRO 2010"
year: 2010
---

# Thread Cluster Memory Scheduling: Exploiting Differences in Memory Access Behavior

## 基本信息

- **作者**：Yoongu Kim, Michael Papamichael, Onur Mutlu, Mor Harchol-Balter（Carnegie Mellon University）
- **发表**：MICRO 2010

## 一句话总结

> 将线程按 memory intensity 分为两个 cluster，分别施加不同调度策略，同时优化 system throughput 和 fairness。

## 问题与动机

在 CMP 系统中，多个线程共享 off-chip memory，memory scheduling algorithm 需要同时追求两个目标：**system throughput**（所有线程整体吞吐高）和 **fairness**（没有线程被过度减速）。这两个目标在先前工作中呈现明确的 trade-off 关系：

- **ATLAS**（HPCA 2010）：优化 system throughput，通过优先服务 attained service 最少的线程，但其严格的 static priority 导致 memory-intensive 线程被严重饿死，fairness 极差（maximum slowdown 比 PAR-BS 高 55.3%）。
- **PAR-BS**（ISCA 2008）：优化 fairness，通过 batching 机制保障公平，但 batch 中 memory-intensive 线程的请求数量远多于 memory-non-intensive 线程，导致后者被隐式阻塞，system throughput 较低（比 ATLAS 低 2.9%）。
- **STFM**（MICRO 2007）：用 stall-time 估计启发式衡量 slowdown 并优先调度被减速最多的线程，但估计不够精准。
- **FR-FCFS**（ISCA 2000）：thread-unaware，仅优先 row-buffer hit，在多线程场景中 throughput 和 fairness 均差。

论文的核心 motivation 是：**没有任何一个先前算法能同时达到最优 throughput 和最优 fairness**——它们在 Figure 1 的 throughput-fairness 二维图中互相 Pareto-dominate，无法同时占据右下角（高 throughput + 低 max slowdown）。

## 核心方法

### 关键思路

TCM 的核心 insight 是：**不同 memory intensity 的线程对调度策略的需求截然不同**——memory-non-intensive（latency-sensitive）线程需要低延迟快速服务以最大化 throughput，而 memory-intensive（bandwidth-sensitive）线程需要公平的带宽分配以避免 starvation。因此，不应对所有线程施加同一种策略，而应**将线程动态分为两个 cluster，分别采用不同的调度策略**。

第二个关键 insight 是：**bandwidth-sensitive 线程之间的干扰程度不同**。高 row-buffer locality 的 streaming 线程会长期霸占某个 bank，对其他线程造成干扰（hostile）；而高 bank-level parallelism 的 random-access 线程因同时访问多个 bank 而更容易被干扰（fragile）。论文构造了一个经典的 motivating example（Table 1 & Figure 2）：同为 100 MPKI 的两个线程，random-access 被 deprioritize 时 slowdown 超过 11x，远超 streaming 线程——这说明调度算法需要区分 bandwidth-sensitive 线程的**干扰特性**。

### 技术细节

#### 1. Thread Clustering（线程分簇）

TCM 在每个 quantum（1M cycles）的起始，按 MPKI（L2 misses per kiloinstruction）对所有线程排序，将最轻的线程放入 **latency-sensitive cluster**，其余放入 **bandwidth-sensitive cluster**。划分依据是参数 **ClusterThresh**：将线程按 MPKI 从小到大加入 latency-sensitive cluster，直到该 cluster 的累计 bandwidth usage 达到上一 quantum 总 bandwidth usage 的 ClusterThresh 倍。

- ClusterThresh 的推荐范围为 `2/N` 到 `6/N`（N 为线程数），论文默认取 `4/24`。
- 分簇在所有 memory controller 之间**同步**进行（通过 centralized meta-controller 交换信息），保证全局一致。

Algorithm 1 的伪代码清晰展示了这一逻辑：按 MPKI 升序依次加入 latency-sensitive cluster，直到带宽预算用尽。

#### 2. Latency-Sensitive Cluster 的调度策略

Latency-sensitive cluster 中的线程**始终被严格优先**于 bandwidth-sensitive cluster。cluster 内部进一步按 MPKI 从低到高排序——MPKI 最低的线程优先级最高。这一策略直接沿用了 ATLAS 等先前工作的 insight：优先服务轻量线程能极大提升 throughput，且因其请求量少，对重量线程几乎无影响。

#### 3. Bandwidth-Sensitive Cluster 的调度策略：Niceness 与 Insertion Shuffle

这是论文最核心的技术贡献。Bandwidth-sensitive 线程之间需要**周期性 shuffle priority order** 以保证公平。但 shuffle 的方式至关重要：

**Niceness metric 的定义：** 对 bandwidth-sensitive cluster 中的线程 i，设其 bank-level parallelism 排名为 b_i（越高越前），row-buffer locality 排名为 r_i，则：

```
Niceness_i = b_i - r_i
```

- 高 BLP → 更 fragile（容易被干扰）→ 更 nice → 应更多时间在高优先级
- 高 RBL → 更 hostile（容易干扰他人）→ 更 not nice → 应更多时间在低优先级

**Insertion Shuffle 算法：** 每 quantum 开始时，按 niceness 升序排列（nicest thread 最高优先级）。之后每隔 ShuffleInterval（默认 800 cycles），执行一次 shuffle 操作。Insertion shuffle 的核心特点是**非对称**的：least nice thread 大部分时间停留在最低优先级，而 nicer threads 更多地占据高优先级位置（如 Figure 3(b) 所示），从而允许 nice threads 之间互相 "leak" memory service。

Algorithm 2 展示了具体操作：交替执行递增/递减排序的子过程，产生一系列确定性排列。

**Round-Robin 为何不行：** 论文深刻分析了两个问题。第一，round-robin 不感知线程间的干扰差异。第二，存在 **memory service "leakage" 效应**：在 per-bank 调度中，最高优先级线程如果在某 bank 没有请求，service 会自然 "漏" 给次优先级线程。Round-robin 保持线程间的相对位置不变，导致某些"幸运"线程始终排在"漏水"线程后面而获得额外 service，造成不公平。

**同质线程的退化处理：** 当 bandwidth-sensitive cluster 中线程的 BLP 和 RBL 差异较小时（max ΔBLP < ShuffleAlgoThresh × NumBanks 且 max ΔRBL < ShuffleAlgoThresh），TCM 自动退化为 **random shuffle**，避免因微小差异导致不合理的固定偏好。ShuffleAlgoThresh 默认为 0.1。

#### 4. Memory Access Behavior 监控

- **Memory intensity**：L2 MPKI，由 L2 cache controller 计算。
- **Row-buffer locality**：使用 **shadow row-buffer index**（记录每个线程在每个 bank 单独运行时应打开的行），计算 shadow hit rate。
- **Bank-level parallelism**：采样每个线程在有 outstanding request 时同时有请求的 bank 数目，取 quantum 内平均值。

硬件开销方面，每个 controller 额外存储不到 4 Kbits（Table 2 详列）；如果仅用 random shuffle，则不到 0.5 Kbits。

#### 5. 完整的优先级规则（Algorithm 3）

1. **Highest-rank first**：latency-sensitive > bandwidth-sensitive；cluster 内按各自策略排序。
2. **Row-hit first**：同优先级下，row-buffer hit 请求优先。
3. **Oldest first**：其余情况下，older request 优先。

#### 6. OS 接口：Thread Weights 与 Trade-off Knob

- **Thread weights**：TCM 在 cluster 框架内尊重 OS 权重——latency-sensitive cluster 中用权重缩放 MPKI，bandwidth-sensitive cluster 中按权重分配高优先级时间，但不会因高权重 bandwidth-sensitive 线程而牺牲 latency-sensitive 线程的性能。
- **ClusterThresh** 作为 trade-off knob 暴露给系统软件，可在 throughput 和 fairness 之间平滑调节。

### 与现有工作的区别

| 维度 | ATLAS | PAR-BS | TCM |
|------|-------|--------|-----|
| 核心策略 | 严格按 attained service 排全局优先级 | Batching + 批内按 BLP 排序 | 双 cluster + 分策略调度 |
| Throughput | 最优（先前） | 较低（batch 阻塞轻线程） | 最优 |
| Fairness | 极差（heavy 线程 starve） | 最优（先前） | 最优 |
| 可调性 | QuantumLength 调节范围有限，始终偏 throughput | BatchCap 调节范围有限，始终偏 fairness | ClusterThresh 可平滑连续调节 |
| 线程干扰感知 | 无 | 感知 BLP 但不感知 RBL | Niceness = BLP ranking − RBL ranking |

## 实验评估

### 实验设置

- **仿真平台**：自研 cycle-level x86 CMP simulator（前端基于 Pin），memory subsystem 使用 DDR2 timing 参数（经 DRAMSim 和真实硬件验证）。
- **硬件配置**：24 cores，4 independent DRAM controllers（每个 6.4 GB/s peak bandwidth），128-entry instruction window，3-wide pipeline，512KB L2/core（8-way），DDR2-800（tCL=tRCD=tRP=15ns，BL/2=10ns，4 banks/rank，2KB row-buffer）。
- **Workload**：SPEC CPU2006 的 25 个 benchmark，按 MPKI > 1 划分为 memory-intensive / memory-non-intensive。构造 50%、75%、100% memory intensity 的 multiprogrammed workloads，每类 32 个，共 96 个 workloads，每个运行 100M cycles。
- **对比 baseline**：FR-FCFS、STFM、PAR-BS、ATLAS。
- **评价指标**：Weighted Speedup（throughput）、Maximum Slowdown（fairness）、Harmonic Speedup。

### 关键结果

1. **全 96 workloads 平均**：TCM 比 ATLAS（最优 throughput baseline）提升 throughput 4.6% 并降低 max slowdown 38.6%；比 PAR-BS（最优 fairness baseline）提升 throughput 7.6% 并降低 max slowdown 4.6%。
2. **100% memory-intensive workloads**：TCM 优势更大——比 PAR-BS 提升 throughput 7.4%、降低 max slowdown 5.8%；比 ATLAS 提升 throughput 10.1%、降低 max slowdown 48.6%。
3. **Shuffle 算法对比**（Table 6，32 workloads）：Round-robin max slowdown 均值 5.58，Random 5.13，Insertion 4.96，TCM（自适应切换）4.84 且方差最低（0.85 vs round-robin 的 1.61）。
4. **OS Thread Weights 场景**（Figure 8）：在最不利权重分配下，TCM 比 ATLAS 提升 throughput 82.8% 并降低 max slowdown 44.2%。

### 结果分析

**ClusterThresh 的 sensitivity**（Section 7.1, Figure 6）：从 2/24 到 6/24 变化时，TCM 在 throughput-fairness 平面上形成平滑的 Pareto 曲线，而 ATLAS 和 PAR-BS 的参数调节只能在各自偏好的维度上微调，无法有效 trade-off。这是 TCM 最显著的结构性优势之一。

**Workload intensity 的影响**（Section 7.2, Figure 7）：随着 workload memory intensity 增加，TCM 的优势增大——因为在高竞争下，ATLAS 的 strict ranking 导致最重线程严重 starve，PAR-BS 的 batching 导致轻线程被阻塞，而 TCM 的双 cluster 设计能有效隔离这两类问题。

**系统配置 sensitivity**（Table 8）：在不同 core 数（4-32）、controller 数（1-8）、cache size（512KB-2MB）下，TCM 始终优于 ATLAS（throughput 提升 0%-5%，max slowdown 降低 4%-53%）。

**ShuffleInterval sensitivity**（Table 7）：减小 ShuffleInterval 会降低 row-buffer locality，导致小幅性能下降；800 cycles 是一个较好的折中。

## 审稿人视角

### 优点

1. **问题分解精准，设计逻辑清晰**：将 throughput 和 fairness 问题解耦到两个不同的 cluster 分别优化，是一个极具洞察力的架构设计选择。整篇论文的逻辑链非常完整：observation → insight → mechanism → evaluation，几乎每一步设计都有对应的 motivating experiment。

2. **Niceness metric 设计简洁有效**：将 BLP 和 RBL 的排名差作为 niceness，直觉清晰且实现成本低。结合 insertion shuffle 的非对称优先级分配，这是一个在 simplicity 和 effectiveness 之间取得了很好平衡的设计。

3. **Evaluation 全面且有说服力**：96 个 workloads、多维度 sensitivity analysis（ClusterThresh、ShuffleInterval、ShuffleAlgoThresh、core/controller/cache 配置）、shuffle 算法的消融实验、OS thread weight 场景，覆盖面非常广。Figure 6 的 Pareto frontier 可视化尤其有力地展示了 TCM 的结构性优势。

4. **Practical considerations 考虑充分**：包括同质线程退化为 random shuffle、meta-controller 的 scalability 分析（每 1M cycles 交换 4 bytes/context/controller）、以及如何在 cluster 框架内尊重 OS thread weights，表明作者充分考虑了实际部署的工程问题。

### 不足

1. **仿真平台的时代局限性**：DDR2-800，4 banks/rank，24 cores 但只有 4 channels——这在 2010 年合理，但对现代系统（DDR5, 16+ banks/bank group, 多 rank, 数十 channel）的适用性存疑。特别是现代系统中 bank group 结构引入了新的 timing constraint（tCCD_S/L），BLP 的含义和效果会有显著变化。论文未讨论这种 scalability concern。

2. **Niceness metric 的粗糙性**：`Niceness = b_i - r_i` 仅用 rank 差值，丢失了绝对值信息。如果 BLP 和 RBL 的分布高度偏斜（例如某线程 BLP 远超其他所有线程），rank-based metric 无法捕捉这种量级差异。此外，niceness 仅考虑两个维度，未纳入 memory intensity 本身在 bandwidth-sensitive cluster 内部的差异。

3. **Centralized meta-controller 的实际可行性**：虽然论文认为通信量小（每 1M cycles 一次），但在现代 NUMA 系统中，跨 socket 的全局同步和 centralized 决策可能成为部署障碍。论文对 distributed 实现方案缺乏深入讨论。

4. **Workload 代表性**：仅使用 SPEC CPU2006 的 multiprogrammed 组合，缺少 server workloads（数据库、web serving 等高并发场景）和 data-intensive workloads（graph analytics、ML training）。此外，100M cycles 的仿真长度对于捕捉长周期 phase behavior 可能不够充分。

5. **与 FR-FCFS 的耦合**：TCM 的 row-hit first 规则（Algorithm 3 的 Rule 2）本质上仍依赖 FR-FCFS 的 row-buffer hit 优先思路。在 row-buffer conflict 严重的场景中，这可能与 fairness 目标产生冲突——一个高 RBL 的 hostile 线程在其"主场 bank"上仍会因 row-hit first 获得大量 service，而 TCM 的 niceness-based shuffling 只控制线程间的**相对优先级**，并不直接限制单线程在单 bank 上的 row-buffer hit 串。

### 疑问或值得追问的点

- **Insertion shuffle 的理论最优性**：论文给出了 insertion shuffle 的直觉解释和经验优越性，但缺乏理论分析——在什么条件下 insertion shuffle 是最优的 shuffling strategy？是否存在更优的非对称 shuffle 方案？
- **动态 phase 适应性**：quantum 长度固定为 1M cycles，如果线程的 memory behavior 在 quantum 内发生剧烈变化（例如从 compute phase 突然进入 memory phase），TCM 的响应会有一个 quantum 的延迟。论文未评估 phase transition 频率对性能的影响。
- **与 prefetching 的交互**：论文评估中未明确提及 hardware prefetcher 的配置。在有 aggressive prefetching 的系统中，MPKI 和 BLP 的测量可能被 prefetch request 扭曲，影响 clustering 和 niceness 的准确性。
- **Shadow row-buffer index 的开销**：每个 thread 每个 bank 需要一个 shadow row-buffer index 来估计 alone RBL——在 thread 数和 bank 数增长后（如现代系统可能有 64+ banks），这部分存储开销的 scalability 值得关注。
