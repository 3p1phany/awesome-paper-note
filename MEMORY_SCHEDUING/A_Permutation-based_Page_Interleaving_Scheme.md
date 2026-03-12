---
title: "A Permutation-based Page Interleaving Scheme to Reduce Row-buffer Conflicts and Exploit Data Locality"
authors: "Zhao Zhang, Zhichun Zhu, Xiaodong Zhang"
venue: "ISCA 2000 (Proceedings of the 27th Annual International Symposium on Computer Architecture)"
year: 2000
---

# A Permutation-based Page Interleaving Scheme to Reduce Row-buffer Conflicts and Exploit Data Locality

## 基本信息
- **发表**：ISCA 2000（第27届国际计算机体系结构年会）
- **作者单位**：College of William and Mary, Department of Computer Science

## 一句话总结
> 通过对 L2 tag 低位与 bank index 做 XOR 生成新 bank index，同时消除 row-buffer conflict 并保留页内 spatial locality。

## 问题与动机

DRAM row-buffer conflict 是制约内存系统性能的关键瓶颈。当连续的访问请求命中同一 bank 的不同 row 时，每次访问都需要额外的 precharge + row access，延迟比 row-buffer hit 高 30%–50%。

现有的 interleaving scheme 在 concurrency 和 data locality 之间存在根本性的 trade-off：

- **Cache line interleaving**：将连续 cache line 分散到不同 bank，最大化 bank-level parallelism，但完全破坏了 row-buffer 的 spatial locality，导致极高的 row-buffer miss rate（19 个 benchmark 中 10 个超过 90%）。
- **Page interleaving**：以 page 为粒度分配 bank，保留了页内 locality，但当 L2 cache conflict miss 发生时，冲突的地址由于 set index 包含 bank index，必然映射到同一 bank 的不同 row，引发 row-buffer conflict。
- **Swapping scheme**（AlphaStation 600 使用）：交换 L2 tag 和 page offset 的部分 bit，能缓解部分 conflict，但代价是降低页内 spatial locality（连续地址块从 2^p 缩小到 2^(p-n)），且对部分程序反而增加 conflict。

论文的核心观察是：在 conventional page interleaving 下，**L2 conflict address 必然是 row-buffer conflict address**（前提是 L2 cache size / associativity > 所有 row-buffer 的累计容量）。这一结构性问题来自三个源头：L2 cache conflict miss、L2 writeback（dirty block 的读写地址映射到同一 set）、以及特定的内存访问模式（stride 为 row-buffer 总容量的倍数）。

## 核心方法

### 关键思路

核心 insight 是：L2 conflict address 之所以在 page interleaving 下产生 row-buffer conflict，是因为它们共享相同的 set index（其中包含 bank index），但有不同的 tag（其中包含 page index）。如果用 tag 的信息来"扰乱" bank index 的分配，就可以将原本映射到同一 bank 的冲突地址分散到不同 bank，同时因为 XOR 不改变 page offset，页内 locality 完全保留。

### 技术细节

#### 地址映射公式

设地址 A 的二进制表示为 $(a_{m-1} a_{m-2} \ldots a_0)$，传统 page interleaving 的 bank index 为 $I = (a_{k+p-1} \ldots a_p)$。Permutation-based scheme 的新 bank index $I'$ 为：

$$a'_i = a_i \oplus a_{m-t+i-p}, \quad \text{for } i = p, \ldots, k+p-1 \tag{1}$$

其中 $k = \log K$（bank 数量的 bit 数），$p = \log P$（page offset bit 数），$t = m - (s + b)$（L2 tag 的 bit 数）。实质上是将 L2 tag 的低 $k$ 位与原始 bank index 做 bitwise XOR。

#### 三大性质

1. **L2 conflict address 被分散到不同 bank**：两个 L2 conflict address 的 set index 相同但 tag 不同，只要 tag 低 $k$ 位不同，XOR 后就会产生不同的 bank index。论文用 16 bank 的例子展示了 4 个原本映射到同一 bank 的地址被分散到 4 个不同 bank。

2. **Spatial locality 完全保留**：同一 page 内的所有地址共享相同的 page index 和 bank index（XOR 的两个操作数对页内地址不变），因此映射后仍在同一 bank 的同一 row。

3. **Pages 均匀映射到所有 bank**：由于保留了原始 bank index 的信息，连续 page 仍然均匀分布在所有 bank 上。

#### 正确性证明（一一映射）

给定内存位置 $A'$，可以通过 $a_i = a'_i \oplus a'_{m-t+i-p}$ 唯一恢复原始地址 $A$，因为在现代系统中 $(s+b) > (k+p)$，XOR 的逆运算成立（公式 2）。

#### XOR 增加 conflict 的风险分析

论文指出 XOR 可能增加 conflict 的场景仅限于跨越 cache size 边界的特定地址对。由于 cache size 足够大，这些地址占整个地址空间的比例极小，实际风险可忽略。

#### 硬件开销

仅需 $k$ 个 XOR 门，延迟约 1 个 CPU cycle。在多级 cache 系统中，此操作不在 critical path 上，可与 cache 访问 overlap。

### 与现有工作的区别

| 方案 | 策略 | Locality | Conflict 处理 | 缺陷 |
|------|------|----------|--------------|------|
| Cache line interleaving | 以 cache line 粒度分 bank | 完全破坏 row-buffer locality | 最大化 bank parallelism | Miss rate 极高，依赖 close-page |
| Swapping scheme (AlphaStation 600) | 交换 L2 tag 与 page offset 的 $n$ bit | Locality 降低（连续块从 $2^p$ 缩至 $2^{p-n}$） | 将同 bank 不同 page 转为同 page | 存在 trade-off：swap 越多，conflict 越少但 locality 越差；对部分程序反而增加 conflict |
| **Permutation-based (本文)** | XOR L2 tag 低 $k$ 位与 bank index | **完全保留**页内 locality | 将同 bank 不同 page 转为**不同 bank** | 极少数跨 cache-size 边界地址可能增加 conflict |

关键差异在于：swapping scheme 试图将"同 bank 不同 page"的访问转化为"同 bank 同 page"（即 row-buffer hit），但代价是损失 locality；permutation-based scheme 则将其转化为"不同 bank"的并行访问，无 locality 损失。

## 实验评估

### 实验设置

- **仿真平台**：SimpleScalar 工具集（sim-cache 用于快速评估 miss rate，sim-outorder 用于全系统性能评估），包含 memory controller、bus contention、bank contention、DRAM precharge/refresh、processor/bus 同步。
- **处理器配置**：8-way superscalar，500 MHz，LSQ 大小 32，RUU 大小 64，最多 8 个 outstanding memory request。
- **Cache 层次**：L1 I/D cache 各 32KB 2-way 32B block（6ns hit），L2 cache 2MB 2-way 64B block（24ns hit）。
- **内存系统**：32 bytes / 83 MHz bus，4~256 bank，2~8 KB row-buffer，DRAM precharge 36ns，row access 36ns，column access 24ns。基于 Compaq XP1000 工作站参数。
- **Workload**：SPEC95 全套（SPECfp95 + SPECint95 共 18 个程序）+ TPC-C（基于 PostgreSQL 6.5）。
- **Baseline**：cache line interleaving（配合 close-page mode）、page interleaving（open-page）、swapping scheme（open-page）。
- **Write policy**：默认使用 write when memory is idle，同时比较了 no-bypass 和 threshold 策略。

### 关键结果

1. **Row-buffer miss rate 大幅降低**：在 32 bank / 2KB row-buffer 配置下，permutation-based scheme 在几乎所有程序上都获得最低的 miss rate。与其余三种方案中各程序的最优者相比，仍可进一步降低 miss rate 超过 80%（5 个程序）和超过 50%（8 个程序）。唯一例外是 m88ksim，比 swapping 方案高 6%。

2. **Memory stall time 显著减少**：相比 cache line interleaving 平均减少 36%（最高 68%），相比 page interleaving 平均减少 37%（最高 50%），相比 swapping scheme 平均减少 33%（最高 53%）。唯一例外是 su2cor，因其 miss rate 仍高达 70%，open-page 的 late precharge 反而不如 close-page 的 cache line interleaving（stall time 增加 11%）。

3. **总执行时间改善**：相比 cache line / page / swapping interleaving，平均执行时间分别减少 12% / 10% / 8%，最高可达 38% / 19%。

4. **Write buffer 不能替代地址映射优化**：即使使用 threshold write policy，permutation-based scheme 仍可在此基础上进一步减少 miss rate 达 65%~87%（相比不同 baseline）。且 threshold 策略虽降低 miss rate，但可能因延迟 writeback 与后续 read 竞争，导致总执行时间反而增加（如 mgrid 的 miss rate 降 48% 但执行时间增 12%）。

### 结果分析

- **Scalability**：随 bank 数从 4 增至 256，permutation-based scheme 的 miss rate 下降比其他方案更接近线性。原因在于 XOR 能将冲突 page 更广泛地分散到更多 bank。
- **Row-buffer size sensitivity**：从 2KB 到 8KB，permutation-based scheme 始终保持最低 miss rate，且与其他方案的差距随 row-buffer 增大而更明显（更大的 row-buffer 使 locality 更有价值）。
- **su2cor 的反例**：当 miss rate 仍然很高时（70%），open-page mode 的 late precharge penalty 会抵消 hit rate 提升的收益，说明 interleaving scheme 需要与 page policy 协同设计。
- **SPECint95 内存占比小**：整数程序的内存 stall 占比可忽略，因此论文仅对 SPECfp95 和 TPC-C 做了详细的 stall time 分析。

## 审稿人视角

### 优点

1. **问题分析清晰且深刻**：论文从 L2 cache 的 set index 与 DRAM bank index 的 bit 位重叠关系出发，精确地识别了 row-buffer conflict 的三个来源（L2 conflict miss、writeback、特定 stride pattern），并给出了形式化的条件（L2 cache size / associativity > 所有 row-buffer 累计容量时，L2 conflict address 必然是 row-buffer conflict address）。这一分析为方案设计提供了坚实的理论基础。

2. **方案设计极其优雅**：仅用 $k$ 个 XOR 门就同时实现了 conflict 消除和 locality 保留，硬件开销几乎为零。三条性质（conflict 分散、locality 保留、均匀映射）的证明简洁有力。这是一个"教科书级"的 address mapping 设计。

3. **实验全面且扎实**：覆盖了 SPEC95 全套 + TPC-C（当时最具代表性的 workload），考虑了 bank 数量（4~256）、row-buffer size（2~8KB）、write buffer policy 等多维度 sensitivity analysis，且包含了与 3 个 baseline 的全面对比。对 su2cor 这一反例的诚实分析和合理解释也体现了严谨性。

4. **影响深远**：这一 XOR-based address mapping 技术后来被广泛采用于实际处理器和 DRAM 控制器设计中，成为现代内存系统设计的标准技术之一。

### 不足

1. **单核、单线程场景的局限性**：所有实验都在单处理器上进行。在多核/多线程场景下，来自不同 core 的访问会引入完全不同的 conflict pattern，XOR mapping 的效果可能显著不同。论文没有讨论这一方向。

2. **Open-page vs. Close-page 的讨论不够深入**：su2cor 的反例表明，当 miss rate 仍然较高时，open-page mode 反而不如 close-page。论文将 cache line interleaving 固定搭配 close-page、其余三种固定搭配 open-page，没有探讨 permutation-based + close-page 的组合，也没有讨论 adaptive page policy 与 XOR mapping 的协同。

3. **Workload 的时代局限性**：SPEC95 和 TPC-C 的内存访问特征与现代 workload（多核 SPEC CPU2006/2017、graph analytics、DNN inference/training 等）差异巨大。尤其是现代 workload 的 working set 远大于 2MB L2 cache，memory access pattern 更为复杂。

4. **缺少对 memory access scheduling 的协同分析**：论文在结尾提到 access scheduling（引用了同年 Rixner et al. 的 ISCA 2000 工作）可以与 permutation-based interleaving 结合，但没有实验验证。实际上，address mapping 和 scheduling 之间存在交互——mapping 改变了到达各 bank 的 request 分布，这会影响 scheduler 的决策空间。

5. **XOR 选取 tag 低 k 位的依据偏经验性**：论文提到选择 tag 低 $k$ 位"achieves or approaches the lowest miss rate"，但没有给出理论分析，仅靠实验验证。对于哪些地址模式下这一选择不是最优的，缺乏系统性讨论。

### 疑问或值得追问的点

- 在现代系统中（多 channel、多 rank、bank group 层级），XOR mapping 应该作用在哪一层？是 channel-level、rank-level 还是 bank-level？不同层级的 XOR 是否可以叠加？
- 论文的分析高度依赖 "L2 cache size / associativity > row-buffer 总容量"这一条件。当 L2 cache 容量进一步增大（如现代处理器的 LLC 达到几十 MB），或 DRAM 配置有大量 bank（如 HBM 的 pseudo-channel 结构），这一条件的适用性如何？
- 在 LLC 为 non-inclusive / exclusive 策略的现代处理器中，writeback 的 pattern 与论文假设的 inclusive L2 有本质区别，XOR mapping 的效果是否仍然显著？
- 结合现代 DRAM 的 bank group 结构（DDR4/DDR5），同 bank group 内的 bank-to-bank switching 有额外 penalty，XOR mapping 是否需要感知 bank group 层级？
