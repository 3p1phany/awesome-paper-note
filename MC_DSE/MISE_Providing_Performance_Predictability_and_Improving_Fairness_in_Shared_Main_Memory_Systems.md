---
title: "MISE: Providing Performance Predictability and Improving Fairness in Shared Main Memory Systems"
authors: "Lavanya Subramanian, Vivek Seshadri, Yoongu Kim, Ben Jaiyen, Onur Mutlu"
venue: "ISCA 2013 (推测，基于作者群体和研究方向；论文本身未明确标注会议名，IEEE版权页显示2013)"
year: 2013
---

# MISE: Providing Performance Predictability and Improving Fairness in Shared Main Memory Systems

## 基本信息
- **作者**：Lavanya Subramanian, Vivek Seshadri, Yoongu Kim, Ben Jaiyen, Onur Mutlu
- **单位**：Carnegie Mellon University (SAFARI Group)
- **发表**：IEEE, 2013

## 一句话总结
> 通过 request service rate 代理性能、最高优先级采样估算 alone performance，构建简洁准确的 memory interference slowdown 估计模型。

## 问题与动机

**核心问题**：在多核系统中，多个 application 共享 main memory 会产生 inter-application interference，导致各 application 经历不同且不可预测的 slowdown。如何在运行时准确估计每个 application 因 memory interference 导致的 slowdown？

**为什么重要**：
- 准确的 slowdown 估计可以支撑 QoS 保障机制——memory controller 可以据此智能分配 memory bandwidth，而非简单地全部优先或全部平等。
- OS 可以利用 slowdown 信息做出更好的 job scheduling 决策（如将互相干扰小的 application co-schedule）。
- 在 server consolidation 场景下，performance predictability 对于 SLA 的满足至关重要。

**现有工作的不足**：
- **STFM (Stall-Time Fair Memory Scheduling, MICRO'07)**：通过统计 application 因 interference 额外增加的 stall cycle 来估计 alone-stall-time，进而推导 slowdown。问题在于：(1) 在 out-of-order 处理器中，interference 导致的额外 latency 可能被 MLP (memory-level parallelism) 所隐藏，导致计数不准确；(2) 没有考虑 non-memory-bound application 的 compute phase，对此类 application 的 slowdown 估计严重偏差。跨 300 个 workload 的平均误差高达 29.8%。
- **FST (Fairness via Source Throttling, ASPLOS'10)** 和 **Du Bois et al. (HiPEAC'13)**：原理与 STFM 类似——都是试图在 application 受干扰的状态下估算干扰带来的额外 cycle，本质上面临相同的 MLP 建模困难。
- 上述工作的 **设计初衷** 并非精确估计 slowdown，而是用粗略的估计来做 prioritization/throttling 决策。MISE 的目标则是 **精确估计 slowdown 本身**。

## 核心方法

### 关键思路

MISE 基于两个关键观察：
1. **Observation 1**：Memory-bound application 的性能与其 request service rate（请求被服务的速率）大致成正比——在真实 Intel Core i7 系统上用 SPEC CPU2006 的 mcf、omnetpp、astar 验证了这一线性关系。
2. **Observation 2**：给一个 application 的 request 赋予最高优先级时，它受到其他 application 的干扰非常小，其 request service rate 近似于 alone 状态下的 service rate。

由此，slowdown 可以简单计算为 alone-request-service-rate (ARSR) 与 shared-request-service-rate (SRSR) 的比值。

### 技术细节

#### 1. 基本 Slowdown 模型（Memory-bound Application）

$$\text{Slowdown} = \frac{\text{ARSR}}{\text{SRSR}} \quad \text{(Equation 1)}$$

- **SRSR 计算**：直接由 memory controller 通过 per-application counter 统计在一个 interval 内被服务的请求数除以 interval 长度。
- **ARSR 估计**：在 application 获得最高优先级的 epoch 内，统计其被服务的请求数，除以（最高优先级的 epoch 周期数 - interference cycle 数）：

$$\text{ARSR} = \frac{\text{\# HPE Requests}}{N \cdot \text{\# HPEs} - \text{\# Interference Cycles}}$$

其中 interference cycle 的定义：在 application 拥有最高优先级期间，其请求在 request buffer 中等待，但上一个被发射到某 bank 的 command 属于另一个 application 的 cycle。这是因为调度是 **work-conserving** 的——当最高优先级 application 没有 ready request 时，controller 会调度其他 application 的 request，而 DRAM command 一旦发射不可抢占。

#### 2. 扩展模型（Non-memory-bound Application）

Non-memory-bound application 有显著的 compute phase（core 不 stall 等内存），request service rate 的变化只影响 memory phase，不影响 compute phase。引入参数 α（stall fraction）：

$$\text{Slowdown} = (1 - \alpha) + \alpha \cdot \frac{\text{ARSR}}{\text{SRSR}} \quad \text{(Equation 3)}$$

$$\alpha = \frac{\text{\# Cycles spent stalling on memory requests}}{\text{Total number of cycles}}$$

**实际使用中的 Threshold 机制**：当 application 中等偏高 memory-intensive（α 较大）时，直接用 Equation 1（即设 α=1）反而更准确。因此最终模型是：当 α < Threshold 时用 Equation 3，否则用 Equation 1。

#### 3. Scheduling 实现：Lottery-Scheduling-like Approach

- 执行时间被划分为 **interval**（M = 5M cycles）和 **epoch**（N = 10K cycles）。
- 每个 interval 开始时，controller 估计所有 application 的 slowdown，并可能调整 bandwidth allocation。
- 每个 epoch 开始时，controller **概率性地** 选择一个 application 赋予最高优先级。概率分布由 bandwidth allocation 决定：分配到更多带宽的 application 有更高概率获得最高优先级。
- 在一个 epoch 内，最高优先级 application 的所有 request 被优先调度；无 ready request 时才调度其他 application 的 request（work-conserving）。

#### 4. 基于 MISE 模型的两个应用机制

**MISE-QoS（Soft QoS Guarantee）**：
- 为一个或多个 Application of Interest (AoI) 设定 slowdown bound B。
- 每个 interval 结束时，若 AoI 的估计 slowdown < B，则减少其 bandwidth 分配 2%；若 > B，则增加 2%。
- 多 AoI 场景：为每个 AoI 独立维护 bandwidth allocation，各自根据自身 slowdown 与 bound 的比较来增减。
- 当无法满足 bound 时（即使 100% bandwidth 给 AoI 仍超 bound），controller 向 OS 汇报。

**MISE-Fair（Minimize Maximum Slowdown）**：
- 维护一个全局 target bound B 和 per-application bandwidth allocation。
- 每个 interval 将 application 分为两个 cluster：slowdown < B 和 slowdown > B。从前者各 steal 2% bandwidth 分配给后者。
- **动态调整 B**：过去 N=3 个 interval 中若所有 application 都满足 bound，则将 B 降低至略低于当前 max slowdown（更激进）；若超过半数不满足，则提高 B 至略高于 max slowdown（更宽松）。

#### 5. Hardware Cost

每个 hardware thread context 需要 5 个 counter（SRSR 1个 + ARSR 3个 + stall cycle 1个）+ 1 个 bandwidth allocation register = **24 bytes per thread context**，非常轻量。

### 与现有工作的区别

| 对比维度 | MISE | STFM [Mutlu & Moscibroda, MICRO'07] |
|---------|------|------|
| 性能代理指标 | Request service rate | Stall time (额外 stall cycle) |
| Alone 性能估计方式 | 给 application 最高优先级，在低干扰下直接测量 | 在受干扰的状态下，试图计数 interference cycle 来反推 |
| 对 MLP 的鲁棒性 | 高——最高优先级下干扰本身就很小，无需建模 MLP | 低——interference 被 MLP 隐藏时计数不准 |
| Non-memory-bound 处理 | 引入 α (stall fraction) 区分 compute/memory phase | 无此机制 |
| 平均估计误差 | 8.1% | 29.8% |

与 **AlwaysPrioritize [Iyer et al., SIGMETRICS'07]** 的区别：AlwaysPrioritize 始终优先 AoI，不能控制优先程度，会不必要地减慢其他 application。MISE-QoS 只分配 "刚好够" 的 bandwidth 给 AoI。

与 **ATLAS [Kim et al., HPCA'10] / TCM [Kim et al., MICRO'10]** 的区别：两者目标是 system throughput/fairness，但不显式估计 slowdown。TCM 的 strict ranking 会破坏 low-ranked application 的 row-buffer locality，导致高 max slowdown。

## 实验评估

### 实验设置

- **仿真平台**：In-house cycle-accurate DDR3-SDRAM simulator + in-house cycle-level x86 simulator (Pin frontend)，模拟 out-of-order core with limited-size instruction window。
- **硬件规格配置**：
  - Processor: 4-16 cores, 5.3GHz, 3-wide issue, 8 MSHRs, 128-entry instruction window
  - LLC: 64B cache line, 16-way associative, 512KB private cache per core
  - Memory Controller: 64/64-entry read/write request queues
  - DRAM: DDR3-1066 (8-8-8), 1 channel, 1 rank/channel, 8 banks/rank, 8KB row-buffer
  - Address mapping: Row-interleaving
- **Workload**：26 benchmarks from SPEC CPU2006，使用 PinPoints 提取 representative phase，运行 200M cycles。300 个 4-core multiprogrammed workload（12 个 mix × 26 个 AoI benchmark × 不同 memory intensity 组合）。
- **对比 Baseline**：
  - STFM [MICRO'07]
  - ATLAS [HPCA'10]
  - TCM [MICRO'10]
  - FRFCFS [Rixner et al., ISCA'00]
  - AlwaysPrioritize [Iyer et al., SIGMETRICS'07]
  - EqualBandwidth（均分带宽）
- **Metrics**：Harmonic Speedup（兼顾 throughput 和 fairness）、Maximum Slowdown（衡量 unfairness）、Weighted Speedup。

### 关键结果

1. **Slowdown 估计精度**：MISE 在 300 个 4-core workload 上的平均估计误差为 **8.1%**，而 STFM 为 **29.8%**，精度提升约 3.7 倍。对 non-memory-bound application（如 povray 0.1% vs STFM 56.3%，calculix 1.3% vs 43.5%），MISE 的优势尤为显著。

2. **MISE-QoS 有效性**：在 3000 个 data point 中，MISE-QoS **满足 slowdown bound 的比例为 80.9%**（vs AlwaysPrioritize 83%，即达到后者 97.5% 的满足率）。MISE-QoS **正确预测 bound 是否被满足的比例为 95.7%**，而 AlwaysPrioritize 完全没有预测能力。当 slowdown bound 为 10/3 时，MISE-QoS 的 harmonic speedup 比 AlwaysPrioritize **提升 12%**，weighted speedup 提升 10%，max slowdown 降低 13%。

3. **MISE-Fair 公平性**：在 16 核系统中，MISE-Fair 的 max slowdown 比 STFM（最佳 prior work）**降低 7.2%**。随核数增加，MISE-Fair 的优势更明显——因为简单优先最慢 application（STFM 做法）在多核下不够有效，而 MISE-Fair 通过全局 bandwidth redistribution 更好地平衡所有 application。

4. **用 STFM 驱动 QoS 机制的对比**：使用 STFM 的 slowdown 估计来驱动 MISE-QoS 的 bandwidth 分配机制，bound met and predicted right 的比例仅 63.7%（vs MISE 78.8%），预测错误率 18.4%（vs MISE 4.3%）。slowdown bound 为 10/3 时，STFM-QoS 的系统性能比 MISE-QoS 低 5%。

### 结果分析

**Sensitivity Analysis**（Section 7, Table 3）：
- **Interval length**：过小（1M cycles）时误差极高（~64%），因为 request service rate 在短时间内不稳定。5M cycles 以上误差基本稳定在 8-11%。最终选择 5M cycles，兼顾精度和 fine-grained QoS enforcement。
- **Epoch length**：过大（1M cycles）时部分 application 可能在一个 interval 内完全得不到最高优先级，无法估计 ARSR。过小时差异不大。10K cycles 效果最佳。
- 最优参数组合：interval = 5M cycles, epoch = 10K cycles, 平均误差 8.1%。

**效果好的场景**：
- Memory-intensive workload 中 MISE-QoS 收益最大（因为 AlwaysPrioritize 对此类 workload 的干扰代价最高）。
- Non-memory-bound application 的 slowdown 估计中 MISE 远优于 STFM。

**效果不佳的场景**：
- gromacs 的 slowdown 被 MISE 略微低估，导致 MISE-QoS 在很紧的 bound 下误判 bound 已满足。
- 对部分 benchmark（如 sphinx3），STFM 的估计与 MISE 准确度接近。
- 模型未考虑 bank-level parallelism (BLP) 和 row-buffer interference 对 ARSR 估计的影响（论文承认这是 limitation，留作 future work）。

## 审稿人视角

### 优点

1. **模型极其简洁优雅**：两个直觉清晰的 observation 构建出一个只需 24 bytes/thread 的轻量级 slowdown 估计模型。避免了 STFM 那种需要精确建模 MLP/OoO execution 的复杂性，转而通过"给最高优先级直接测量"这一聪明的方式回避了最难的建模问题。这种 "observation-driven, estimation-by-measurement" 的思路是论文最大的亮点。

2. **从模型到应用的完整故事**：不仅提出估计模型，还基于模型构建了 MISE-QoS 和 MISE-Fair 两个实用的 memory scheduling 机制，并进行了系统的评估。QoS 机制的 "刚好够" bandwidth 分配思想，比 AlwaysPrioritize 在系统效率上的提升是显著的且实用的。

3. **评估方法论扎实**：300+ workload、3000 data points、多维度 sensitivity analysis、与多个 state-of-the-art baseline 的对比。对 STFM 既做了定性分析（为什么不准）又做了定量对比。Figure 2/3 中逐 interval 跟踪估计值与 actual slowdown 的对比非常有说服力。

4. **Observation 1 有真实系统验证**：在 Intel Core i7 上验证了 request service rate 与 performance 的线性关系，不只是仿真结论。

### 不足

1. **系统模型过于简化**：
   - 只有 1 channel、1 rank——现代系统通常是多 channel 多 rank，cross-channel/cross-rank 的 interference pattern 会更复杂。Observation 2 在多 channel 场景下是否仍然成立（给最高优先级时的干扰是否仍然足够小）需要验证。
   - LLC 是 private 的，完全隔离了 shared cache interference，这使得问题退化为纯 memory interference 场景。实际系统中 shared LLC 的 interference 会与 memory interference 耦合。
   - Row-interleaving 的 address mapping 使得 bank-level contention 特别严重，有利于展示 interference 的效果，但不代表所有现实 mapping 下的情况。

2. **Lottery scheduling 的带宽分配精度**：2% 的固定增减步长是经验值，论文承认更好的自适应方法是 future work。在 workload phase 快速变化的场景下，这种缓慢的线性调整可能跟不上。

3. **模型对 BLP 和 row-buffer interference 的忽略**：论文明确指出未考虑 BLP 和 row-buffer interference 对 ARSR 估计的影响。在高 BLP 的 application（如 streaming access pattern）中，给最高优先级时的 service rate 可能因为 BLP 与 alone 状态下的 BLP 不同（因为 queue 中请求的分布不同）而产生偏差。

4. **α (stall fraction) 的估计粗糙**：对于 OoO 处理器，α 被简单定义为 ROB head 等待 memory request 的 cycle 比例。这实际上低估了 memory 对性能的真实影响——很多非 ROB-head-stall 的 memory 延迟也会间接影响性能（如 dispatch stall、dependent instruction chain 的延迟等）。Threshold 的选择也未充分讨论。

5. **缺少对 prefetching 的讨论**：现代处理器普遍使用 hardware prefetcher。Prefetch request 的优先级处理、prefetch 对 request service rate 的扰动、以及 prefetch accuracy 对模型准确度的影响，论文完全未涉及。

6. **Workload 的时代局限性**：仅使用 SPEC CPU2006 单线程 benchmark 进行 multiprogrammed workload 组合，没有真正的多线程/共享内存 workload（如 PARSEC）。对于共享数据的多线程 application，memory access pattern 和 interference 特征可能截然不同。

### 疑问或值得追问的点

- **最高优先级采样对其他 application 的 perturbation**：虽然 epoch 很短（10K cycles），但频繁地轮流给不同 application 最高优先级，是否会导致所有 application 的 row-buffer locality 被频繁打断？论文没有量化这种采样本身对系统性能的开销。
- **Observation 1 的普适性**：线性关系在 SPEC CPU2006 的几个 memory-bound benchmark 上成立，但对于 access pattern 更复杂的 workload（如 random vs. streaming、不同 BLP 特征），这个线性关系是否会偏离？尤其是当 row-buffer hit rate 随 contention 变化时，service rate 与 performance 的关系可能不再是简单的线性。
- **Scalability 问题**：16 核已经是论文评估的上限。在 64+ 核的系统中，每个 application 获得最高优先级的频率会大幅降低（epoch 数量有限），ARSR 的采样精度是否会显著下降？
- **与 memory channel/bank partitioning 的交互**：论文声称 partitioning 技术与 MISE 互补，但如果 application 已经被 partition 到不同的 channel/bank，interference 本身就被大幅减少，MISE 的价值是否会降低？两者的协同效果缺乏评估。
