---
title: "A Software Memory Partition Approach for Eliminating Bank-level Interference in Multicore Systems"
authors: "Lei Liu, Zehan Cui, Mingjie Xing, Yungang Bao, Mingyu Chen, Chengyong Wu"
venue: "PACT 2012"
year: 2012
---

# A Software Memory Partition Approach for Eliminating Bank-level Interference in Multicore Systems

## 基本信息

- **发表**：PACT (International Conference on Parallel Architectures and Compilation Techniques), 2012
- **单位**：中国科学院计算技术研究所，计算机体系结构国家重点实验室
- **备注**：Lei Liu 与 Zehan Cui 为共同第一作者

## 一句话总结

> 通过修改 OS 物理页分配（page-coloring），将 DRAM bank 按 core/thread 独占划分，以纯软件方式消除多核系统中 bank 级内存干扰。

## 问题与动机

在多核平台上，所有 core 共享 DRAM 内存系统，导致严重的 **memory contention 和 interference** 问题，具体表现为：

1. **Interference**：单线程通常有良好的 row buffer locality，但多核环境下不同线程的请求交织访问同一 bank，导致 row buffer hit rate 大幅下降。论文引用数据表明，row buffer hit rate 从 1 core 的 60%+ 降至 16 core 的 35%。
2. **Unfairness**：传统 MC 调度（如 FR-FCFS）偏向 row buffer locality 好的 memory-intensive 应用，导致 non-intensive 应用 slowdown 可达 7.74X（4-core）甚至 11.35X（8-core），而 intensive 应用仅 1.04X/1.09X。

**现有硬件调度方案的不足**：TCM、ATLAS、PAR-BS 等算法虽然有效，但需要修改 memory controller 硬件——增加复杂调度逻辑、额外存储（如 TCM 需要 4Kbits 额外存储支持 24 核）、以及线程行为监控机制。工业界对采纳这些激进的硬件调度算法持保守态度。

论文提出的核心问题是：**能否用纯软件方法达到与这些硬件调度方案类似的效果？**

## 核心方法

### 关键思路

论文的核心 observation 是：**单个线程所需的有效 bank 数量是有限的**（实验表明 8~16 个 bank 即可达到 90% 以上的峰值性能），这是因为单核受 memory dependency、高 cache hit rate 和有限 MSHR 数量的制约，无法产生足够多的并发 memory request 来充分利用所有 bank。然而现有系统采用 bank-interleaved address mapping 让每个程序访问所有 bank，这不但没有性能收益，反而引入了不必要的 inter-thread bank conflict。

基于此 observation，论文提出通过 OS 层面的 page-coloring 将 bank 按 core/thread 独占划分，从而让 MC "被动地"以 thread-cluster 的方式调度请求。

### 技术细节

#### 1. Bank Color Bits 的确定

物理地址中存在若干位同时决定了 OS page index 和 DRAM bank index，这些位称为 **bank color bits**。在论文的实验平台上（Intel i7-860, 8GB DDR3, 64 banks），共有 5 个 bank color bits：bit 13~15 和 bit 21~22，因此共 2^5 = 32 种 bank color，每个 color 对应 2 个 bank（跨 dual-channel 绑定）。

其中 bit 13~15 与 LLC（8MB 16-way）的 cache set index 重叠，因此 BPM 天然地同时实现了 **cache partition**（2^3 = 8 个 cache color）和 **bank partition**（额外 2 bit 扩展到 32 个 bank color）。这使得 BLP 可以与 cache partition 正交。

#### 2. 软件探测地址映射算法（Algorithm 1）

由于 MC 的地址映射策略可由 BIOS 配置且不固定，论文提出了一个三步软件探测算法：

- **Step 1 (Detect Row Bits)**：利用 row buffer miss 延迟远高于 hit 延迟的特征，逐 bit 翻转地址位并测量 uncached 访问延迟，高延迟组为 row address bits。
- **Step 2 (Detect Column Bits)**：利用 BLP（两个不同 bank 的并发访问）延迟低于单 bank 内 row buffer conflict 延迟的特征，从剩余位中区分 column bits 和 bank bits。
- **Step 3 (Detect XOR Policy, Optional)**：检测 MC 是否使用 XOR 混合策略，通过配对测试 `<u,v>` 位组合来判断。

该算法通过 HMTT（硬件内存总线监控工具）验证了正确性，可嵌入 OS boot 阶段。

#### 3. Linux 内核实现

在 Linux 2.6.32.15 内核中修改 buddy system：对每个 order 的 free page list，按 32 个 bank color 重新组织为 **32 个 colored free list**（层次化结构）。每个 process 拥有自己的 color 集合。page fault 时，内核搜索对应 color 的 free list 进行分配，对应用完全透明。

- **Multi-programmed workloads**：bank color 静态分配给各程序
- **Multi-threaded workloads**：提供新的内核 API 让程序员感知 bank color 并主动映射
- **开销**：平均 kernel time overhead 仅 0.3%

#### 4. 分配策略

论文采用 **static uniform partition**：4-programmed workload 中每个程序 8 个 color（16 banks/2GB）；8-programmed workload 中每个程序 4 个 color（8 banks/1GB）。

#### 关键设计权衡（Trade-off）

- **BLP 损失 vs. interference 消除**：bank partition 减少了单个线程可用的 bank 数量（从 64 降至 8~16），牺牲了一定 BLP，但由于单核实际能利用的 bank parallelism 有限（受 MSHR 等约束），这个损失很小（Figure 1 显示 16 banks 已能达到 90%+ 的峰值性能），而消除 inter-thread interference 带来的收益更大。
- **Cache partition 副作用**：由于 bank color bits 与 cache set bits 部分重叠（3 bits），BPM 同时引入了 cache partition，这虽然有助于消除 cache-level interference，但也限制了每个线程可用的 cache way 数量。
- **静态分配 vs. 动态调整**：论文仅采用静态均匀分配，没有根据线程实际内存需求动态调整 bank 分配，这在 memory intensity 差异大的 workload 中可能不够优化。

### 与现有工作的区别

1. **vs. TCM [Kim et al., MICRO 2010]**：TCM 是硬件调度方案，在 MC 中按 memory-intensive/non-intensive 分组并采用不同调度策略，需要额外 4Kbits 存储和复杂逻辑；BPM 是纯软件方案，无需任何硬件修改，通过 OS 层面消除 bank-level interference。
2. **vs. Mi et al. [NPC 2010] 和 Jeong et al. [HPCA 2012]**：之前的 bank partition 工作仅在仿真环境验证，未部署在真实硬件上；且未考虑 cache partition 与 bank partition 的关系。BPM 首次在真实机器上实现并评估了 bank-level partition。
3. **vs. Channel Partition [Muralidhara et al., MICRO 2011]**：channel partition 按 channel 划分线程数据，但 channel 数量有限（通常 2~4），且不适用于 cache line interleaving across channels 的系统；BPM 在 bank 粒度划分，颗粒度更细，适用范围更广。

## 实验评估

### 实验设置

- **真实硬件平台**（非仿真）：
  - Intel Core i7-860, 4 核 (Hyper-Threading 支持 8 线程), 2.8GHz
  - 8MB 16-way shared LLC
  - 8GB DDR3, 64 banks (每 bank 125MB)
  - 5 bank bits → 32 colors, 每 color 对应 2 banks (dual-channel)
- **OS**：CentOS 5.4, 修改后的 Linux kernel 2.6.32.15
- **功耗测量**：自研 DIMM wrapper card 硬件工具，可精确测量内存功耗
- **Workload**：
  - SPEC CPU2006, 随机生成 20 组 multi-programmed workloads（10 组 4-programmed + 10 组 8-programmed）
  - PARSEC 2.1 中的 streamcluster（multi-threaded）
  - ref input size (SPEC), native input (PARSEC)
- **Baseline**：未修改的标准 Linux kernel（bank-interleaved address mapping）
- **Metrics**：Weighted Speedup (WS) 衡量 throughput，Maximum Slowdown (MS) 衡量 fairness

### 关键结果

1. **System Throughput**：BPM 平均提升 Weighted Speedup 4.7%（最高 8.6%），所有 workload 均无性能退化。8-programmed workloads 平均提升 5.3%，优于 4-programmed 的 4.1%，表明干扰越严重 BPM 效果越好。
2. **Fairness**：BPM 平均降低 Maximum Slowdown 4.5%（最高 15.8%）。对于包含 stream-like 高 memory-intensive 应用（如 libquantum, RBL=99.22%）与 non-intensive 应用混合的 workload，公平性提升尤为显著。
3. **Row Buffer Miss Rate**：BPM 使整体 row buffer miss rate 降低约 10%（以 workload_11 为例，从近 50% 显著下降）。
4. **功耗**：BPM 配合 open-page policy 可节省内存系统功耗 5.2%，因为减少了高功耗的 activate 操作。
5. **Page Policy**：BPM 使多核系统下的 open-page policy 重新变得有效，open-page + BPM 相比 close-page 提升 6.3% weighted speedup。
6. **Cache vs. Bank Partition**：仅 cache partition（3 bits, 8 colors）提升 throughput 3.1%、fairness 3.4%；加入额外 2 bit 形成 32 bank colors 后进一步提升至 4.7%、4.5%，证明 bank partition 与 cache partition 正交且有额外收益。
7. **Multi-threaded**：streamcluster 在 4/8 线程下分别提升 1.7%/2.3%，低于 multi-programmed workloads，因为线程间存在较多共享数据。

### 结果分析

- **性能改善指标**：论文发现 **Sum(BW) × Stdev(RBL)** 是预测 BPM 收益的最佳指标——该值越大（即总带宽需求高且线程间 row buffer locality 差异大），BPM 改善越明显。其他单一指标（Average(RBL), Sum(BW), Stdev(RBL)）均无法很好地匹配 BPM 的改善趋势。
- **负面情况**：包含 429.mcf（MPKI=99.8）的若干 workload 出现了 fairness 轻微恶化，原因是 mcf 从 64 banks 降至 16 banks 几乎不损失性能（仅 2%），导致 BPM 对 mcf 的改善反而超过了 non-intensive 应用，产生了反向不公平。
- **Per-core Bandwidth Sensitivity**：通过降频（1333→800 MHz）模拟不同 per-core bandwidth，发现 bandwidth 越低 BPM 改善越大（负相关），表明 BPM 对未来 many-core 平台（per-core bandwidth 更低）有更好的前景。

## 审稿人视角

### 优点

1. **实用性强，工程价值高**：纯软件方案，无需任何硬件修改，在真实机器上实现并评估（而非仿真），这在 memory scheduling 领域以硬件方案为主流的背景下极具吸引力。Linux 内核的实际实现也证明了方案的可部署性。
2. **Key Observation 扎实**：单线程所需有效 bank 数量有限这一发现有坚实的实验支撑（Figure 1，23 个 SPEC2006 benchmark），为 bank partition 的可行性提供了令人信服的理论基础。
3. **Algorithm 1 的通用性**：提出的地址映射探测算法不依赖特定平台手册，可在任意机器上自动发现 bank bits，包括检测 XOR mapping，具有较好的通用性和实用性。
4. **分析维度丰富**：论文不仅报告了 throughput/fairness，还涵盖了 page policy、功耗、cache partition 正交性、per-core bandwidth sensitivity、性能预测指标等多个维度，实验分析较为全面。

### 不足

1. **仅采用静态均匀分配，缺乏动态策略**：所有线程均分 bank color，完全忽视了线程间 memory intensity 的巨大差异。论文自身数据已显示 mcf（MPKI=99.8）和 namd（MPKI=0.3）对 bank 数量的需求天差地别，但仍然给它们分配相同数量的 bank。一个根据 runtime profiling 动态调整 bank 分配的策略是显而易见的改进方向，但论文仅作为 future work 提及。
2. **Workload 规模和多样性不足**：仅 20 组随机生成的 multi-programmed workloads + 1 个 multi-threaded benchmark（streamcluster），对于声称"practical"的方案而言，缺乏更多 real-world server workloads 的验证。Multi-threaded 场景仅用了一个 data-parallel 的程序，没有验证 producer-consumer、pipeline 等更复杂的共享模式。
3. **与硬件调度方案缺乏直接比较**：论文的 motivation 是替代 TCM 等硬件方案，但实验中没有与任何硬件调度算法（哪怕是 simulated 的）进行直接性能比较。4.7% 的 throughput 提升相比 TCM 等方案的改善幅度（通常 10%+）可能存在差距，但缺乏数据支撑。
4. **Scalability 讨论不充分**：仅在 4-core/8-thread 平台上实验。随着核数增加到 16、32 甚至更多，每个 core 分到的 bank 数量会急剧减少，此时单线程 8~16 banks 足够的 observation 是否仍然成立值得质疑——尤其是当 bank 总数不变但核数增加时。
5. **Cache Partition 副作用未充分讨论**：BPM 由于 bank bits 与 cache set bits 重叠，强制引入了 cache partition，对于 cache-sensitive 但 memory-non-intensive 的 workload，这种 cache 容量缩减可能造成性能损失，但论文未深入分析这一 trade-off。
6. **OS Swap 被禁用的假设较强**：论文禁用了 OS swap 并假设服务器有足够内存。但当 bank partition 限制了每个线程的物理内存范围后，在内存压力较大的场景下可能导致 OOM 或频繁的 color 内页面回收，这一问题未被讨论。

### 疑问或值得追问的点

- 论文中 bank color 静态分配给 process，那么当一个 core 上发生 context switch 运行不同 process 时，新 process 如何获得 bank color？是否需要 page migration？这对实时性和 TLB shootdown 的影响如何？
- 论文的 bank partition 实际上也消除了 rank-level interference（因为 bank 天然分布在不同 rank 上），但论文未讨论 rank switching overhead 的影响——在消除 bank sharing 后，MC 的 rank switching 行为是否也发生了变化？
- Sum(BW) × Stdev(RBL) 这个指标的发现很有意思，但它是在线程独立运行时采集的数据计算得出的，实际共享运行时的 BW 和 RBL 会相互影响，该指标在 online 场景下的预测准确度需要进一步验证。
- 论文仅在 Intel 平台上实验，AMD 平台（不同的 memory controller 架构和地址映射策略）上的效果如何？Algorithm 1 是否能正确处理更复杂的 interleaving 策略（如 fine-grained channel interleaving + bank hashing）？
