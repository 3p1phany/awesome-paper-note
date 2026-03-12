---
title: "DReAM: Dynamic Re-arrangement of Address Mapping to Improve the Performance of DRAMs"
authors: "Mohsen Ghasempour, Aamer Jaleel, Jim D. Garside, Mikel Luján"
venue: "MEMSYS 2016"
year: 2016
---

# DReAM: Dynamic Re-arrangement of Address Mapping to Improve the Performance of DRAMs

## 基本信息
- **发表**：MEMSYS (International Symposium on Memory Systems), 2016
- **单位**：University of Manchester (英国) & NVIDIA

## 一句话总结
> 通过运行时监测物理地址各 bit 的变化率（近似 entropy），动态生成 workload-specific 的 address mapping 以减少 page conflict。

## 问题与动机

现代 DRAM memory controller 使用**固定的 address mapping scheme** 将物理地址映射到 DRAM 内部层次结构（channel/rank/bank/row/column）。然而，不同 workload 的 memory access pattern 差异巨大，且同一 workload 的访问模式也会在运行时动态变化。固定的 address mapping 无法适应这种动态性，导致大量不必要的 **page conflict**（同一 bank 内连续访问不同 row），严重损害 DRAM 性能。

现有的缓解手段主要是 **memory request scheduler**（如 FR-FCFS）通过 reorder 命令来减少 page conflict，但 scheduler 受限于其可见的 request 数量（由 queue depth、线程数、数据依赖等决定），存在根本性的优化上限。另一条路径是软件层面（OS memory allocator），但面对多应用并发执行和虚拟化场景时复杂度极高，且依赖于针对特定硬件的编译。

论文用实验数据支撑了这一动机：在 BIOBENCH、COMMERCIAL、HPC、PARSEC、SPEC 五类 benchmark suite 上评估了三种经典 address mapping scheme（标准 row-buffer locality mapping、Zhang et al. 的 permutation-based page interleaving [MICRO 2000]、Kaseridis et al. 的 minimalist open-page scheme [MICRO 2011]），结果表明**没有任何一种固定映射在所有 workload 上都表现最优**，证实了 workload-adaptive mapping 的必要性。

## 核心方法

### 关键思路

DReAM 的核心 insight 是：**物理地址每个 bit 在连续 memory request 之间的变化频率（bit-change rate）与 DRAM 性能之间存在强相关性**（实验测得相关系数 0.89，p-value = 1.97×10⁻¹⁵）。基于此，DReAM 监测每个物理地址 bit 的变化率，将变化率低的 bit 分配给 row address（减少 row switching / page conflict），变化率中等的分配给 bank address，变化率高的分配给 column address（利用 locality）。这本质上是一种基于 entropy 近似的 address bit 重排策略。

### 技术细节

#### 1. Address Mapping Estimation（地址映射预测）

**硬件需求极简**：每个物理地址 bit 对应一个 counter、一个 history register 保存上一次访问地址、一组 XOR 门检测相邻两次访问的 bit 变化。具体流程：

- 对于每次 memory request，将当前地址与上一次地址逐 bit XOR；
- 若某 bit 发生变化，对应 counter 递增；
- 在一个 time window（基于时钟周期数或 request 数量）结束后，counter 数组构成一个 **bit-change pattern（signature）**；
- 根据 signature 排序：变化率最低的 bit 映射为 row address，中等的映射为 bank address，最高的映射为 column address；
- DReAM 限制自己**不重排 column address bits**，以避免 cache-line 级别的数据迁移开销。

**触发条件**：新映射方案只有在能将 bit-change rate 相比 baseline 改善超过一个可编程阈值（实验中为 7%）、且在连续多个 time window 中持续满足该条件（Consistency Threshold）时，才会被启用。

**Rollback 机制**：启用新映射后，DReAM 继续监测。若新映射不再优于 baseline，则触发 rollback，将已迁移的 row 恢复到原始位置，切换回原映射。系统最多同时管理两种映射方案，rollback 完成后可引入第三种。

#### 2. Data Migration（数据迁移）

改变 address mapping 后，已在 DRAM 中的数据位置与新映射不一致，需要数据迁移。论文提出两种方案：

**Scenario 1 - Offline Data Migration**：
- 适用于行为稳定的应用场景（如数据库系统）；
- 在 Region of Interest（ROI）期间监测访问模式，估计最优映射；
- 将新映射保存到 memory controller，重启时通过 BIOS 选择使用；
- 无需运行时迁移开销，但需要一次重启。

**Scenario 2 - Online Data Migration**：
- 运行时 on-demand 迁移，**仅在 row 被访问时才迁移**（lazy migration）；
- 每个 row 用 2 bit 状态追踪：Migration-Bit（是否已迁移到新位置）和 Swap-Bit（是否被 swap）；
- 全局维护 Migration Table（MT）和 Swap Table（ST）；
- 访问流程：
  - 用 PAMS（原映射）和 EAMS（新映射）同时翻译地址；
  - 若 row 未迁移（两个 bit 均为 0）：用 PAMS 访问，然后将 row 迁移到 EAMS 指向的位置；若目标位置已被占用，执行 swap 而非 chain migration；
  - 若已迁移：直接用 EAMS 访问；
  - 若已被 swap：通过 reverse mapping 追溯实际位置（可能需要多次追溯直到找到 swap-bit = 0 的位置），访问后再迁移到正确位置。

**DRAM 侧硬件依赖**：
- 利用 Seshadri et al. [MICRO 2013] 的 RowClone 进行 bulk data copy；
- 利用 Kim et al. [ISCA 2012] 的 MASA（Multiple Activated Subarrays）机制实现 subarray 级并行；
- 论文认为**只有 inter-bank migration 有意义**（intra-bank 迁移不减少 page conflict），因此仅执行跨 bank 迁移。

**迁移延迟开销**：row buffer 大小 4 Kbit/device，64-bit I/O bus，单次 row transfer 需 64 memory clock cycles，worst-case swap 需 128 memory clock cycles（= 512 CPU clock cycles，假设 4:1 频率比）。论文假设迁移期间处理器 stall（最悲观估计）。

#### 3. Mapping-Sensitive vs. Mapping-Insensitive 分类

论文将 workload 分为两类：
- **Mapping-sensitive**：row address 区域内存在高变化率 bit，且可以与低变化率的非 row bit 交换（如 libquantum，bit 14 变化率极高但被映射到 row space）；
- **Mapping-insensitive**：row address 区域内的 bit 变化率已经是最低的，没有交换余地（如 stream）。

### 与现有工作的区别

| 维度 | DReAM | Zhang et al. (Permutation-based) [MICRO 2000] | Kaseridis et al. (Minimalist Open-Page) [MICRO 2011] | PARDIS [ISCA 2012] |
|------|-------|-----------------------------------------------|------------------------------------------------------|---------------------|
| 映射方式 | 运行时动态生成 workload-specific mapping | 固定的 XOR-based bank interleaving | 固定的 XOR + column 重排 | 可编程 MC，但映射通过 offline profiling 获得 |
| 自适应性 | 完全自适应，支持 rollback | 无 | 无 | 需离线分析 |
| 数据迁移 | 提出 on-demand lazy migration + swap 机制 | 不涉及 | 不涉及 | 不涉及 |
| 硬件开销 | 每 bit 一个 counter + XOR + MT/ST 表 | 仅 XOR 门 | 仅 XOR 门 | 完整可编程 MC |

DReAM 与 scheduler-based 方案（STFM、PAR-BS、ATLAS 等）是正交的：scheduler 优化 command reordering，DReAM 优化 data placement，两者可叠加使用。

## 实验评估

### 实验设置
- **仿真平台**：USIMM（Utah Simulated Memory Module），经修改以支持三种 address mapping 和完整 DReAM 架构
- **硬件规格配置**：
  - CPU 3.2 GHz，ROB size 32
  - Memory bus 800 MHz，1 channel，1 rank/channel，8 banks/rank，65536 rows/bank
  - 4 GB DRAM，cache line 64 Byte
  - Scheduling: FR-FCFS
- **Workload**：48 个单线程 workload（来自 PARSEC、SPEC CPU2006、BIOBENCH、HPC、COMMERCIAL）+ 20 个随机组合的多线程 workload mix（4-thread 和 8-thread）
- **对比 baseline**：Permutation-based Page Interleaving（三种固定映射中表现最优的）
- **DReAM 阈值**：bit-change rate 改善 > 7% 才触发 online 模式

### 关键结果

1. **DReAM-Offline** 平均优于 permutation-based baseline **5%**，最高达 **28%**（跨所有 workload）；
2. **DReAM-Online** 在满足阈值的 12 个 workload 上平均优于 baseline **4.5%**，最高达 **23%**；
3. 按 mapping sensitivity 分类：mapping-sensitive workload 平均改善 **9%**，mapping-insensitive workload 平均改善 **2%**；
4. 对于 BIOBENCH 和大部分 PARSEC workload，baseline mapping 已足够好，DReAM 不会触发切换（DReAM-Online 对性能无损，DReAM-Offline 约 1% noise 级退化）；
5. **libquantum** 获得最显著的性能提升（约 28%），因其 bit 14 变化率极高却被映射到 row space；
6. 多线程 workload mix 中 DReAM 依然有效（GMEAN 约 3-5% 改善）；
7. **Storage overhead** 极低：对于 512 GB DRAM，mapping estimation 仅需 ~60 bytes，data migration 的 MT/ST 仅占 DRAM 容量的 3×10⁻⁵%；
8. 12 个触发 online migration 的 workload 中，仅 ferret（10% rollback）和 libquantum（39% rollback）需要 rollback；87.5% 的数据迁移为 inter-bank。

### 结果分析

论文将 DReAM 定位为一种"保险策略"（insurance policy）：当固定映射不适合当前 workload 时，DReAM 能检测并切换到更好的映射；当固定映射已经足够好时，DReAM 的阈值机制会阻止不必要的切换，避免性能退化。libquantum 的 case study 很好地说明了 DReAM 的工作原理：该 workload 的 bit 14 在 row address space 内频繁变化导致大量 page conflict，DReAM 将其重新映射到 bank/column space，把 page conflict 转化为 bank-level parallelism。

## 审稿人视角

### 优点

1. **核心 idea 简洁优雅**：通过 bit-change rate 近似 entropy 来指导 address mapping 重排，硬件开销极低（每 bit 一个 counter），概念清晰且易于实现。这种"用最简单的信号做最有用的事"的设计哲学值得赞赏。

2. **问题定义清晰，动机充分**：Figure 4 的 cross-benchmark 性能对比有力地证明了固定映射的局限性；Figure 6 的 48 个 workload 的 bit-change pattern 可视化非常直观，让读者能直接"看到"不同 workload 的 mapping sensitivity 差异。

3. **完整的系统设计**：不仅提出了 mapping prediction 算法，还认真处理了 data migration 这个实际部署中绑死的难题，提供了 offline 和 online 两种方案，并设计了 rollback 机制防止 degradation loop。工程完整性较好。

4. **评估覆盖面广**：48 个单线程 + 20 个多线程 workload，涵盖 5 个 benchmark suite，且对 mapping-sensitive 和 mapping-insensitive workload 分别讨论，分析较为全面。

### 不足

1. **实验配置过于简化，缺乏说服力**：
   - 仅 1 channel、1 rank、单线程为主的配置，与实际系统差距巨大。现代服务器典型配置为 4-8 channel、2 rank/channel、16-32 banks/rank。在更高并行度的配置下，bank conflict 的 pattern 会根本性改变，DReAM 的 bit-change rate 信号是否仍然有效是一个重要疑问。
   - ROB size 仅 32，远小于现代处理器（通常 200+），这严重限制了 MLP（Memory-Level Parallelism），可能人为放大了 address mapping 的影响。

2. **Online migration 的 swap chain 追溯开销被低估**：
   - 论文描述当 row 被 swap 后，需要通过 reverse mapping 多次追溯找到实际位置。在 worst case 下这可能导致多次 DRAM 访问才能定位数据，但论文没有给出 swap chain 长度的分布分析，也没有讨论最坏情况对 tail latency 的影响。
   - 假设 CPU 在迁移期间完全 stall 是"最悲观估计"，但论文并未给出实际的 stall overhead 量化结果，也没有将 migration latency 纳入最终性能数字中（从文中描述看 DReAM-Online 的性能数字似乎已包含，但 offline 数字忽略了重启成本）。

3. **Bit-change rate 作为 mapping quality 的代理指标的局限性**：
   - 虽然报告了 0.89 的相关系数，但这是在特定配置下的统计结果，论文未讨论该相关性在不同系统配置（多 channel、多 rank）下是否仍然成立。
   - 更根本地，bit-change rate 是一个全局统计量，无法捕捉 **temporal clustering** 等局部行为。例如，两个 workload 可能有相同的全局 bit-change rate 分布，但一个的 page conflict 集中在 burst 阶段，另一个均匀分布，它们需要不同的优化策略。

4. **与 scheduler 的交互效果未评估**：
   - 论文使用 FR-FCFS 作为唯一 scheduler。FR-FCFS 本身已有一定的 page conflict 缓解能力（优先 row-hit request）。如果换用更激进的 scheduler（如 PAR-BS、BLISS），DReAM 的增益是否会被大幅压缩？这是一个关键的 sensitivity analysis 缺失。

5. **Time window 大小和 consistency threshold 的 sensitivity 分析缺失**：
   - 论文选择了 250K memory requests 作为 sampling window，7% 作为 bit-change rate improvement threshold，但未给出这些参数的 sensitivity analysis。这些参数的选择直接影响 DReAM 的响应速度和稳定性之间的 trade-off。

6. **DRAM 侧修改的可行性存疑**：
   - Online migration 依赖 RowClone 和 MASA，这两项技术至今（2016 年论文发表后至今）都未被主流 DRAM 厂商采纳。这使得 Scenario 2 更多停留在理论层面。Scenario 1 虽然不需要 DRAM 修改，但 applicability 有限（仅适合长期稳定运行的应用）。

### 疑问或值得追问的点

- 在多 channel/多 rank 配置下，不同 channel 的 bit-change pattern 可能不同。DReAM 是否需要 per-channel 独立的 mapping？如果是，address mapping 的组合空间爆炸如何处理？
- 论文仅限制了"不重排 column bits"，但 bank bits 和 row bits 的重排同样可能导致大量 inter-bank migration。是否应该引入更 fine-grained 的重排约束来控制迁移量？
- Bit-change rate 本质上是一个一阶统计量（相邻两次访问的差异）。是否考虑过更高阶的统计信息（如 bit-change 的 auto-correlation）来提升预测质量？
- DReAM 对 address mapping 的"优化"本质上是一种 greedy sorting（按 change rate 从低到高分配给 row → bank → column）。是否存在 non-greedy 的最优分配方案，或者能否证明 greedy 是最优的？
- 在 DReAM 的 online 模式下，mapping 切换期间（部分 row 已迁移，部分未迁移）的 mixed state 对 scheduler 的决策质量有何影响？这个过渡态可能持续很长时间（因为是 lazy migration），期间的性能行为值得深入分析。
