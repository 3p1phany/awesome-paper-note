---
title: "CAMPS: Conflict-Aware Memory-Side Prefetching Scheme for Hybrid Memory Cube"
authors: "Muhammad M. Rafique, Zhichun Zhu"
venue: "ICPP 2018"
year: 2018
---

# CAMPS: Conflict-Aware Memory-Side Prefetching Scheme for Hybrid Memory Cube

## 基本信息

- **发表**：ICPP (International Conference on Parallel Processing), 2018
- **单位**：University of Illinois at Chicago
- **DOI**：https://doi.org/10.1145/3225058.3225112

## 一句话总结

> 利用 HMC 内部 TSV 大带宽和 logic layer，基于 row buffer conflict 感知和 row utilization 信息做 memory-side prefetching 决策与 buffer 管理。

## 问题与动机

**核心问题**：Memory wall 问题在 many-core + big-data 时代愈发严重，传统 DDRx 系统的 off-chip 带宽和延迟瓶颈限制了 prefetching 的效果。

**为什么 HMC 是一个好的切入点**：

- HMC 的 3D 堆叠架构通过 TSV 提供了巨大的**内部带宽**（高达 320 Gbps），memory-side prefetching 可以利用这一内部带宽搬运数据到 prefetch buffer，而**不占用外部 serial link 带宽**。
- HMC 的 logic layer 提供了放置 prefetch buffer 和控制逻辑的物理空间，每个 vault 功能独立，可以各自管理 prefetching。
- Prefetched data 不主动推送到 cache，避免了 cache pollution，只在被请求时才 push，实质上将 "prefetching" 转化为 "demand fetching"。

**现有工作的不足**：

1. Core-side prefetching 缺乏对 memory bank 状态（row buffer 状态、channel 可用性）的感知，aggressive prefetching 容易浪费带宽并造成网络拥塞。
2. 已有的 memory-side prefetching 方案（如 MMD [Yedlapalli et al., PACT'13]）虽然可以动态调整 prefetch degree，但其 prefetch 决策没有考虑 **row buffer conflict** 信息，且 prefetch buffer 管理仅使用简单的 **LRU** 策略，未考虑 row 的 utilization。
3. Baseline 方案（每次访问都 prefetch 整行）过于激进，accuracy 低，能耗高。

## 核心方法

### 关键思路

**Key Observation 1**：频繁引发 row buffer conflict 的 row 如果被 prefetch 到 buffer 中，后续对这些 row 的访问可以直接从 buffer 命中，从而从根源减少 conflict。

**Key Observation 2**：Row 的 utilization（同一 row 内被访问的不同 cache line 数量）是判断该 row 是否值得 prefetch 的重要信号——高 utilization 的 row prefetch 收益更大。

**Key Observation 3**：Prefetch buffer 替换不应仅看 recency，还应考虑 utilization——高 utilization 且近期被访问的 row 应保留更久。

### 技术细节

整个方案由两部分组成：**Conflict-Aware Prefetching (CAMPS)** 和 **Utilization & Recency Based Prefetch Buffer Management (CAMPS-MOD)**。

#### 1. Conflict-Aware Prefetching

核心数据结构：

- **Row Utilization Table (RUT)**：每个 vault 16 个 entry（对应 16 个 bank），每个 entry 20 bits。记录当前 row buffer 中 open 的 row 被访问了多少个不同的 cache line（utilization counter）。
- **Conflict Table (CT)**：每个 vault 32 个 entry，fully associative，所有 bank 共享。记录最近被替换出 row buffer 的 row 的 utilization 信息。用于检测哪些 row 反复被换进换出（即频繁引起 conflict）。

**Prefetch 决策流程**（参考论文 Figure 3）：

1. 请求到达 vault controller，先查 **prefetch buffer**：
   - **Hit**：直接从 buffer 服务，更新 utilization counter 和 recency counter。
   - **Miss**：进入下一步。
2. 查 **row buffer**：
   - **Row buffer hit**：正常服务；将该 row 的 utilization 信息写入 RUT（若不存在）；每次 hit 递增 RUT 中的 utilization counter。
     - **若 utilization counter 超过阈值（实验中设为 4）**：将整行 prefetch 到 prefetch buffer，并 precharge bank。释放 RUT entry。
     - **否则**：保持 row open，继续累积 utilization。
   - **Row buffer miss**：precharge bank，activate 新 row，正常服务。
     - **同时检查新 row 是否在 CT 中有 entry**：
       - **是**（说明该 row 最近刚被替换出过，又再次被访问 → 发生了 row buffer conflict）：将该 row prefetch 到 buffer，从 CT 中移除该 entry，precharge bank。
       - **否**：保持 row open，将之前被替换出的 row 的 RUT entry 移入 CT（CT 满则 LRU 淘汰）。

**设计上的关键 trade-off**：

- Utilization 阈值（=4）的选择：太低会导致 over-prefetching（低 utilization 的 row 也被 prefetch），太高则可能错过有价值的 row。论文选择 4 作为经验值。
- CT 大小（=32 entries）：决定了能追踪多少历史 conflict 信息。全 vault 共享意味着 hot bank 可以占用更多 entry。

#### 2. Utilization & Recency Based Prefetch Buffer Management

Prefetch buffer 配置：每个 vault 16KB，fully associative，1KB line（即 15 个 entry，因为 row buffer size = 1KB）。

每个 prefetch buffer entry 维护两个 counter：

- **Utilization Counter (UC)**：记录该 row 中被访问的不同 cache line 数量。
- **Recency Counter (RC)**：最近被访问的 row RC 设为 15（= buffer entry 数），所有 RC 大于该 row 之前 RC 值的 entry 递减 1。最久未访问的 row RC = 0。

**替换决策流程**（参考论文 Figure 4）：

1. 优先替换 **utilization 已达最大值**的 row（所有 cache line 都已被访问过，说明数据已全部传给 processor，无需保留）。
2. 若无此类 row，计算每个 entry 的 **UC + RC 之和**，替换和最小的 row。
3. 若存在多个和相同的 row，替换 **UC 最低**的（即优先保留 utilization 高的 row）。

**设计直觉**：高 utilization 说明 row 中仍有未被访问的 cache line 可能被后续请求命中；高 recency 说明 row 近期活跃。两者加权综合考虑优于单纯 LRU。

#### 3. 硬件开销

- RUT：16 entries × 20 bits × 32 vaults = 1.25 KB
- CT：32 entries × 20 bits × 32 vaults = 2.5 KB
- **总额外开销仅 3.75 KB**，相对于 prefetch buffer 本身（16KB × 32 = 512 KB）微不足道。
- Prefetch buffer 大小与对比方案一致，公平比较。

### 与现有工作的区别

| 方案 | Prefetch 决策 | Buffer 管理 | 关键区别 |
|------|-------------|-----------|---------|
| **BASE** | 每次访问都 prefetch 整行 | LRU | 最激进，accuracy 最低，无 conflict 感知 |
| **MMD** [Yedlapalli, PACT'13] | 根据 prefetched data 的 usefulness 动态调整 prefetch degree | LRU | 有自适应 degree，但不考虑 row buffer conflict；buffer 管理仅 LRU |
| **CAMPS/CAMPS-MOD** | 基于 utilization 阈值 + conflict 检测 双条件触发 | UC + RC 加权替换 | 唯一考虑 row buffer conflict 信息的方案；替换策略同时考虑 utilization 和 recency |

## 实验评估

### 实验设置

- **仿真平台**：gem5 全系统模拟器（cycle-accurate, x86 full system）
- **HMC 模型**：基于 [HMC Spec 2.1] 建模，包含 vault controller、serial links、crossbar switch、external HMC controller；DRAM 操作基于 Hansson et al. (ISPASS'14) 和 Jung et al. (VLSI-SoC'14) 的模型
- **处理器**：8 cores @ 3GHz, issue width = 4, OoO, x86 ISA
- **Cache 层次**：L1 I/D 32KB private 2-way (2 cycles), L2 256KB private 4-way (6 cycles), L3 16MB shared 16-way (20 cycles), 64B line
- **HMC 配置**：8 DRAM layers, 32 vaults, 2 banks/vault-layer → 共 512 banks; 1KB row buffer
- **DRAM 参数**：DDR3-1600, tRCD=tRP=tCL=11 cycles, R/W queue size = 32
- **Serial Links**：4 links, 16 I/P + 16 O/P lanes, 12.5 Gbps, full duplex
- **Prefetch Buffer**：16KB/vault, fully associative, 1KB line, hit latency = 22 cycles
- **Scheduling**：FR-FCFS, Open Page policy
- **Address Mapping**：RoRaBaVaCo (Row-Rank-Bank-Vault-Column)

- **Workload**：SPEC CPU2006 组成 12 组 8-core multiprogramming workloads
  - **HM (High Memory Intensive, MPKI ≥ 20)**：HM1-HM4（bwaves, gems, gcc, lbm, milc, mcf, sphinx, omnetpp 等）
  - **LM (Low Memory Intensive, 1 ≤ MPKI < 20)**：LM1-LM4（cactus, bzip2, astar, wrf, tonto, zeusmp, h264ref 等）
  - **MX (Mixed)**：MX1-MX4（从 HM 和 LM 各取 4 个）
- **模拟方法**：fast-forward 2B instructions → warmup 100M instructions → detailed simulation 800M instructions

- **对比 Baseline**：
  - **BASE**：每次 row 首次访问就 prefetch 整行
  - **BASE-HIT**：read queue 中有 2+ hits 的 row 才 prefetch 整行
  - **MMD**：动态调整 prefetch degree + LRU buffer 管理 [Yedlapalli, PACT'13]
  - **CAMPS**：本文 conflict-aware prefetching（不含 buffer 管理优化）
  - **CAMPS-MOD**：CAMPS + utilization/recency-based buffer 管理

### 关键结果

1. **整体性能提升**：CAMPS-MOD 平均比 BASE 提升 **17.9%**，比 BASE-HIT 提升 **16.8%**，比 MMD 提升 **8.7%**（geometric mean of IPC）。
2. **高内存密集型 workload 收益最大**：HM 平均提升 24.9%（vs BASE）、21.8%（vs MMD）；LM 平均提升 9.4%（vs BASE）、4.9%（vs MMD）；MX 介于两者之间。
3. **Row buffer conflict 降低**：CAMPS-MOD 平均减少 row buffer conflict 率 **16.3%**（vs BASE-HIT）和 **13.6%**（vs MMD）。
4. **Prefetching accuracy**：CAMPS-MOD 平均 accuracy **70.5%**，分别比 BASE、BASE-HIT、MMD 高出 33.3%、28.4%、4.1%。注意 CAMPS（无 buffer 管理优化）accuracy 比 MMD 低 1.5%，这正是引入 CAMPS-MOD 的动机。
5. **AMAT 降低**：CAMPS-MOD 平均比 BASE 降低 AMAT **26%**，比 MMD 降低 **16.3%**。
6. **能耗降低**：CAMPS-MOD 比 BASE 节能 **8.5%**，比 MMD 节能约 **2.5%**，主要因为更少的 activation/precharge 操作。

### 结果分析

- **HM workloads 收益最大**的原因很直观：off-chip 访问越频繁，prefetch buffer hit 的概率越高，减少 row buffer conflict 的累积收益越大。
- **CAMPS（无 buffer 优化）accuracy 略低于 MMD**：这说明 conflict-aware prefetching 本身更激进（会 prefetch conflict row），但如果 buffer 管理不当，利用率不够高。加入 UC+RC 替换策略后 accuracy 大幅提升，说明 buffer 管理优化是关键互补组件。
- 论文**缺乏 sensitivity analysis**：未对 utilization 阈值（=4）、CT 大小（=32）、prefetch buffer 大小（=16KB）等关键参数做扫参分析。

## 审稿人视角

### 优点

1. **问题意识清晰，HMC 内部带宽利用的动机合理**。利用 TSV 内部带宽做 row 粒度的 prefetch 而不占用外部 serial link，是一个很自然且有效的 insight。将 prefetch buffer 放在 vault controller 内、每个 vault 独立管理，也与 HMC 的 vault-based 架构天然契合。
2. **Conflict-aware prefetching 是一个有价值的 idea**。通过 CT 追踪被替换出 row buffer 的 row 并在其再次被访问时触发 prefetch，直接从根源减少 row buffer conflict，比单纯依赖 access pattern 或 prefetch degree 调整更有针对性。
3. **硬件开销极小**（3.75 KB）且实现逻辑简单（计数器+查表），工程可行性高。
4. **Buffer 替换策略的设计有理有据**：优先淘汰已全部被访问过的 row、然后用 UC+RC 加权是比 LRU 更合理的策略，实验也验证了其有效性。

### 不足

1. **实验平台和参数建模偏简化，缺乏说服力**：
   - HMC 模型是基于 gem5 的 "simplified yet accurate" 模型，但没有详细说明 TSV 带宽竞争、crossbar contention、vault-to-vault 通信等关键微架构细节是如何建模的。Prefetch buffer hit latency = 22 cycles 这个数字的来源也未说明。
   - DRAM 参数使用 DDR3-1600，而 HMC 内部 DRAM 的时序参数（尤其 tRCD/tRP）在 3D 堆叠下可能不同于标准 DDR3，这可能影响结果的准确性。

2. **关键设计参数缺乏 sensitivity analysis**：
   - Utilization 阈值为什么是 4？CT 大小为什么是 32？Prefetch buffer 16KB/vault 是否是最优？这些参数对性能的影响完全没有讨论。对于一篇提出新的 prefetching 策略的论文，参数敏感性分析是基本要求。

3. **Workload 和系统配置的局限性**：
   - 仅使用 SPEC CPU2006 8-core multiprogramming，缺少真正的多线程 workload（如 PARSEC）和新一代 benchmark（如 SPEC CPU2017）。对于 HMC 这种面向高带宽场景的技术，缺少带宽密集型和 data-intensive workload 是明显缺陷。
   - 8 core 的规模偏小，HMC 更适合 many-core 场景（如 GPU、accelerator），论文未讨论 scalability。

4. **Prefetch 粒度固定为整行（1KB）的合理性未充分论证**：
   - 虽然利用 TSV 带宽 prefetch 整行成本较低，但对于 spatial locality 不好的 workload，prefetch 整行仍可能浪费 buffer 空间。论文未考虑可变粒度 prefetch（如只 prefetch row 中 likely-to-be-accessed 的 cache line 子集）。

5. **与 core-side prefetcher 的交互完全未考虑**：
   - 实际系统中必然存在 core-side prefetcher（如 stride prefetcher、L2 stream prefetcher），memory-side prefetching 与 core-side prefetching 之间的协同或冲突未讨论。这是一个重要的系统级问题。

6. **论文写作质量一般**：
   - Abstract 末尾出现了明显的模板残留文字（"In this sample-structured document, neither the cross-linking..."），说明投稿前的检查不够仔细。
   - 部分表述重复，Section 2.4 Motivation 与 Section 1 Introduction 有大量内容重叠。

### 疑问或值得追问的点

- **CT 的 fully associative 共享设计是否合理？** 32 个 entry 被 16 个 bank 共享，对于 bank 间访问极不均匀的 workload，某些 bank 的 conflict 信息可能被其他 bank 挤出。是否考虑过 per-bank CT 或 set-associative CT？
- **Utilization 阈值 = 4 意味着一个 1KB row 中至少 4 个不同的 64B cache line 被访问才触发 prefetch**（row 内共 16 个 cache line）。这个阈值对不同 workload 的敏感度如何？是否可以做 adaptive threshold？
- **Prefetch buffer hit latency = 22 cycles 是否过高？** 如果 DRAM row buffer hit latency（tCL）仅 11 cycles，那么 prefetch buffer hit 反而比 row buffer hit 慢，这在部分场景下可能削弱 prefetching 的收益。论文未讨论这一点。
- **Dirty data 如何处理？** 论文只讨论了 read prefetch，对于 write-back 场景下 prefetch buffer 中 dirty data 的一致性维护完全没有提及。
- **BASE 方案中 "no row buffer conflicts" 的说法值得商榷**：即使每次 prefetch 整行后 precharge bank，如果两个连续请求访问同一 bank 的不同 row，在 prefetch 操作完成前仍可能产生 bank conflict（而非 row buffer conflict）。论文对此的区分不够清晰。
