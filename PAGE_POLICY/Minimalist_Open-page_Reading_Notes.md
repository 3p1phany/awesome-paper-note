---
title: "Minimalist Open-page: A DRAM Page-mode Scheduling Policy for the Many-core Era"
authors: "Dimitris Kaseridis, Jeffrey Stuecheli, Lizy Kurian John"
venue: "MICRO 2011"
year: 2011
---

# Minimalist Open-page: A DRAM Page-mode Scheduling Policy for the Many-core Era

## 基本信息
- **发表**：MICRO (44th International Symposium on Microarchitecture)，2011
- **作者单位**：UT Austin ECE & IBM Corp.
- **注**：第二作者 Jeffrey Stuecheli 来自 IBM，论文中的 prefetch engine 设计和 multi-line prefetch 机制直接借鉴了 IBM POWER6 的工业实践，具有较强的工业背景。

## 一句话总结
> 通过地址映射限制每次 activation 的 page hit 数量（约4次），配合 prefetch 元数据驱动调度，在 many-core 环境下同时改善吞吐量和公平性。

## 问题与动机

### 核心问题
在 many-core 系统中，传统的 open-page policy（如 FR-FCFS）会导致严重的 **row buffer starvation** 和 **thread unfairness**。根本原因是：

1. **贪婪的地址映射**：传统 open-page 地址映射将所有 column bits 放在低位，使得一个顺序地址流的全部 cache line 都映射到同一个 DRAM bank 的同一 row。这导致一个 streaming workload 可以长时间占据一个 bank 的 row buffer，饿死其他线程。

2. **FR-FCFS 的优先级反转**：FR-FCFS 给 page hit 请求更高的优先级，当多个线程竞争同一 bank 时，row buffer locality 高的线程会持续获得优先调度，其他线程被迫等待。

3. **Bank-Level Parallelism (BLP) 低**：传统映射将顺序地址集中到一个 bank，无法利用 bank 间并行性。

### 现有工作的不足
- **PAR-BS** [Mutlu & Moscibroda, ISCA'08]：通过 batch scheduling 改善公平性，但本质上仍在 Scenario 1（优先 page hit workload）和 Scenario 2（中断 page hit workload）之间选择，无法避免 bank conflict 本身。
- **ATLAS** [Kim et al., HPCA'10]：基于 attained service 的线程级优先级，偏向吞吐量而牺牲公平性；对 streaming workload 会过度惩罚。
- **STFM** [Mutlu & Moscibroda, MICRO'07]：识别被 stall 的线程并提升其优先级，但平均情况下退化为 FR-FCFS。
- **TCM** [Kim et al., MICRO'10]：对高 row-buffer locality 的 workload 降低优先级，但仍依赖复杂的线程分类和优先级策略。

所有这些工作的共同局限是：**它们试图通过调度策略（priority scheme）来弥补地址映射导致的结构性不公平，而非从根源上消除问题**。

### 关键观察：Page-mode 的边际递减效应
论文通过三个维度论证了 page-mode 收益的边际递减：

1. **功耗**：Activate power 随 column access per activate 增加而被摊薄，但 RD/WR/Termination power 和 background power 不变。在 DDR3-1333 的功耗模型中，4 次 access/activate 就能获得约 80% 的理想功耗收益。
2. **Bank utilization**：在 2-Rank DDR3-1333、60% bus utilization 下，closed-page (1 access/activate) 的 bank utilization 高达 62%，而 4 accesses/activate 就降至 16%。
3. **tFAW 约束**：DDR3-1333 1KB page 的 tFAW = 30ns，限制每 30ns 最多 4 次 activate。1 access/activate 时峰值利用率只有 80%（6ns×4/30ns），而 2 accesses/activate 就完全消除 tFAW 瓶颈（12ns×4/30ns > 1）。

另一个关键观察是：在现代带有大型 LLC（如 POWER7 的 32MB L2）的处理器中，**row buffer 实际上只利用 spatial locality**——temporal locality 已被 LLC 过滤。而 spatial locality 高度可预测，大部分可被 prefetch engine 捕获。

## 核心方法

### 关键思路
Page-mode 的收益在约 4 次 access/activate 时就已接近饱和，因此没有必要追求最大 page hit rate。通过地址映射将每次 row activation 的 page hit 上限控制在一个小的目标值（如4），可以同时获得三重好处：（1）保留大部分 page-mode 收益，（2）大幅提高 bank-level parallelism，（3）从根本上消除 row buffer starvation。在此基础上，利用 prefetch engine 的元数据来驱动精确的 page-mode 操作和请求优先级分配。

### 技术细节

#### 1. DRAM 地址映射方案（Section 4.1）

这是整个方案的核心机制。关键改动是将传统的 7-bit column address **拆分为两部分**：

- **低 2 bits**：紧接在 Block（cache line offset）位之上，允许 4 条连续 cache line 映射到同一 page（同一 bank 的同一 row）。
- **高 5 bits**：放在 Bank bits 之上、Row bits 之下。

对比传统映射（Figure 5(a)）：
```
[Row | DIMM | Rank | Bank | Column(7b) | Block | MC]
  19   18     17    16-14    13-7        6-5     0
```

Minimalist 映射（Figure 5(b)）：
```
[Row | Col_MSB(5b) | DIMM | Rank | Bank | Col_LSB(2b) | Block | MC]
  19    18-14        13      12    11-9     8-7           6-5     0
```

**效果**：顺序地址流每 4 个 cache line 就会切换到不同的 bank，将原本映射到单一 bank 的长序列 stream 分散到所有 bank 上。例如，原先 128 个连续 cache line 都在同一 bank 同一 row 中，现在每 4 个就换 bank，总共分散到 32 个 bank 组合中。

此外，高位地址 bits 与 bank bits 做 XOR，进一步减少 row buffer conflict（引用 Zhang et al. [21] 的 permutation-based interleaving）。

**设计权衡**：
- 优点：从结构上保证了公平性，避免了 row buffer starvation。
- 代价：牺牲了对超长顺序流的 page hit rate（从理想的 128 降到 4），但论文论证了这部分损失的收益很小。

#### 2. 数据预取引擎（Section 4.2）

预取引擎的设计直接来源于 IBM POWER6 架构的工业实践：

- **基础**：stride-1 硬件 prefetcher，每核 32 个 stream tracker（参考 Lee et al. [9]，即 POWER6 论文）。
- **深度控制**：采用 Adaptive Stream Detection (ASD) 中的 Stream Length Histograms (SHL) 来动态决定每个 access stream 的 prefetch depth。新 stream 逐步增大 depth（ramp up），只有当确认 stream 有效且足够长时才达到最大深度。
- **Cache 污染控制**：所有 prefetch 放入 LLC 的 LRU 位置，直到被实际使用才提升。

**Multi-line Prefetch Requests（Section 4.2.1）**：
这是连接 prefetch engine 和 memory controller 的关键接口。一个 multi-line prefetch request 将映射到同一 DRAM page 的多条 cache line 合并为单个请求发送到 memory controller queue。好处：
- 减少 command bandwidth 和 queue resource 占用
- 保证同一 page 的多次 access 被连续调度（back-to-back），之后立即关闭 page
- 简化 priority scheme——multi-line request 作为一个整体被调度

#### 3. 内存请求队列调度（Section 4.3）

**优先级计算（Table 1）**：
使用 3-bit 优先级（0-7），分两类：

**Normal Read Requests**——基于 Memory-Level Parallelism (MLP)：
| MLP 级别 | Priority |
|---------|----------|
| MLP < 2（Low） | 7（最高） |
| 2 ≤ MLP < 4（Medium） | 6 |
| MLP ≥ 4（High） | 5 |

MLP 通过 MSHR 的已占用条目数直接获取——L2 outstanding miss 数量少意味着每个 miss 都在 critical path 上。

**Prefetch Requests**——基于与 stream head 的距离：
| Distance (cache blocks) | Priority |
|------------------------|----------|
| 0 ≤ d < 4 | 4 |
| 4 ≤ d < 8 | 3 |
| 8 ≤ d < 12 | 2 |
| 12 ≤ d < 16 | 1 |
| d ≥ 16 | 0（最低） |

**动态优先级提升**：每 100ns 每个请求的优先级 +1。如果 prefetch 请求的优先级超过 prefetch 最大值（4），则直接丢弃——因为等待太久的 prefetch 大概率已经没用了（demand request 即将到来）。

**Prefetch→Demand 升级**：如果一个 demand read 到达时，queue 中已有对应的 prefetch request，则将该 prefetch 升级为 demand request，重新按 MLP 分配优先级。

**调度规则（Priority Rules 1）**：
1. Higher Priority Request First
2. Ready-Requests First（同优先级下，已打开 page 的 multi-line request 的后续 access 优先）
3. First-Come First-Served

**注意**：如果有更高优先级的请求需要访问不同的 bank（且该 bank 已超过 tRC），可以中断正在处理的 multi-line request。这保证了 critical request 的低延迟。

**Page Closure Policy（Section 4.3.2）**：
- Multi-line prefetch：处理完后使用 auto-precharge 立即关闭 page。
- Single read / single-line prefetch：在 tRC 窗口内投机性地保持 page open（因为 tRC=50ns 远大于 tRP=12.5ns，即使不关 page 也不影响该 bank 的下次 activate，相当于"免费"的 open page 窗口）。

#### 4. Write 处理（Section 4.3.4）
采用类似 Virtual Write Queue (VWQ) [Stuecheli et al., ISCA'10]（同一组作者的前序工作）的方案，将 read 和 write 分离处理。

### 与现有工作的区别

| 维度 | FR-FCFS / PAR-BS / ATLAS | Minimalist |
|------|--------------------------|------------|
| **公平性来源** | 调度策略（priority scheme）补救 | 地址映射从根源消除 starvation |
| **优先级粒度** | 线程级（per-thread） | 请求级（per-request），基于 MLP 和 prefetch distance |
| **地址映射** | Column-centric（全部 column bits 连续） | Column bits 拆分，限制 page hit ≤ 4 |
| **Prefetch 利用** | 无感知或粗粒度感知 | Prefetch meta-data 直接驱动 page-mode 和 priority |
| **多控制器协调** | 部分需要（如 ATLAS 的 attained service 跟踪） | 不需要，每个 controller 独立 |
| **复杂度** | 较高（batch 形成、attained service 跟踪等） | 较低（主要是地址映射改变 + 简单优先级） |

## 实验评估

### 实验设置
- **仿真平台**：Simics 功能模型 + GEMS 工具集 + 自研详细内存子系统模型
- **处理器配置**：
  - 8 核 CMP，4GHz，30 级流水线，4-wide fetch/decode
  - 128-entry ROB，64-entry Scheduler
  - L1 D/I Cache：64KB, 2-way, 3 cycles
  - L2 Cache (shared)：16MB, 8-way, 12 cycles bank access
  - 每核 16 个 outstanding memory requests
- **内存配置**：
  - 2 个 Memory Controller，每 controller 2 Ranks
  - 每 Rank 8× 4Gbit DDR3-1333 chips
  - 32-entry Read Queue + 32-entry Write Queue per controller
  - DDR3-1333 8-8-8 timing (CL=8 mem cycles, tRCD=12ns, tRP=12ns, tRAS=36ns, tRC=48ns, tFAW=30ns, tRFC=300ns)
  - 总带宽：21.333 GB/s，best-case idle latency：65ns
- **H/W Prefetcher**：stride-1 with dynamic depth, 32 streams/core
- **Workload**：SPEC CPU2006 suite 随机选取的 27 个 8 核多程序组合，带宽覆盖 45%-100% peak DRAM bandwidth（注：DDR timing 限制实际持续利用率约 70%，因此 100% peak 意味着内存已严重过饱和）
- **方法论**：Fast-forward 到代表性执行阶段 → 100M instructions warm-up → 运行至最慢 benchmark 完成 100M instructions
- **Metrics**：
  - Throughput: Weighted Speedup = Σ(IPCi / IPCi,FR-FCFS)
  - Fairness: Harmonic mean of weighted speedup = N / Σ(IPCi,alone / IPCi)

### 对比 Baseline
| Scheme | Hash | Priority | Prefetch Drop |
|--------|------|----------|---------------|
| FR-FCFS | Typical (Fig 5a) | FR-FCFS | 400ns |
| PAR-BS | Typical (Fig 5a) | Batch | 400ns |
| ATLAS | Typical (Fig 5a) | ATLAS | 400ns |
| Minimalist Hash | Proposed (Fig 5b) | FR-FCFS | 400ns |
| Minimalist Hash+Priority | Proposed (Fig 5b) | Per-request | 100-400ns |

### 关键结果

1. **Throughput**：Minimalist Hash+Priority 平均吞吐量提升 **10%**（相对 FR-FCFS），而 ATLAS 和 PAR-BS 分别只提升 3.2% 和 2.8%。
2. **Fairness**：Minimalist 平均公平性提升 **7.5%**（相对 FR-FCFS），相对 PAR-BS 提升 3.4%，相对 ATLAS 提升 2.5%，最大提升达 15%。
3. **Page Accesses per Activation**：FR-FCFS/PAR-BS/ATLAS 的实际 page access rate 为 4.25/4.64/4.63（远低于理想的 128），Minimalist 约 3.5（接近目标值 4）——说明激进 open-page 策略实际上也没有获得多少额外的 page hit。
4. **能耗**：Minimalist Hash+Priority 的 DRAM 能耗相对 FR-FCFS 降低约 **5%**，与 PAR-BS、ATLAS 相当。
5. **Target Page-hit Count Sensitivity**（Figure 9）：目标值为 4 时性能最优；目标 2 和 8 的表现均不如 4。

### 结果分析

- **Ablation**：Minimalist Hash（仅地址映射，无优先级）在大多数情况下与 PAR-BS/ATLAS 相当，但在 workload 5/6/13/18 上明显落后。这些 workload 的共同特征是包含高带宽 workload 与低 MLP 的 demand-intensive workload（如 GemsFDTD）的组合——此时 per-request criticality-based priority 至关重要。加入 Priority scheme 后差距消除。
- **libquantum 的特殊行为**：libquantum 顺序访问 32MB 向量，是典型的 streaming workload（Figure 4 的 pathological case），但多副本运行时，有时会出现"锁步"（lock-step）执行，所有副本同步访问内存，反而不产生 bank conflict——说明 workload 交互行为复杂，难以一概而论。
- **为什么 ATLAS/PAR-BS 增益有限**：这两个方案依赖 memory queue 中积累足够多的请求来发挥重排效果，只有在内存接口完全饱和时才有意义。论文的 27 个 workload 涵盖了中等到高带宽利用率，很多中等负载场景下 ATLAS/PAR-BS 退化为 FR-FCFS。

## 审稿人视角

### 优点

1. **Insight 深刻且有说服力**：page-mode 边际递减效应的量化分析（Figure 2a/2b）非常直观地说明了为什么 4 accesses/activate 就"够用"。从功耗、bank utilization、tFAW 三个维度交叉验证，形成了完整的论证链。这个 observation 具有长期的工业参考价值。

2. **从根源解决问题而非打补丁**：通过地址映射从结构上消除 row buffer starvation，而非像 PAR-BS/ATLAS/TCM 那样用复杂的调度策略去"补救"。这种思路更 elegant，也更容易实现。实际上，改变地址映射的硬件成本几乎为零。

3. **系统性设计，各组件协同**：地址映射（保证公平性和 BLP）→ prefetch engine（提供 meta-data 和 multi-line consolidation）→ priority scheme（per-request criticality）→ page closure policy（tRC-aware），四个组件形成了一致的设计哲学。

4. **工业背景增强可信度**：Prefetch engine 和 multi-line prefetch 来源于 IBM POWER6 的实际设计，VWQ 也是同组前序工作中已验证的机制。论文不是纯学术 proposal，而是有工业实践支撑的。

5. **实验覆盖面广**：27 个 workload mix 从 45% 到 100% 带宽利用率系统性覆盖，避免了只挑 favorable case 展示的问题。Ablation（Hash only vs Hash+Priority）也做得到位。

### 不足

1. **Prefetch 依赖过强**：整个方案的 page-mode 收益依赖 prefetch engine 的精确预测。论文假设的 stride-1 prefetcher 对规则的 streaming pattern 有效，但对 irregular access pattern（如 pointer chasing、indirect array access）几乎无用。对于 prefetch-unfriendly 的 workload，Minimalist 实际上退化为 closed-page policy（每次 activate 只有 1 次 demand access），可能反而不如传统 open-page。论文没有充分分析这种情况。

2. **Target page-hit count 的选择缺乏自适应性**：论文将 target 固定为 4（DDR3-1333 的 sweet spot），并在 Section 5.4 展示了 2/4/8 的对比。但这个最优值随 DRAM 技术参数（tRC, tRAS, page size, frequency）变化而变化（论文自己也承认"different system configuration may shift the optimal page-mode hit count"），却没有提出自适应调整机制。在 DDR4/DDR5 时代，timing 参数和 page size 已经显著变化，固定目标值的可移植性存疑。

3. **多线程/多核场景的 scalability 分析不足**：论文只评估了 8 核。随着核数增加（如 16/32/64 核），每个 bank 的竞争强度增加，4 accesses/activate 的目标是否仍然足够？bank 数量的增加是否能完全抵消？论文缺少这方面的 scalability 分析。

4. **Workload 多样性有限**：仅使用 SPEC CPU2006 的多程序组合，缺少多线程 workload（如 PARSEC、SPLASH-2）的评估。多线程 workload 中，多个线程可能共享数据、产生同步行为，其内存访问模式与多程序组合有本质不同。

5. **与 closed-page policy 的对比缺失**：论文只与 open-page 策略对比（FR-FCFS, PAR-BS, ATLAS），没有包含 closed-page baseline。鉴于论文的核心思路是"只需要少量 page hit"，与 closed-page 的差距（即 page-mode 的实际增量贡献）是一个关键问题，但论文回避了这个对比。

6. **实验基础设施的年代局限**：基于 Simics + GEMS 的全系统模拟，memory controller 部分是自研模型。论文没有开源代码或详细的 validation 数据，可复现性较低。

### 疑问或值得追问的点

- **Row Buffer 大小的敏感性**：论文结尾提到 Minimalist 允许 DRAM 使用更小的 row buffer 以节能，这是一个有趣但未展开的方向。小 row buffer 会如何影响 page hit rate 和系统性能？
- **与 XOR-based bank interleaving 的关系**：论文提到使用了 Zhang et al. [21] 的 XOR 映射来减少 row buffer conflict，但没有详细分析 XOR 的贡献和 column bit split 的贡献各有多少。是否可以只用 XOR 而不拆 column bits？
- **DDR4/DDR5 时代的适用性**：DDR4 引入了 bank group 概念，DDR5 进一步增加了 bank 数量并改变了 timing 结构。Minimalist 的地址映射方案在 bank group 架构下需要如何调整？target page-hit count 是否需要重新标定？
- **Multi-line prefetch 的通用性**：这个机制来源于 IBM POWER 架构，但在 x86/ARM 生态中并不常见。如果没有 multi-line prefetch 支持，方案的效果会降低多少？
