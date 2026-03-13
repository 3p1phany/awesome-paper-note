---
title: "Prefetch-Aware DRAM Controllers"
authors: "Chang Joo Lee, Onur Mutlu, Veynu Narasiman, Yale N. Patt"
venue: "MICRO 2008"
year: 2008
---

# Prefetch-Aware DRAM Controllers

## 基本信息
- **发表**：MICRO 2008（第41届 IEEE/ACM International Symposium on Microarchitecture）

## 一句话总结
> 根据 prefetch accuracy 动态调整 DRAM controller 的 scheduling 优先级和 buffer 管理策略，最大化有用 prefetch 收益、最小化无用 prefetch 危害。

## 问题与动机

这篇论文要解决的核心问题是：**DRAM controller 如何智能地处理 prefetch request 与 demand request 之间的优先级关系**。

现有 DRAM controller 在处理 prefetch request 时采用两种刚性策略，各有致命缺陷：

1. **demand-prefetch-equal**（如 FR-FCFS [Rixner, ISCA 2000]）：不区分 demand 和 prefetch，统一按 row-hit 优先 + FCFS 调度。当 prefetch 准确率低时（如 galgel 69%、ammp 94%、xalancbmk 91% 的 prefetch 是无用的），无用 prefetch 与 demand 竞争 bandwidth 和 buffer entry，导致严重性能损失——milc 在该策略下 prefetching 反而降低 36% 性能。

2. **demand-first**：始终优先 demand request。当 prefetch 准确率高时（如 libquantum 接近 100%），这种策略忽略了 DRAM access latency 的非均匀性——一个 row-hit prefetch（~12.5ns）被迫排在 row-conflict demand（~37.5ns）后面，浪费了 DRAM throughput。libquantum 在 demand-first 下只获得 60% 提升，而 demand-prefetch-equal 下可获得 169% 提升。

**关键洞察**：没有任何一种刚性策略在所有 workload 上都最优。性能取决于 prefetch 的有用性（usefulness），而这在不同程序、不同执行阶段差异巨大（如 milc 的 prefetch accuracy 在 0%~90% 之间剧烈波动）。

此外，即使 demand-first 策略也无法完全消除无用 prefetch 的危害，因为无用 prefetch 仍然会：
- 占用 memory request buffer (MRB) entry，阻止有用请求进入
- 消耗 DRAM bandwidth
- 导致 cache pollution（替换有用数据）

## 核心方法

### 关键思路

PADC 的核心 idea 是**将 prefetch accuracy 作为运行时反馈信号，驱动 DRAM controller 同时在两个维度上做自适应调整**：(1) scheduling priority——在 prefetch 准确时将其提升为与 demand 同等优先级以最大化 DRAM throughput，不准确时降级；(2) buffer management——主动丢弃长期滞留在 MRB 中的可能无用的 prefetch request，释放资源给有用请求。

关键 observation 有两个：
- 有用 prefetch 因为会被后续 demand hit（在 MRB 中转为 demand），其平均 service time 显著短于无用 prefetch（milc 中有用 prefetch 平均 1486 cycles vs 无用 2238 cycles），因此"老"的 prefetch 更可能是无用的。
- Prefetch accuracy 具有强烈的 phase behavior（如 milc 在 150M-275M cycles 期间 accuracy 接近 0%），需要动态适应。

### 技术细节

PADC 由三个核心组件构成：Prefetch Accuracy Measurement、Adaptive Prefetch Scheduling (APS)、Adaptive Prefetch Dropping (APD)。

#### 1. Prefetch Accuracy Measurement

每个 core 维护以下硬件结构：

- **Prefetch (P) bit**：每个 L2 cache line 和 MRB entry 各 1 bit。MRB entry 的 P bit 在 prefetch 生成时置位，当同地址 demand 命中该 entry 时清零（转为 demand）。Cache line 的 P bit 在 prefetch 填入时置位，demand hit 时清零。
- **Prefetch Sent Counter (PSC)**：16-bit per core，记录发出的 prefetch 总数。
- **Prefetch Used Counter (PUC)**：16-bit per core，记录有用 prefetch 数（prefetched cache line 被 demand hit，或 MRB 中 prefetch 被 demand 匹配）。
- **Prefetch Accuracy Register (PAR)**：8-bit per core，存储上一 interval 的 accuracy = PUC / PSC。

每 100K cycles 为一个 interval，interval 结束时更新 PAR，重置 PSC 和 PUC。

#### 2. Adaptive Prefetch Scheduling (APS)

APS 将请求分为 **critical** 和 **non-critical** 两类：
- 若某 core 的 prefetch accuracy ≥ **promotion_threshold**（85%），该 core 的所有 prefetch 被标记为 critical（与 demand 同优先级）
- 否则，该 core 的 prefetch 为 non-critical，demand 优先

调度规则（按优先级递减）：

| 优先级 | 规则 | 含义 |
|--------|------|------|
| 1 | **Critical (C)** | Critical request（demand + 有用 prefetch）优先于 non-critical |
| 2 | **Row-hit (RH)** | Row-hit 优先于 row-conflict |
| 3 | **Urgent (U)** | 低 accuracy core 的 demand 优先于其他请求 |
| 4 | **FCFS** | 老请求优先于新请求 |

**Urgent request 的设计动机（重要 trade-off）**：当高 accuracy core 的 prefetch 被提升为 critical 后，其 demand + prefetch 的总量会压倒低 accuracy core 的 demand-only 请求，导致低 accuracy core 的 demand 饥饿。Urgent 规则通过 boost 低 accuracy core 的 demand 来缓解这种不公平。实验表明（Case Study III），不使用 urgent 时 omnetpp 的 individual speedup 仅为 0.206，使用后恢复至 0.352；unfairness 从 4.547 降至 1.845；HS 从 0.410 提升至 0.524。

#### 3. Adaptive Prefetch Dropping (APD)

APD 的核心依据：**老的 prefetch request 更可能是无用的**。有用 prefetch 因为会被 demand 命中而提前服务，平均驻留时间短；无用 prefetch 则长期滞留。

机制：若某 prefetch request 在 MRB 中的驻留时间超过 **drop_threshold**，则将其移除。Drop 前需先 invalidate 对应的 MSHR entry，防止后续 demand 匹配到已移除的 prefetch。

drop_threshold 是动态的，根据 prefetch accuracy 分 4 级：

| Prefetch Accuracy | drop_threshold (cycles) |
|---|---|
| 0% - 10% | 100 |
| 10% - 30% | 1,500 |
| 30% - 70% | 50,000 |
| 70% - 100% | 100,000 |

设计理由：accuracy 极低时（接近 0%），几乎所有 prefetch 都无用，需要快速移除（100 cycles）；accuracy 高时，需要保守以避免误杀有用 prefetch（100K cycles）。

**APS 与 APD 的正向交互**：APS 将低 accuracy core 的 prefetch 标记为 non-critical，自然延迟其服务，使其在 MRB 中驻留更长时间，从而更容易被 APD 捕获并移除。

#### 4. 硬件开销

每个 MRB entry 新增字段：U (1 bit)、P (1 bit)、ID (log₂N bits)、AGE (10 bits，每 100 cycles 递增一次)。C、RH、FCFS 字段已存在于 baseline demand-first FR-FCFS 中。

4-core 系统总开销仅 34,720 bits (~4.25KB)，其中 P bit per cache line 占 ~95%（32,896 bits）。若处理器已有 prefetch bit，额外开销仅 ~228B。

### 与现有工作的区别

| 方案 | 调度策略 | 是否考虑 prefetch usefulness | Buffer 管理 |
|------|---------|---------------------------|------------|
| FR-FCFS [Rixner, ISCA'00] | Row-hit > FCFS，demand = prefetch | 否 | 无 |
| Demand-first [Lin, HPCA'01 等] | Demand > Prefetch | 否（刚性优先） | 无 |
| FDP [Srinath, HPCA'07] | 不改变 DRAM 调度 | 在 prefetcher 端 throttle aggressiveness | 无（从源头减少） |
| **PADC (本文)** | 基于 accuracy 动态切换 | 是（per-core accuracy 驱动） | APD 主动丢弃老 prefetch |

与 FDP 的关键差异值得强调：FDP 在 prefetcher 端 throttle aggressiveness（减少发出的 prefetch 数量），而 APD 在 DRAM controller 端 drop 已发出的无用 prefetch。APD 的优势在于：(1) FDP 在新 phase 开始时增加 aggressiveness 较慢，可能错过有用 prefetch；(2) APD 始终保持 prefetcher aggressive，只在 MC 端过滤，不会因 throttle 而丢失有用 prefetch。实验显示 APS+APD (PADC) 优于 APS+FDP。

## 实验评估

### 实验设置

- **仿真平台**：Cycle-accurate x86 CMP simulator（非公开模拟器），忠实建模 port contention、queuing effect、bank conflict 以及 DDR3 DRAM timing constraints
- **硬件规格配置**：
  - 单核/4核/8核 CMP
  - 每核：15-stage OoO，256-entry ROB，32-entry LSQ，L1 I/D 32KB 4-way，L2 512KB (单核1MB) 8-way 8-bank
  - DRAM：DDR3 1333MHz，16B bus，8 banks，4KB row buffer，tRP/tRCD/CL = 15ns
  - MRB：64/128/256 entry（1/4/8核）
  - Stream prefetcher：32 streams，degree 4，distance 64
  - Memory controller：单核/4核/8核各 1 个 controller（8核也测了 2 controller）
- **Workload**：
  - 55 个 SPEC CPU 2000/2006 benchmarks，ICC/IFORT -O3 编译，reference input，200M representative instructions（Pinpoints 选取）
  - 单核：全部 55 个
  - 4核：32 个随机组合的 multiprogrammed workload
  - 8核：21 个随机组合
- **对比 baseline**：
  - no-prefetch
  - demand-first（FR-FCFS + demand 优先 prefetch）—— 主 baseline
  - demand-prefetch-equal（标准 FR-FCFS）
  - APS-only（仅自适应调度，不 drop）
  - FDP [Srinath, HPCA'07]（Feedback Directed Prefetching）
  - 还测试了 stride、C/DC、Markov 三种其他 prefetcher

### 关键结果

1. **单核性能**：PADC 在 55 个 SPEC benchmark 上平均提升 IPC **4.3%**（相对 demand-first baseline），同时降低 bus traffic **10.4%**。APS 单独贡献 3.6% 提升，APD 额外贡献 0.7%。

2. **4核系统性能**：32 个 workload 平均，PADC 提升 weighted speedup **8.2%**（相对 demand-first 和 demand-prefetch-equal 两者均为 8.2%），降低 bus traffic **10.1%**。PADC 在 32 个 workload 中 31 个优于两种刚性策略，唯一退化 workload（vpr+gamess+dealII+calculix）WS 仅降低 1.2%。

3. **8核系统性能**：
   - 单 controller：WS 提升 **9.9%**，bandwidth 降低 **9.4%**。注意在 8 核下刚性策略反而使 prefetching 整体降低性能（bandwidth 太稀缺），而 PADC 仍能从 prefetching 中获益。
   - 双 controller：WS 仍提升 **5.5%**，bandwidth 降低 **13.2%**。

4. **与 FDP 的对比**（4核）：demand-first+FDP WS 提升 1.7%，demand-first+APD 提升 2.9%；APS+FDP 提升 7.4%，APS+APD (PADC) 提升 8.2%。PADC 优于所有 FDP 组合。

### 结果分析

论文通过三个 case study 深入分析了 PADC 在不同 workload composition 下的行为：

- **全 prefetch-friendly**（swim+bwaves+leslie3d+soplex）：PADC WS 提升 31.3%（vs demand-first），主要收益来自 APS 提升 DRAM throughput，APD 贡献较小（仅 0.9% bandwidth 节省）但通过 drop leslie3d/soplex 的少量无用 prefetch 将 prefetch coverage 从 56% 提升至 73%。

- **全 prefetch-unfriendly**（art+galgel+ammp+milc）：PADC WS 提升 17.7%，HS 提升 21.5%，bandwidth 降低 9.1%。APD 大幅消除无用 prefetch，使 WS/HS 恢复到接近 no-prefetch 水平（差距仅 2%/1%）。

- **混合 workload**（omnetpp+libquantum+galgel+GemsFDTD）：PADC 通过 APD 消除 omnetpp 和 galgel 的 67%/57% 无用 prefetch，释放资源给 libquantum 和 GemsFDTD，总 bandwidth 降低 14.5%。

**核心不足场景**：对于 prefetch-insensitive（class 0）或非 memory-intensive 的 workload 组合，PADC 收益有限甚至微弱退化（如 vpr+gamess+dealII+calculix，WS 降 1.2%）。

论文还展示了 SPL（Stall cycles Per Load）指标，PADC 单核平均降低 SPL 5.0%，直观反映了 DRAM service 效率的提升。

**Sensitivity 方面**：论文测试了 4 种不同 prefetcher（stream、stride、C/DC、Markov），PADC 在所有 prefetcher 上均提升性能并降低 bandwidth，证明了方法的通用性。Promotion threshold 选择 85%，drop threshold 分 4 级，但论文未提供这些参数的 sensitivity analysis 曲线。

## 审稿人视角

### 优点

1. **问题定义精准且实际**：论文准确识别了 DRAM controller 层面 prefetch 处理的 fundamental trade-off——row-buffer locality 最大化 vs demand 优先保护。Figure 1 和 Figure 2 的 motivating example 非常有说服力，用 10 个 benchmark 清晰地展示了两种刚性策略各自的失败场景，这种"两头都不好"的论证方式很有力。

2. **APS 和 APD 的协同设计很优雅**：两个机制不是简单叠加，而是有正向交互——APS 延迟非 critical prefetch 的服务，使其在 MRB 中驻留更久，为 APD 的 age-based dropping 创造条件。这种 co-design 体现了对问题的深入理解。

3. **Urgent request 机制考虑了 fairness**：在 multi-core 场景下，简单的 critical/non-critical 二分法会导致低 accuracy core 被饥饿。论文通过 Table 6 的定量分析清晰展示了 urgent 机制的必要性（unfairness 从 4.547 降至 1.845），这在 2008 年 fairness 刚开始受到关注的背景下是前瞻性的。

4. **极低的硬件开销**：4.25KB（含 prefetch bit），若已有 prefetch bit 则仅 228B。对于 DRAM controller 级别的优化，这个开销几乎可以忽略。

5. **实验全面**：55 个单核 benchmark、32+21 个多核 workload 组合、4 种 prefetcher、与 FDP 的对比、公平性分析、单/双 controller 配置，覆盖面很广。

### 不足

1. **Prefetch accuracy 作为唯一决策信号过于粗粒度**：整个 PADC 的调度和 dropping 决策完全基于 per-core 的 prefetch accuracy，这是一个全局聚合指标。同一个 core 在同一 interval 内可能对不同 memory region 有截然不同的 prefetch 行为（如部分 stream 非常准确，部分 stride 很差），per-core accuracy 无法区分。更细粒度的信号（如 per-stream 或 per-bank accuracy）可能带来更好的效果。

2. **Threshold 参数缺乏 sensitivity analysis**：promotion_threshold = 85% 和 4 级 drop_threshold 的具体数值选择缺乏系统的 sensitivity study。论文直接给出了这些"magic numbers"但未展示参数扫描结果，读者无法判断这些值是精心调优的最优点还是相对鲁棒的选择。这对实际部署的信心有影响。

3. **APD 的 age-based dropping 机制存在误杀风险**：论文假设"老的 prefetch 更可能无用"，但这个假设在某些场景下不成立——例如当 MRB 拥塞导致所有请求（包括有用 prefetch）都老化时，APD 可能错误地丢弃有用 prefetch。论文未讨论这种 corner case 的影响。

4. **Workload 的代表性问题**：全部使用 SPEC CPU 2000/2006 单线程 benchmark 的 multiprogrammed 组合，缺少真正的 multi-threaded workload（如 PARSEC、SPLASH-2）。Multi-threaded 程序中 prefetch 和 demand 的交互模式可能与 multiprogrammed 有本质不同（如 shared data 上的 prefetch 冲突）。

5. **与 prefetch-first 策略的对比不充分**：论文仅在脚注 3 中提到 prefetch-first 策略（平均降 5.8%），但未将其纳入主实验图表，也未分析其在高 accuracy workload 上是否有优势场景。

6. **单一 memory controller 架构假设**：论文的 PADC 设计假设 centralized MC 有 global view，但未讨论在 distributed MC（如 NUMA 系统中每个 node 有独立 MC）或多 channel interleaving 场景下的适用性。虽然测试了 8 核双 controller，但两个 controller 是独立工作的，没有跨 controller 的协调。

### 疑问或值得追问的点

- **Interval 长度（100K cycles）的选择是否最优？** 过短可能导致 accuracy 估计噪声大，过长则无法跟踪快速 phase change。milc 的 phase 在 ~50M cycles 级别变化，100K cycles 的 interval 看似足够细粒度，但其他 workload 可能不同。
- **APD drop 的 prefetch 是否会导致后续重新 fetch？** 论文提到 drop 前 invalidate MSHR，但如果该地址后来确实被 demand 访问，它将成为一个额外的 L2 miss——这种"false drop"的代价是否被量化？
- **在现代 HBM/DDR5 多 channel、多 bank group 的系统中，PADC 的 effectiveness 如何变化？** 更高的 bandwidth 可能降低 prefetch interference 的严重性，但更复杂的 timing constraint（bank group 切换）可能引入新的交互。
- **promotion_threshold 是否应该 per-core 动态调整而非全局固定？** 不同 core 运行不同程序，其 prefetch accuracy 的分布特征可能需要不同的 threshold。
