---
title: "MITTS: Memory Inter-arrival Time Traffic Shaping"
authors: "Yanqi Zhou, David Wentzlaff"
venue: "ISCA 2016"
year: 2016
---

# MITTS: Memory Inter-arrival Time Traffic Shaping

## 基本信息
- **发表**：ISCA (International Symposium on Computer Architecture), 2016
- **作者单位**：Princeton University, Electrical Engineering Department

## 一句话总结
> 提出基于 memory request inter-arrival time distribution 的分布式源端流量整形机制，实现细粒度的 per-core 内存带宽分配与隔离。

## 问题与动机

### 核心问题
在多核/众核系统中，多个 core 共享 memory channel，缺乏对 off-chip memory bandwidth 进行 per-core/per-application/per-VM 粒度的精确 provisioning、scheduling 和 isolation 的能力。

### 为什么重要
1. **Core 数增长快于带宽增长**：片上 core 数量的增速远超 off-chip memory bandwidth 的增速，带宽成为瓶颈。
2. **IaaS Cloud 场景**：当前 IaaS 提供商（如 Amazon EC2）按 CPU-time 等价收费，缺乏按内存带宽需求差异化计费的能力。一个激进使用内存的 customer 可以抢占其他 customer 的 fair-share，造成性能波动。
3. **应用异质性**：不同应用对内存带宽的敏感度差异巨大——有的需要低延迟突发带宽，有的需要大平均吞吐，仅用单一速率限制无法满足多样需求。

### 现有工作不足
- **集中式 memory scheduler**（TCM、MISE、ATLAS、PAR-BS、Fair Queuing 等）：依赖 memory controller 端的集中式决策，硬件复杂度高（comparison tree、thread state table），扩展性差；且无法在 LLC 之前进行带宽限制，不能防止 aggressive application 污染 shared LLC。
- **Source Throttling（FST）**：虽然是源端控制，但仅控制发送速率，不控制 inter-arrival time 的分布，丢失了对 burstiness 的细粒度管理。
- **MemGuard**：区分 guaranteed 和 best-effort bandwidth，但不考虑系统 fairness，aggressive application 仍可抢占资源。
- **静态带宽分配**：无法适应应用的 phase 变化和 burstiness 特征。

## 核心方法

### 关键思路
**Key Observation**：应用的 memory access pattern 可以用 memory request inter-arrival time 的统计分布来刻画。该分布同时编码了 latency（inter-arrival time 本身）和 bandwidth（各 inter-arrival time 区间的请求频率）两个维度。即使两个应用的平均带宽相同，其 burstiness 可能截然不同——bursty traffic 的分布是多峰的，constant traffic 是单峰的。

**Core Idea**：在每个 core 或 LLC 侧放置一个简单的分布式硬件 traffic shaper，将应用的 memory traffic 整形为一个预设的 inter-arrival time distribution。用户/OS/hypervisor 可以配置这个目标分布，从而实现细粒度的带宽分配、隔离和差异化计费。

### 技术细节

#### 1. 分布式源端控制架构（Source Control）
- MITTS 模块放置在每个 core 侧（L1 cache 之后），而非 memory controller 端。
- 不依赖集中式硬件结构，天然支持 manycore 扩展。
- OS 或 hypervisor 通过 memory-mapped control register 配置每个 core 的 inter-arrival time distribution。

#### 2. Bin-based Bandwidth Shaper
核心硬件结构是一组 **traffic bin**（默认 N=10 个 bin）：
- 每个 bin_i 对应一个 inter-arrival time 区间 `[t_i - L/2, t_i + L/2)`，L=10 CPU cycles。
- 每个 bin 包含若干 **credit**（每个 credit 代表一次 memory transaction 的发送权限）。
- 发送 memory request 时，根据该请求与上一次请求的 inter-arrival time，从对应的 bin 扣减一个 credit。
- **Stalling 机制**：若对应 bin 无 credit 可用，则 stall 该 memory transaction，直到：(a) 等待足够长时间使其 inter-arrival time 匹配到更远的有 credit 的 bin；(b) 或 credit 被 replenishment。
- credit 上限为 1024（10-bit register per bin）。

#### 3. Credit Replenishment
- 定义 replenishment period `T_r = Σ(K_max × t_i)`，其中 K_max 是最大 bin size。
- 采用 "reset" 策略：每 T_r 周期将所有 bin 的 credit 重置为预设值 K_i（Algorithm 1）。
- 硬件上只需一个 T_r register、一个 counter T_c、一个 N-entry table K。

#### 4. 硬件放置的 Design Tradeoff
论文详细讨论了三种放置方案：
- **After L1**：将 L1 miss 当作 memory request。问题：shared LLC hit 也会被计入，不准确。
- **After LLC**：将 LLC miss 当作 memory request。问题：shared/distributed LLC 中难以实现 per-core tracking。
- **Hybrid（最终采用）**：MITTS 放在 L1 之后，但利用 LLC 反馈的 hit/miss 信息来修正 credit。

Hybrid 方案的两种具体实现：
- **Method 1（Speculate Hit, Rollback on Miss）**：乐观假设 L1 miss 是 LLC hit，LLC miss 时再扣 credit。需要 per-request timestamp table。slightly aggressive。
- **Method 2（Speculate Miss, Rollback on Hit，最终选择）**：保守假设 L1 miss 是 LLC miss，先扣 credit，LLC hit 时再加回。需要 per-request bin-number table。更简单，out-of-sync 更少。

#### 5. Global Traffic Management
- 支持 oversubscription 和 undersubscription。
- 问题：所有 core 可能同时花掉 bursty credit，导致瞬时带宽超出物理限制。
- 解决：memory controller 端放置 32-entry FIFO 吸收 burstiness。这是概率性极小事件。

#### 6. Online Genetic Algorithm 自动配置
- 搜索空间为 `K_max^10`，是一个非凸空间。
- 使用遗传算法（population=30, generations=20, EPOCH=20000 cycles）。
- 分为 CONFIG PHASE（多代遗传算法搜索最优 bin 配置）和 RUN PHASE（使用最优配置运行）。
- 支持 phase-based 重配置：在每个程序 phase 开始时重新运行 CONFIG PHASE。
- 利用 MISE 的 online profiling 技术测量 application slowdown。
- 软件 overhead 很小：runtime 仅被调用 20 次，每次约 5000 cycles。

### 与现有工作的区别

| 维度 | MITTS | TCM [MICRO'10] | MISE [HPCA'13] | FST [ASPLOS'10] |
|------|-------|---------------|---------------|-----------------|
| 控制位置 | 源端（Core/L1 之后） | Memory Controller | Memory Controller | 源端（Core） |
| 控制维度 | Inter-arrival time distribution | Application clustering + priority | Slowdown estimation + priority | 发送速率 |
| 带宽表达 | 多 bin 分布（bandwidth + burstiness） | 二元分类（high/low intensity） | 公平 slowdown | 单一 throttling rate |
| 可扩展性 | 分布式，O(N_core) | 集中式，需 ranking | 集中式，需 estimation | 分布式 |
| LLC 保护 | 可在 LLC 前限制流量 | 不能 | 不能 | 可以 |
| IaaS 计费 | 支持差异化 distribution 计费 | 不支持 | 不支持 | 不支持 |

## 实验评估

### 实验设置
- **仿真平台**：SDSim（基于 SSim core simulator 改编）+ DRAMSim2 集成；SDSim 由 GEM5 Alpha ISA full system simulator 驱动，支持 trace-driven 和 execution-driven simulation。
- **硬件配置**：
  - Core: 2.4GHz, 4-wide issue, 128-entry instruction window
  - L1: 32KB per-core, 4-way, 64B block, 8 MSHRs
  - L2: 64B line, 8-way; single-program 64KB, multi-program 1MB shared
  - Memory Controller: 32-entry transaction queue
  - DRAM: DDR3 1333MHz, 1 channel, 1 rank, 8 banks, 8KB row-buffer
- **Workload**：
  - Single-program: SPECint 2006 全套 + PARSEC + Apache + bhm Mail Server
  - Multi-program: 6 组混合 workload（4-program × 3 + 8-program × 3）
  - ROI: 200M cycles；Apache: 3000 requests, concurrency 10
- **对比 Baseline**：FR-FCFS, Fair Queue, TCM, FST (Source Throttling), MemGuard, MISE, 以及 Static Bandwidth Allocation

### 关键结果

1. **vs. Static Bandwidth Allocation**：在相同平均带宽（1GB/s）约束下，MITTS 平均获得 **1.18x speedup**（offline GA），mcf 和 omnetpp 分别达到 1.64x 和 1.68x。这证明了 distribution-based shaping 相比单一速率限制的优势。

2. **vs. Conventional Memory Schedulers（Throughput/Fairness）**：
   - 4-program workloads: 最佳改进为 throughput +17% / fairness +52%（workload 3）
   - 8-program workloads: 最佳改进为 throughput +12% / fairness +32%（workload 6）
   - 对比对象包括 FR-FCFS, Fair Queue, TCM, FST, MemGuard, MISE

3. **Hybrid MITTS+MISE**：MITTS 与 MISE 结合后，在 8-program workloads 上平均额外获得 **4% throughput + 5% fairness** 的增益，说明 MITTS 与集中式 controller 互补。

4. **IaaS Performance/Cost**：相比最优静态带宽分配，MITTS 的 performance-per-cost geometric mean 提升 **2.69x**，mcf 达到约 10x。

### 结果分析

- **为什么 MITTS 优于集中式 scheduler**：(1) 在 LLC 前限制流量，防止 aggressive app 污染 shared LLC；(2) 源端限流使 memory controller 端的 request scheduling window 更大、更有效；(3) 不同 app 可有不同的 inter-arrival distribution，允许 constructive inter-application effects。
- **Larger LLC (8MB)**：LLC miss 减少，但 MITTS 仍优于最佳 conventional scheduler，workload 1 的 throughput/fairness 改善为 5.3%/12.7%。
- **Bin 数量敏感性**：6-bin > 4-bin 约 10%，8-bin > 6-bin 约 5%，10-bin > 8-bin 约 2%，收益递减。
- **多线程应用**：shared MITTS（所有线程共享 credit）比 per-thread MITTS 好 2x 以上，因为 shared 方案允许 idle thread 的 credit 被其他 thread 使用。
- **Online vs. Offline GA**：Online GA 略逊于 offline GA，但能适应 phase 变化和 input set 变化，更实用。

## 审稿人视角

### 优点

1. **创新的抽象**：将 memory traffic 用 inter-arrival time distribution 来刻画是一个非常优雅的抽象，它同时编码了 bandwidth 和 burstiness 两个维度，比传统的 latency-sensitive / bandwidth-sensitive 二分法更具表达力。这个抽象也自然地引出了差异化计费的 IaaS 商业模型。

2. **真实硅片验证**：MITTS 不仅停留在仿真层面，而是在 25-core 32nm processor（OpenPiton）上完成了 Verilog 实现和 tapeout，面积仅 0.0035mm²（<0.9% core area）。这在体系结构论文中非常有说服力，RTL 代码还集成到了开源项目中。

3. **分布式 + 可组合**：MITTS 的源端分布式设计天然适合 manycore 扩展，且与集中式 scheduler（如 MISE）正交互补，hybrid 方案可叠加收益。

4. **全面的评估维度**：覆盖了 throughput、fairness、bandwidth isolation、economic efficiency (performance/cost)、phase adaptation、threaded workloads、bin 数量敏感性等多个维度，评估较为系统。

### 不足

1. **仿真配置的代表性存疑**：
   - 单通道、单 rank、DDR3 1333MHz 的配置过于简单，与 2016 年主流多通道配置差距大。多通道场景下，MITTS 的收益是否还能保持？inter-arrival time distribution 的语义在多通道下如何扩展？
   - 单程序实验使用 64KB L2 cache，这个配置在实际处理器中极为罕见。虽然论文补充了 8MB LLC 实验，但主要结论仍基于小 cache 配置。

2. **Genetic Algorithm 的 overhead 和收敛性讨论不足**：
   - CONFIG PHASE 需要 20 代 × 30 children × 20000 cycles/EPOCH，总计约 12M cycles 的配置开销。对于短运行 workload 或频繁 phase change 的应用，这个 overhead 可能不可忽略。
   - 论文未讨论 GA 的收敛稳定性——不同随机种子下结果差异多大？是否经常陷入 suboptimal？

3. **IaaS 定价模型过度简化**：
   - 假设 "1 core = 1.6GB/s bandwidth" 的定价换算过于粗糙，缺乏经济学论证。
   - Bin credit 的定价使用线性惩罚因子 `2 - (t_i / t_N)`，这个选择缺乏理论依据。在真实 Cloud market 中，定价应由供需决定，而非简单线性关系。
   - Performance/cost 的 2.69x 增益高度依赖于定价模型的选择，换一种定价函数结果可能大不相同。

4. **对 row-buffer locality 的影响缺乏分析**：MITTS 在源端改变了 memory request 的 arrival pattern，这必然影响 memory controller 端的 row-buffer hit rate 和 scheduling 效率。但论文几乎没有分析 MITTS 对 DRAM 端行为的影响（row-buffer hit/miss ratio、bank-level parallelism 变化等）。

5. **Replenishment 策略过于简单**："Reset" 策略意味着如果一个 replenishment period 内 credit 没用完就浪费了，无法 rollover。这对 bursty workload 可能不友好。论文没有讨论 rolling/accumulative replenishment 等替代方案的 tradeoff。

### 疑问或值得追问的点

- MITTS 整形后的 traffic pattern 对 DRAM scheduler（如 FR-FCFS）的 row-buffer locality 和 bank-level parallelism 有何影响？整形是否可能破坏原本有利的访问局部性？
- 在 SMT 场景下，同一 core 上多个 hardware thread 如何共享/分割 MITTS credit？论文只讨论了 per-thread vs. shared MITTS for multi-threaded apps，但未涉及 SMT。
- 10 个 bin、L=10 cycles 的参数选择对不同 workload 和不同 DRAM 配置（如 DDR4/DDR5、多 channel）的鲁棒性如何？
- 如果将 MITTS 的思路推广到 HBM 场景（多 channel、高带宽），distribution-based shaping 的意义是否会降低（因为带宽不再是瓶颈）？
- Online GA 的 CONFIG PHASE overhead 对 serverless / short-lived function 类 workload 是否构成致命问题？
