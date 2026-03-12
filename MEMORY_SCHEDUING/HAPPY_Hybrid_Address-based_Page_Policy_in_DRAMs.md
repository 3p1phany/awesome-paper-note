---
title: "HAPPY: Hybrid Address-based Page Policy in DRAMs"
authors: "Mohsen Ghasempour, Aamer Jaleel, Jim D. Garside, Mikel Luján"
venue: "MEMSYS 2016"
year: 2016
---

# HAPPY: Hybrid Address-based Page Policy in DRAMs

## 基本信息
- **发表**：MEMSYS (International Symposium on Memory Systems), 2016
- **机构**：University of Manchester (Ghasempour, Garside, Luján) + NVIDIA (Jaleel)

## 一句话总结
> 利用物理地址比特位与 DRAM 结构的固有映射关系，将 page closure predictor 的监控粒度从 per-row/per-bank 压缩到 per-address-bit，以对数级存储开销实现等同甚至略优的预测精度。

## 问题与动机

**核心问题**：DRAM memory controller 中的 page closure policy predictor 随内存容量增长面临严重的可扩展性问题。

**问题重要性**：静态 page policy（open-page 或 close-page）无法适应所有 workload——论文实验表明约 68% 的 workload 偏好 open-page，32% 偏好 close-page，两者之间存在高达约 40% 的性能差异（如 libquantum 在 open-page 下节省约 18%，tigr 在 close-page 下节省约 18%）。因此需要动态 hybrid page policy，即运行时在 open-page 和 close-page 之间切换。

**现有工作不足**：现有的 hybrid page policy predictor（无论是 access-based 还是 time-based）都需要在 per-row 或 per-bank 粒度上维护 performance counter。以传统 Hybrid 为例，一个 4 GB 内存系统（1 channel, 2 ranks, 8 banks, 32768 rows/bank）需要 524,288 个 2-bit saturating counter。随着内存向 64 TB（RAMCloud）甚至 150 TB（Facebook memcache）扩展，这种线性增长的硬件开销变得不可接受。而如果转向 coarse-grain 方案（如 per-bank），则会损失预测精度。

## 核心方法

### 关键思路

HAPPY 的核心 observation 是：**物理地址比特与 DRAM 内部结构（channel, rank, bank, row, column）之间存在固定的映射关系**。既然所有现有 predictor 都是在地址映射之后（即在 DRAM 结构层面）做监控，那么同样的信息完全可以在映射之前（即直接从物理地址比特中）提取。由此，HAPPY 将 predictor 的监控单元从 per-row（随内存容量线性增长）转变为 per-address-bit（随内存容量对数增长），这是一种从根本上改变 scalability 特性的编码方式。

### 技术细节

#### 1. 基本编码原理

对于一个物理地址中参与 DRAM 结构定位的每一个 bit position，HAPPY 分配**两个 encoding counter**：一个追踪该 bit 为 '1' 时的 page hit/conflict 行为，另一个追踪该 bit 为 '0' 时的行为。关键点在于：column 和 cache-line offset 对应的地址位不参与编码，因为 page hit/miss 是 per-row 粒度的事件，进一步减少了开销。

#### 2. Access-based 应用：Hybrid-HAPPY

原始 Hybrid page policy 为每个 row 维护一个 2-bit saturating counter（page conflict 时递增，page hit 时递减，高位决定 open/close）。HAPPY 的替代方案是：

- 每个参与的物理地址 bit 位置有两个 2-bit saturating counter（分别对应 '0' encoding 和 '1' encoding）
- 训练方式与原始 Hybrid 相同：page conflict 时递增对应 counter，page hit 时递减
- **决策方式一：Majority Vote** —— 对于给定物理地址，根据每个 bit 的值查找对应的 counter，看其高位判断该 bit 投 open-page 还是 close-page 票，最终取多数票决定
- **决策方式二：Aggregation** —— 将所有参与 counter 的值求和，与 threshold（= 地址位宽 × counter 最大值 / 2）比较

实验表明两种决策方式效果相似，最终采用 Majority Vote。

#### 3. Time-based 应用：Intel-adaptive-HAPPY

Intel Xeon X5650 的 Intel-adaptive open-page policy 的原始结构为每个 bank 的 row buffer 配备一个 Timeout Counter (TC)、一个 Timeout Register (TR) 和一个 4-bit Mistake Counter (MC)。行为逻辑：
- row 保持打开直到 TC 达到 TR 后关闭
- MC 追踪"错误"：本该是 page-empty 却发生 page conflict（MC 递减）；本该是 page-hit 却已关闭（MC 递增）
- 周期性检查 MC：超过高阈值则增大 TR（更激进地保持打开），低于低阈值则减小 TR

HAPPY 的替代方案是：
- 每个物理地址 bit 位置配备一个**完整的 Monitoring Unit**（包含 MC + TR），而非简单的 saturating counter
- MC 的更新从 per-bank 变为 per-address-bit
- row 的 timeout 值通过**聚合所有参与 bit 的 TR 值**来确定，而非使用单一全局 TR
- 仍需一个 per-bank 的全局 TC 来追踪时间
- 为公平比较，每个 bit 位置的 TR 大小经过调整，使最大可能的聚合 timeout 值等于原始单 bank TR 的最大值

#### 4. Scalability 分析

| 实现方式 | 所需 counter 数量 |
|---------|------------------|
| Hybrid (原始) | X × Y × Z × W |
| Hybrid-HAPPY | (log₂X + log₂Y + log₂Z + log₂W) × 2 |
| Intel-adaptive (原始) | (X × Y × Z) × 2 |
| Intel-adaptive-HAPPY | (log₂X + log₂Y + log₂Z + log₂W) × 4 |

其中 X = channels, Y = ranks, Z = banks, W = rows。可以看出 HAPPY 版本的开销随内存容量呈**对数增长**，而原始版本呈**线性增长**。

#### 5. Ensemble Methods 的理论解释

论文将 HAPPY 与 Machine Learning 中的 Ensemble Methods 做类比：每个 address bit 对应的 predictor pair 相当于一个弱分类器，使用不同 feature（不同 bit 位置）和不同 dataset（'0' 和 '1' 出现时的训练样本），通过非线性组合（Majority Vote）得到最终预测。内存翻倍只增加一个 address bit，即增加一对 predictor，同时可表示的决策空间指数级增长（2^counter数量）。

### 与现有工作的区别

1. **vs. 传统 Hybrid page policy**：传统方案为每个 row 分配一个 saturating counter，开销与 row 数量线性相关。HAPPY 将 counter 绑定到 address bit 而非 row，存储开销降低数个数量级（64 GB 时约 182,000×）。

2. **vs. Intel-adaptive open-page (Xeon X5650)**：Intel 方案为每个 bank 维护一套 MC + TR，开销随 bank 数量线性增长。HAPPY 版本将 monitoring unit 绑定到 address bit，64 GB 时存储开销降低约 5×，512 GB 时降低约 40×。

3. **vs. Awasthi et al. (PACT 2011) 的 history table 方案**：该方案维护每个 row 的历史访问次数表，开销同样与 row 数量成正比，且需要额外的表查找逻辑。HAPPY 无需任何 per-row 存储。

## 实验评估

### 实验设置
- **仿真平台**：USIMM（Utah Simulated Memory Module），扩展支持 7 种 page closure policy
- **处理器配置**：3.2 GHz, pipeline depth 10, ROB size 32
- **DRAM 配置**：2 GB（single-thread）/ 4 GB（multi-thread），Bus Speed 800 MHz，1 Channel, 1 Rank, 8 Banks, 65536 rows/bank, 128 cache lines/row
- **调度算法**：FR-FCFS (First Ready, First Come First Serve)
- **Workload**：70 个 memory intensive workload mix，来源包括 SPEC（20 个）、PARSEC（8 个）、BIOBENCH（2 个）、HPC（13 个）、COMMERCIAL（5 个）共 48 个单线程应用，以及 22 个随机生成的多核 workload mix（4/8/16 线程组合）
- **地址映射**：测试了 3 种 mapping scheme（标准 row-buffer locality 最大化、Zhang et al. XOR-based permutation、Kaseridis et al. minimalist open-page），最终主要采用 Mapping 3（minimalist open-page）
- **对比 baseline**：Open-Page, Close-Page, Hybrid (原始), Fixed-Open, Intel-adaptive (原始), Static Profiling (per-workload 最优静态策略作为 upper bound)
- **Memory Footprint 覆盖**：单线程平均 30%（最高 97%），4 线程平均 50%（最高 75%），8 线程平均 70%（最高 85%），16 线程平均 95%（最高 99.8%）

### 关键结果

1. **性能（单线程）**：Intel-adaptive-HAPPY 相比 open-page 和 close-page 分别平均提升约 5% 和 8%（最高分别达 12% 和 20%），与原始 Intel-adaptive 性能持平或略优（最高提升约 2%）。

2. **性能（多线程）**：Intel-adaptive-HAPPY 相比 open-page 和 close-page 分别平均提升约 5% 和 14%（最高分别达 9% 和 22%），即使在内存占用率高达 99.8% 的情况下仍保持稳定。

3. **预测精度**：Intel-adaptive-HAPPY 的 page-hit 预测精度为 81.9%，page-miss 预测精度为 84.9%，均略优于原始 Intel-adaptive（80.0% / 83.5%）。Hybrid-HAPPY 也略优于原始 Hybrid（68.9% vs 68.3% page-hit, 61.5% vs 59.0% page-miss）。

4. **Scalability**：Intel-adaptive-HAPPY 在 64 GB 时存储开销降低约 5×，512 GB 时降低约 40×。Hybrid-HAPPY 在 64 GB 时降低约 182,000×，512 GB 时降低约 1.2M×。HAPPY 的存储开销随内存容量呈对数增长。

### 结果分析

- **Address Mapping 敏感性**：在 3 种地址映射方案下，Intel-adaptive-HAPPY 的预测精度（page-hit 和 page-miss）均与原始实现持平或略优，表明 HAPPY 对地址映射不敏感。

- **Memory Size 敏感性**：测试至 64 GB，HAPPY 行为一致。论文指出影响 HAPPY 性能的关键因素是**地址空间利用率**而非内存绝对大小，因此特意在多线程实验中使用了高内存占用率场景。

- **表现较差的场景**：HAPPY 作为全局编码方案，在 workload 行为高度动态且仅针对 DRAM 局部区域时（如 'tigr'），无法像 fine-grain per-row 方案那样精确追踪。Intel-adaptive 类方案在应用访问模式频繁切换（如 'canneal', 'comm1'）时因 TR 更新粒度过细（每次仅 ±1）而反应迟缓。

- **Page-hit 精度对性能的影响更大**：从精度结果可以推断，page-hit prediction accuracy 对整体执行时间的影响大于 page-miss prediction accuracy。

## 审稿人视角

### 优点

1. **核心 insight 简洁有力**：利用物理地址到 DRAM 结构的固定映射关系，将 per-row 监控转化为 per-bit 监控，从而将存储开销从 O(N) 降至 O(log N)，这是一个非常优雅且直觉清晰的 idea。这种"编码压缩"的思路具有通用性，不限于 page closure policy。

2. **评估较为全面**：70 个 workload mix 覆盖了 SPEC、PARSEC、BIOBENCH、HPC、COMMERCIAL 等多种 benchmark suite，包含单线程和多线程场景，并且特别关注了 memory footprint 的覆盖率（高达 99.8%），增强了结果的可信度。同时测试了 3 种 address mapping scheme 的敏感性。

3. **实用性强**：方法本身几乎不改变原有 predictor 的训练和决策逻辑，仅替换了存储/索引结构，因此与现有 page closure policy 设计的集成非常自然。论文展示了在两种截然不同的 predictor（access-based 和 time-based）上的应用，说明了方法的通用性。

4. **Ensemble Methods 视角的理论支撑**：虽然非形式化证明，但从 ensemble learning 的角度解释 HAPPY 为何有效（多个弱分类器基于不同 feature 和 dataset 的组合）是一个有价值的理论补充。

### 不足

1. **实验平台配置过于简化**：1 Channel / 1 Rank / 8 Banks 的配置在 2016 年就已经远低于真实系统规格。虽然论文解释这是为了增加 memory congestion，但这同时也降低了 bank-level parallelism 的影响，可能夸大了 page policy 对性能的相对影响。更重要的是，ROB size 仅为 32，远小于当时主流处理器（如 Haswell 的 192-entry ROB），这意味着 memory-level parallelism 被严重低估，从而可能影响 page policy 的实际重要性。

2. **缺乏与更先进调度算法的交互分析**：实验中使用的是 FR-FCFS 调度，这是一个已知偏好 row-hit 的 scheduler。在更先进的调度策略（如考虑 fairness 的 BLISS、PAR-BS 等）下，page closure policy 的相对重要性和 HAPPY 的表现可能不同。论文未讨论此交互效应。

3. **"略优于原始实现"的性能提升缺乏深入解释**：HAPPY 在某些 workload 上不仅匹配而且超越了原始 per-row 实现的精度和性能（最高约 2-3%），但论文对此现象的解释不够充分。如果 per-row 跟踪是"完美"粒度，那么 per-bit 的聚合编码为何能更好？这可能暗示了某种正则化效果或抗噪声能力，值得深入分析。

4. **Scalability 论证的实际意义存疑**：论文推算了 512 GB 甚至更大内存下的存储开销对比，但实际仿真只做到了 4 GB。对于大内存系统，物理地址空间远大于实际使用的内存空间，per-bit predictor 在大量 address bit 从未被训练的情况下表现如何，论文缺乏有效论证。

5. **缺少功耗和面积的具体量化评估**：论文以存储 bytes 数作为 scalability 的度量，但未给出实际的面积（mm²）和功耗（mW）估算。考虑到 HAPPY 需要在每次内存访问时读取并更新多个 counter（与地址位宽成正比），其访问能耗和延迟开销可能并不可忽略，特别是在关键路径上。

6. **Majority Vote 的硬件实现成本被忽视**：对于一个 30+ bit 宽的地址，Majority Vote 需要一个多输入的 population count 电路。虽然这在逻辑上不复杂，但在 memory controller 的紧凑时序约束下，这个组合逻辑的延迟是否在关键路径上需要评估。

### 疑问或值得追问的点

- HAPPY 在 XOR-based address mapping（如 Mapping 3）下仍然有效，这是否意味着 HAPPY 实际上不依赖物理地址与 DRAM 结构的"直接对应"关系，而是在利用地址空间中更一般的空间局部性模式？如果是，那么 Section 3.1 中关于"固定映射创造强相关性"的核心 motivation 就需要重新审视。

- 在多核场景下，不同 core 的地址空间交错访问同一 bank 的不同 row，HAPPY 的全局 per-bit counter 如何区分不同 core 的行为差异？是否会出现 destructive interference？

- 论文提到 HAPPY 可以应用到 DRAM 结构的"其他方面"（Section 3.1 末尾），但未给出具体示例。将该编码思路应用于 DRAM refresh scheduling、write buffer management 或 bank-level parallelism 优化是否可行？

- 在 DDR5 及更新的 DRAM 标准中引入了 same-bank refresh、per-bank refresh 等新特性，HAPPY 的 encoding 机制是否仍然适用？
