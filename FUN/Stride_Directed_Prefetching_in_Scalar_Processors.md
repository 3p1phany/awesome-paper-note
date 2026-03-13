---
title: "Stride Directed Prefetching in Scalar Processors"
authors: "John W. C. Fu, Janak H. Patel, Bob L. Janssens"
venue: "MICRO 1992 (25th Annual International Symposium on Microarchitecture)"
year: 1992
---

# Stride Directed Prefetching in Scalar Processors

## 基本信息
- **发表**：MICRO 1992（IEEE）
- **作者单位**：Intel Corporation（Fu）；University of Illinois at Urbana-Champaign（Patel, Janssens）

## 一句话总结
> 提出 Stride Prediction Table (SPT) 硬件机制，将向量处理器上的 stride directed prefetching 扩展到标量处理器，通过记录指令地址与数据地址的映射关系来预测数组访问步长并指导 prefetch。

## 问题与动机

数值密集型程序（numerically intensive programs）的执行依赖高速内存访问来支撑流水线化的算术单元。Cache 是降低平均访存延迟的经典手段，但数值程序由于数据集大、数据复用率低、向量步长与 cache block size 之间的交互等原因，cache 性能往往较差。

论文以矩阵乘法 C=A×B 为例，在 4KB direct-mapped cache 上用 trace-driven simulation 展示了标量与向量执行的 miss ratio 对比（Table 1）：向量执行的 MATRIX-only miss ratio 高达 0.485–0.588，而标量执行为 0.258–0.303。标量执行虽然 miss ratio 更低（因为标量访问包含大量对 index 变量的 hit），但矩阵数据本身的 miss ratio 仍然很高，是制约高速算术执行的瓶颈。

先前工作 [FuPa91] 在向量处理器上提出了利用向量步长与 cache block size 关系来指导 prefetch 的方案，但向量处理器的步长是显式已知的（由向量指令提供）。**标量处理器缺乏显式步长信息**，这是本文要解决的核心问题：如何在标量处理器上动态检测并预测数组访问的步长，从而实现 stride directed prefetching。

## 核心方法

### 关键思路

核心观察：在标量处理器执行数值程序时，循环体内的同一条 load/store 指令在相邻迭代中访问的内存地址之间存在固定的步长关系（stride = addr[i+1] - addr[i]）。通过记录每条内存引用指令上一次访问的数据地址，即可在下一次执行该指令时计算出步长，并据此生成 prefetch 地址。这一机制本质上将向量处理器中指令显式携带的步长信息，替换为通过运行时历史推断的隐式步长。

### 技术细节

#### 1. Stride Prediction Table (SPT) 结构

SPT 是一个以指令地址（IA）为索引的硬件表，每个 entry 包含 3 个字段：

| 字段 | 含义 |
|------|------|
| IA (Instruction Address) | 内存引用指令的 PC |
| MA (Memory Address) | 该指令上次引用的数据地址 |
| V (Valid) | 有效位 |

**工作流程**：

- 每次处理器发出内存访问，产生 (IA_cur, MA_cur) 对。
- 用 IA_cur 查表：
  - **SPT hit**（entry valid 且 IA 匹配）：读出 MA_spt，计算 stride = MA_cur - MA_spt，生成 prefetch address = MA_cur + stride。若 stride ≠ 0 则发起 prefetch。更新 MA 字段为 MA_cur。
  - **SPT miss**（entry invalid）：将 (IA_cur, MA_cur) 写入 SPT，置 V=1。

硬件上只需一个 **subtracter**（计算 stride）和一个 **adder**（计算 prefetch address），非常轻量。

#### 2. SPT 组织形式

论文考虑了两种 SPT 组织：
- **Infinite SPT**：entry 数无限，用于评估 upper bound。
- **Finite SPT with direct mapping**：有限 entry 数（64–1K），采用 direct-mapped 组织。由于指令地址的顺序性，direct mapping 已经足够有效。

#### 3. 可选的 Previous Stride 字段

SPT 可增加一个"previous stride"字段，当当前计算的 stride 与上次不一致时抑制 prefetch。这可以避免在外层循环索引跳变或随机步长场景下发起错误的 prefetch。但实验表明，对于高度可向量化的程序（ARC2D、BDNA），stride change 占比不到 1%，因此该字段并非必需。

#### 4. Prefetch 发起策略

论文评估了三种 prefetch initiation 条件：

| 策略 | 触发条件 | 特点 |
|------|----------|------|
| **pf_miss** | demand access miss 时发起 prefetch | prefetch 数量最少；prefetch 请求与 demand 请求冲突 |
| **pf_hit** | demand access hit 时发起 prefetch | prefetch 请求不与 demand 请求冲突；但 miss rate 高时 hit 少，prefetch 机会有限 |
| **pf_all** | 任何 demand access（hit 或 miss）均发起 | prefetch 数量最多，性能最好 |

Prefetch 地址 = demand address + calculated stride。每次仅 prefetch 一个 block。Prefetch 数据仅在 prefetch access miss 时才加载进 cache，且 prefetch block 与 demand block 在 cache 中享受同等替换策略（可互相替换）。

#### 5. 关键设计权衡

- **Prefetching vs. Larger Block Size**：论文提出了一个重要的对比视角——与其让 prefetch cache 使用 block size n 并 prefetch 额外 n 字节，不如直接用 2n 的 block size 而不 prefetch？这本质上是在比较"顺序数据加载的准确性"与"数据访问模式预测的准确性"。对于 ARC2D、BDNA、MATRIX，pf_all 策略始终显著优于简单增大 block size。
- **Prefetch Overhead 度量**：定义 PF₀ = (PF + CM_pf) - CM_np，其中 PF 是 prefetch 次数，CM_pf 和 CM_np 分别是 prefetch/no-prefetch cache 的 miss 数。PF₀/PF 的含义是 prefetch load 中无效（未被后续引用或替换了有用数据）的比例。三个极值：0 表示完美 prefetch，1 表示 prefetch 无效果，2 表示 prefetch 反而有害（替换了有用数据）。

### 与现有工作的区别

1. **[FuPa91] (Fu & Patel, ISCA 1991)**：在多处理器向量 cache 上做 stride directed prefetching，步长由向量指令显式提供。本文将该思想扩展到标量处理器，需要通过 SPT 硬件动态推断步长。
2. **[BaCh91] (Baer & Chen, Supercomputing 1991)**：独立提出了类似的 on-chip preloading 方案，但依赖 branch prediction 和 lookahead program counter 来发起 prefetch，即通过指令流预测来提前知道未来的访存指令。本文方案更简单，直接基于指令地址和数据地址的历史关系。
3. **[Skle92] (Sklenar, ISCA 1992 poster)**：在标量计算机上为向量操作设计 prefetch unit，思路类似但独立提出。

## 实验评估

### 实验设置

- **Trace 采集平台**：Encore Multimax，使用 TRAPEDS inline tracing 系统采集 trace。
- **仿真方法**：Trace-driven simulation，采用 immediate simulation（每条引用产生后立即模拟）。
- **Cache 配置**：64 KB direct-mapped cache，block size 从 8 到 256 bytes 变化。
- **Workload**：
  - PERFECT benchmarks：ADM、ARC2D、DYFESM、BDNA（均为数值计算程序）
  - SPEC benchmarks：MATRIX-3000（规则数组访问，预期效果好）、ESPRESSO（不规则访问，预期效果差）
  - 所有 PERFECT 程序和 MATRIX 原为 Fortran，经 f2c 转换为 C
  - 仿真截止条件：1.2 billion data references 或执行完毕
- **仿真内容**：仅模拟 data references，不模拟 instruction stream 和 stack references。
- **SPT 配置**：Section 4–6 使用 infinite SPT；Section 7 评估 64–1K entries 的 finite SPT。
- **Reference 规模**（Table 2）：ARC2D 2000M、BDNA 1973M、MATRIX 651M、ESPRESSO 31M data references。

### 关键结果

1. **高度可向量化程序的 miss rate 大幅降低**：对于 ARC2D、BDNA、MATRIX，pf_all 策略下 spt-prefetch 相比 no-prefetch 显著降低 miss ratio。MATRIX 在小 block size（8 bytes）时 miss ratio 从约 0.35 降至接近 0.05，降幅接近 85%。与增大 block size 对比，MATRIX 的 pf_all 在所有 load size 下 miss rate 下降接近 100%。

2. **Prefetch overhead 极低**（高可向量化程序）：ARC2D 和 MATRIX 在 block size ≤ 128 bytes 时，pf_all 和 pf_hit 的 overhead（PF₀/PF）仅 3–4%。BDNA 稍高，从 5%（8 bytes）到约 20%（128 bytes）。

3. **不规则程序收益有限且 overhead 高**：ADM、DYFESM、ESPRESSO 的 miss rate 本身已较低，prefetch 带来的改善较小。ESPRESSO 在 8 bytes block size 下约 60% 的 prefetch load 未被使用，意味着为消除一次 demand miss 需要发起近两倍的内存请求。

4. **SPT 仅需少量 entry 即可有效**：ARC2D 使用 256 entries 时，超过 90% 的内存引用命中 SPT；1K entries 时性能接近 infinite SPT。BDNA 因循环更长，SPT hit rate 较低（1K entries 时 85%，64 entries 时仅 10%）。Stride change 在 infinite SPT 下 ARC2D 和 BDNA 均不到 1%。

### 结果分析

- **程序特征是决定性因素**：深层嵌套循环、高可向量化比例、循环不变步长的程序受益最大。ARC2D 和 BDNA 的向量化引用分别超过 90% 和 80%，且加载长向量。
- **Prefetch initiation 策略敏感性高**：pf_miss 因发起次数少且与 demand 请求冲突，效果最差；pf_hit 在小 block size 时因 hit 率太低而受限；pf_all 始终最优但带宽需求最高。
- **Block size 交互效应**：block size 增大时，no-prefetch cache 性能提升，prefetch 的边际收益递减。256 bytes 时 BDNA 的 pf_hit 和 pf_all 的 overhead 甚至超过 1，说明大 block 引发 cache 内部冲突加剧。
- **SPT size 的 sensitivity**：SPT 只需容纳当前最热循环的内存引用指令即可有效。由于指令地址的顺序性，direct mapping 效果很好。从一个循环切换到另一个循环时，只需一次循环遍历即可重新填充 SPT。

## 审稿人视角

### 优点

1. **思路简洁且硬件开销极低**：SPT 仅需一个减法器、一个加法器和少量表项（实验证明 256–1K entries 即可），是一个非常实用的硬件 prefetch 机制。相比依赖 branch prediction + lookahead PC 的方案 [BaCh91]，设计更优雅。

2. **实验设计中 "Prefetching vs. Larger Block" 的对比视角很有洞察力**：将 prefetch 带来的额外数据加载量与等量的 block size 增大进行公平对比，避免了简单地在同等 block size 下对比造成的"以多打少"的不公平。这一方法论在后续 prefetching 研究中被广泛沿用。

3. **Prefetch overhead metric 的定义合理**：PF₀/PF 指标综合了 prefetch 的准确性和对 cache 的污染效应，比单纯的 prefetch accuracy 更有实际意义。它能捕捉到"prefetch 正确但替换了有用数据"这类隐性开销。

4. **对 SPT size sensitivity 的分析较为充分**：从 64 到 infinite entries 的渐进分析，加上 spt hit rate、stride change rate、prefetch attempts 等细粒度指标的分解，使读者能清晰理解硬件预算与性能的 trade-off。

### 不足

1. **Cache 配置单一，缺乏 associativity 和 capacity 的 sensitivity 分析**：全文仅使用 64KB direct-mapped cache。Direct-mapped cache 的冲突 miss 率较高，这可能放大了 prefetch 的收益（因为 baseline miss rate 高）。如果评估 2-way 或 4-way set-associative cache，prefetch 的边际收益可能会缩小。此外，64KB 在 1992 年是合理的 L1 cache size，但缺乏对不同 cache capacity 的 sensitivity 分析。

2. **未考虑 prefetch 的时序效应**：论文明确声明不考虑 prefetch access timing 和 memory bandwidth 对系统性能的影响，仅评估 miss count 的减少。然而，prefetch 的价值很大程度上取决于能否及时地（在 demand access 之前）将数据加载到 cache。pf_all 策略虽然 miss rate 最低，但仅提前一个 stride 的距离，在 memory latency 较高的系统中可能不够。论文未讨论 prefetch distance（提前几个 stride）的问题。

3. **Benchmark 数量和代表性有限**：仅 6 个程序，且 4 个来自 PERFECT（均为科学计算），1 个 MATRIX（规则数组）、1 个 ESPRESSO（逻辑综合）。缺乏对整数密集型程序、指针追踪型访问模式的评估。论文也坦承时间限制导致程序数量不足。

4. **未讨论与 instruction cache / TLB 的交互**：SPT 查询需要在每次 data access 时进行，这增加了数据通路的延迟。论文未讨论 SPT lookup 是否位于 critical path 上，以及如何与处理器流水线集成。

5. **Stride change 的处理过于简单**：论文仅提出可以用 previous stride 字段来过滤 stride change，但未深入评估该机制的效果。对于外层循环边界的 stride 突变，一个更完善的方案可能需要状态机（如后来 RPT 中常见的 init → transient → steady 三态机）来管理 stride 的可信度。

### 疑问或值得追问的点

- 论文中 prefetch unit 固定为一个 block。如果允许一次 prefetch 多个 block（即 prefetch degree > 1），是否能进一步提升效果？这对应于后来 stride prefetcher 中 prefetch degree 和 prefetch distance 的概念。
- SPT 仅使用 IA（PC）作为索引。对于同一条 load 指令在不同调用上下文中呈现不同 stride 的场景（如函数在不同循环中被调用），SPT 会如何表现？是否需要引入 calling context 信息？
- Fortran 程序经 f2c 转换为 C 后，编译器可能引入额外的间接访问和 index 计算，这是否影响了 stride 的规律性和 SPT 的有效性？论文未对此做 sensitivity 分析。
- 论文发表于 1992 年，此后 stride prefetcher 的设计（如 RPT with state machine、GHB-based stride detection 等）已有长足发展，但本文提出的基于 PC-indexed table 检测 stride 的基本范式至今仍是主流硬件 stride prefetcher 的基础。
