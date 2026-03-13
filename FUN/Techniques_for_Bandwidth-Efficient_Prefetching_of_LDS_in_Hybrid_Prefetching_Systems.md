---
title: "Techniques for Bandwidth-Efficient Prefetching of Linked Data Structures in Hybrid Prefetching Systems"
authors: "Eiman Ebrahimi, Onur Mutlu, Yale N. Patt"
venue: "HPCA 2009"
year: 2009
---

# Techniques for Bandwidth-Efficient Prefetching of Linked Data Structures in Hybrid Prefetching Systems

## 基本信息

- **作者**：Eiman Ebrahimi (UT Austin), Onur Mutlu (CMU), Yale N. Patt (UT Austin)
- **发表**：HPCA 2009

## 一句话总结

> 通过 compiler-guided prefetch filtering 和 coordinated prefetcher throttling 两项技术，实现低硬件开销、高带宽效率的 LDS prefetching 与 stream prefetching 的协同工作。

## 问题与动机

### 要解决的问题

现代处理器普遍部署了 stream prefetcher 来处理 stride/streaming access pattern，但对 linked data structure (LDS) 的 pointer-chasing access pattern 无能为力。论文的核心目标是：**在已有 stream prefetcher 的系统中，如何以低硬件开销、高带宽效率的方式加入 LDS prefetcher，构建 hybrid prefetching system。**

### 问题的重要性

论文通过 oracle 实验量化了 LDS prefetching 的潜力：在已有 aggressive stream prefetcher 的基础上，如果理想地将所有 LDS 访问的 LLC miss 转化为 hit，平均性能可提升 53.7%（去掉 health 后为 37.7%）。在 mcf、astar、xalancbmk、omnetpp、ammp、bisort、health、pfast 这 8 个 benchmark 上，stream prefetcher 的 coverage 低于 20%，大量 cache miss 由 LDS 的 non-streaming 访问导致。

### 现有工作的不足

现有 LDS prefetcher 存在两大核心缺陷：

1. **硬件/存储开销过大**：Jump pointer prefetcher、pointer cache、hardware correlation prefetcher 需要大量存储来记录 pointer 值（correlation table 通常需 1-2MB）。Pre-execution 方案需要额外 thread context 或专用硬件。

2. **产生大量 useless prefetch**：Content-Directed Prefetching (CDP) 虽然是 stateless 的（不需要存储 pointer），但它贪婪地对 cache block 中发现的**所有** pointer 地址发起 prefetch，导致 accuracy 极低（如 mcf 仅 1.4%、xalancbmk 仅 0.9%），严重的 cache pollution 和带宽浪费。实验显示，将 CDP 加入已有 stream prefetcher 的系统后，平均性能下降 14%，带宽消耗增加 83.3%。

此外，还存在一个被忽视的系统级问题：**当 LDS prefetcher 和 stream prefetcher 共存时，两者对 memory subsystem 资源的竞争**（memory request buffer entries、DRAM bus bandwidth、DRAM bank occupancy、LLC space）会导致 useful prefetch 的 latency 增加 52%，削弱双方的有效性。

## 核心方法

### 关键思路

论文的核心洞察有两个：

1. **并非所有 pointer 都值得 prefetch**：在 CDP 扫描到的 pointer 中，只有一部分（属于 "beneficial Pointer Groups"）的 prefetch 是有用的，编译器可以通过 profiling 区分 beneficial 和 harmful Pointer Groups，并以 hint bit vector 的形式告知硬件。

2. **Hybrid prefetching system 中多个 prefetcher 的资源竞争需要协调管理**：不能简单地将两个 prefetcher 叠加使用，而应根据各 prefetcher 的 runtime accuracy 和 coverage 反馈，协调调节各自的 aggressiveness，让表现更好的 prefetcher 获得更多资源。

### 技术细节

#### 组件一：Efficient Content-Directed Prefetching (ECDP)

**Pointer Group (PG) 的定义**：对于一条 load 指令 L 和一个偏移量 X，PG(L, X) 定义为所有由 L 取回的 cache block 中，距离 L 访问地址偏移为 X 处的 pointer 集合。由于 LDS 节点通常连续分配在内存中，同一 cache block 内的 pointer field 相对于触发 miss 的 load 地址总是位于固定偏移处。在程序抽象层面，每个 PG 对应代码中的一个 pointer field（如 `node->left`、`node->right`）。

**PG 的有用性分析**：论文将 PG 分为 beneficial（>50% 的 prefetch 有用）和 harmful（>50% 的 prefetch 无用）。实验显示，在 astar、omnetpp、bisort、mst 等 benchmark 中，大量 PG 是 harmful 的。以 mst 中的 hash table 遍历为例：节点包含 Key、D1、D2、Next 四个字段，遍历链表时只有 Next pointer 的 prefetch 有意义（因为大部分 key 不匹配需要继续遍历），而 D1、D2 的 prefetch 几乎总是无用的。

**Compiler-guided mechanism**：

- **Profiling 阶段**：编译器通过模拟目标机器的 cache hierarchy 和 prefetcher 行为（不需要 detailed timing simulation），或利用 informing load 等硬件支持，收集每条 load 指令关联的各 PG 的 usefulness 信息。
- **Hint 传递**：编译器为每条 load 指令生成一个 hint bit vector。对于 64B cache block + 4B 地址的配置，bit vector 为 16 bits。第 n 位置位表示距 load 访问地址偏移 4×n 处的 PG 是 beneficial 的。该 bit vector 通过 ISA 新增指令传递给微架构。
- **Runtime 行为**：当 demand miss 发生时，CDP 扫描取回的 cache block，但**只对 hint bit vector 标记为 beneficial 的 pointer 发起 prefetch**。注意：如果 cache block 是由 CDP prefetch 请求取回的（而非 demand miss），则仍然 prefetch 其中发现的所有 pointer（递归 prefetch 不受 hint 约束）。

**关键设计选择**：
- 选择在 demand miss 触发的 cache block 上应用 filtering，而递归 prefetch 的 block 不 filtering——这是因为递归 prefetch 的 block 已经过第一层 filtering 筛选，其中的 pointer 更可能有用。
- Hint bit vector 同时编码正偏移和负偏移（论文实现中使用了两个 bit vector）。

#### 组件二：Coordinated Prefetcher Throttling

**反馈信息收集**：为每个 prefetcher 维护两个计数器——total-prefetched（发出的 prefetch 总数）和 total-used（被 demand request 使用的 prefetch 数），加上一个全局 total-misses 计数器（demand request 的 LLC miss 数）。由此计算：
- Accuracy = total-used / total-prefetched
- Coverage = total-used / (total-used + total-misses)

每个 cache block 的 tag entry 扩展 2 个 prefetched bit（分别对应 CDP 和 stream prefetcher）。采用 interval-based sampling，interval 以 L2 eviction 数为度量（阈值 8192），每个 interval 结束时通过指数平滑更新计数器值（新值 = 1/2 × 旧值 + 1/2 × 本 interval 值），兼顾历史行为和最近行为。

**Aggressiveness 级别**：
| Level | Stream Distance | Stream Degree | CDP Max Recursion Depth |
|-------|----------------|---------------|------------------------|
| Very Conservative | 4 | 1 | 1 |
| Conservative | 8 | 1 | 2 |
| Moderate | 16 | 2 | 3 |
| Aggressive | 32 | 4 | 4 |

**Throttling 启发式规则**（5 条核心规则，对两个 prefetcher 对称应用）：

| Case | Deciding Prefetcher Coverage | Deciding Prefetcher Accuracy | Rival Prefetcher Coverage | Decision |
|------|-----|----------|-------|----------|
| 1 | High | - | - | Throttle Up |
| 2 | Low | Low | - | Throttle Down |
| 3 | Low | Medium/High | Low | Throttle Up |
| 4 | Low | Low/Medium | High | Throttle Down |
| 5 | Low | High | High | Do Nothing |

**阈值设定**：T_coverage=0.2, A_low=0.4, A_high=0.7。论文指出这些值是 empirically determined 但未 fine-tuned，参数数量少（仅 3 个）使得调优可行。

**关键设计洞察**：
- Case 1 的合理性：高 coverage 的 prefetcher 不太可能 accuracy 低（因为 pollution 会导致更多 miss，从而降低 coverage），因此直接 throttle up。
- Case 4 vs Case 5 的区别：当 rival 表现好时，如果 deciding prefetcher accuracy 高则保持不变（避免伤害有用 prefetch），accuracy 低/中则让步给 rival。
- 对称性设计：相同规则适用于两个 prefetcher，潜在可扩展至 >2 个 prefetcher 的场景。

### 与现有工作的区别

1. **vs. Original CDP [Cooksey et al., ASPLOS 2002]**：原始 CDP 不加区分地 prefetch 所有识别到的 pointer，导致 accuracy 极低、bandwidth 浪费严重。ECDP 通过 compiler-guided hint 实现 fine-grained filtering，将 very useful PGs 比例从 27% 提升至 68.5%。

2. **vs. Guided Region Prefetching (GRP) [Wang et al., ISCA 2003]**：GRP 同样使用 compiler hints 控制 CDP，但其控制粒度是 coarse-grained 的——对一条 load 指令要么启用所有 pointer prefetch，要么全部禁用。ECDP 是 fine-grained 的——可以选择性地启用/禁用特定 pointer field 的 prefetch。GRP 的 coarse-grained 控制仅提升 0.4% 性能。

3. **vs. Feedback Directed Prefetching (FDP) [Srinath et al., HPCA 2007]**：FDP 独立地对每个 prefetcher 进行 feedback-directed throttling，无法感知 prefetcher 间的交互。Coordinated throttling 的核心区别在于 throttling 决策同时考虑 rival prefetcher 的 coverage，从而协调资源分配。实验显示 coordinated throttling 比 FDP 高 5% 性能。

4. **vs. Correlation prefetchers (Markov [Joseph & Grunwald, ISCA 1997], GHB [Nesbit & Smith, HPCA 2004])**：这些方法需要大量存储（Markov 1MB，GHB 12KB），且只能 prefetch 之前观察到的地址相关性。ECDP 仅需 2.11KB 存储，且可以 prefetch 从未出现过的 pointer 地址（因为它直接从 cache block 内容中发现 pointer）。

## 实验评估

### 实验设置

- **仿真平台**：Execution-driven x86 simulator，详细建模了 port contention、queuing effects、bank conflicts（含 DRAM system）
- **处理器配置**：
  - 15-stage OoO core，4-wide decode/retire，8-wide issue/execute
  - 256-entry ROB，32-entry load-store queue，256 physical registers
  - L1-I: 32KB 4-way 2-cycle; L1-D: 32KB 4-way 4-bank 2-cycle
  - L2: 1MB 8-way 8-bank 15-cycle, 128B line, 32 MSHRs, LRU
  - Memory: 450-cycle min latency, 8 banks, 32B bus @ 5:1 frequency ratio
  - Baseline stream prefetcher: IBM POWER4/5 style, 32 streams, degree 4, distance 32
- **多核配置**：Private L2 per core, on-chip DRAM controller, memory request buffer = 32 × core_count
- **Workload**：
  - SPEC CPU2006/2000 中的 pointer-intensive benchmarks + Olden benchmark suite（共 14 个应用）
  - 生物信息学应用 pfast
  - 多核实验：12 个 2-benchmark 组合（2-core）、4 个 4-benchmark 组合（4-core）
  - Profiling 使用 train input set（SPEC）或 smaller training input（Olden），实际运行使用不同 input set
- **对比方案**：
  - Original CDP
  - Dependence-Based Prefetcher (DBP, ~3KB)
  - Markov Prefetcher (1MB)
  - GHB-based Global Delta Correlation (G/DC, 12KB)
  - Hardware Prefetch Filter (8KB)
  - Feedback Directed Prefetching (FDP)

### 关键结果

1. **单核性能与带宽**：ECDP + Coordinated Throttling 平均提升性能 22.5%（去掉 health 后 16%），同时降低带宽消耗 25%（去掉 health 后 27.1%），对比 baseline stream prefetcher。11 个 benchmark 性能提升超 5%，8 个 benchmark 带宽降低超 20%。

2. **Prefetch accuracy 大幅提升**：ECDP + Throttling 将 CDP accuracy 提升 129%，stream prefetcher accuracy 提升 28%（对比原始 CDP + stream prefetcher）。Very useful PGs 占比从 27% 提升至 68.5%。

3. **优于其他 LDS/correlation prefetcher**：分别比 DBP、Markov、GHB 高出 19%/7.2%/8.9% 性能（去掉 health 后 12.7%/7.1%/5%），且硬件开销仅 2.11KB，远低于 Markov（1MB）和 GHB（12KB）。

4. **多核有效性**：2-core 系统上 weighted speedup 提升 10.4%，bus traffic 降低 14.9%；4-core 系统上 weighted speedup 提升 9.5%，bus traffic 降低 15.3%。

### 结果分析

**ECDP 和 Coordinated Throttling 的协同效应**：两者单独使用分别提升 8.6% 和 9.4% 性能，但组合使用提升 22.5%——存在显著的正向交互。原因在于：
- ECDP 消除了 harmful PGs 产生的 useless prefetch，提高了 CDP 的 accuracy
- Coordinated throttling 管理了 stream prefetcher 和 ECDP 之间的资源竞争
- 具体例子：在 perlbench/bisort/health 中，stream prefetcher 因 coverage 低于 CDP 而 throttle down（Case 4），为 ECDP 的 useful prefetch 让出资源；在 gcc 中，ECDP 因 stream prefetcher 的 57% high coverage 而 throttle down，避免干扰

**Coverage 的小幅下降换取 accuracy 的大幅提升**：ECDP + throttling 略微降低了两个 prefetcher 的 coverage，但显著提升了 accuracy，net effect 是大幅的性能和带宽收益。

**Profiling input set 敏感性低**：使用不同 input set 做 profiling 与使用相同 input set 做 profiling，性能差异仅在 mst 上超过 1%（4%），说明 PG 的 beneficial/harmful 分类在不同 input 之间相当稳定。

**对非 pointer-intensive 应用无害**：在 remaining SPEC/Olden benchmarks 上，平均性能提升 0.3%，带宽降低 0.1%。

## 审稿人视角

### 优点

1. **问题定义精准，motivation 充分**：论文清晰地识别了两个独立但相关的问题——CDP 的 useless prefetch 和 hybrid prefetching 的资源竞争——并分别提出了针对性解决方案。Oracle study 和 bisort/mst 的案例分析提供了极有说服力的 motivation。

2. **硬件开销极低，实用性强**：整个方案仅需 2.11KB 额外存储（主要是 L2 cache 中每个 block 2 bit 的 prefetched bit），不在 critical path 上，不需要额外 thread context。相比 Markov（1MB）和 pointer cache（1.1MB），这是数量级的优势。

3. **Pointer Group 抽象设计精巧**：将 cache block 中 pointer 的 prefetch 决策映射为 load 指令级别的 bit vector hint，利用了 LDS 节点在内存中连续分配导致 pointer 位于固定偏移的这一 structural property，使得编译器的 profiling 信息可以简洁地编码并高效地传递给硬件。

4. **实验全面且公正**：与 5 种不同方案（包括 3 种 LDS/correlation prefetcher、hardware filter、FDP）进行了直接对比，并分别展示了 ECDP 和 throttling 的单独/组合效果。多核实验、profiling sensitivity 分析、non-pointer-intensive benchmark 验证都增强了结论的可信度。

5. **Coordinated throttling 的通用性**：虽然主要评估了 ECDP + stream 的组合，但 throttling 机制本身是 prefetcher-agnostic 的，可应用于任意 hybrid prefetching 系统，且论文实际展示了它与 GHB 组合的效果。

### 不足

1. **对 compiler 支持的依赖未充分讨论**：ECDP 需要修改 ISA（新增指令承载 hint bit vector）、需要 profiling compiler 支持。论文对 profiling 实现只给出了 sketch，未详细讨论编译时间开销、对 dynamic memory allocation pattern 变化的鲁棒性（论文自身也承认 dynamic allocation/deallocation 可能改变 cache block 中 pointer 的 layout）、以及在大型真实应用上 profiling 的可扩展性。

2. **32-bit 地址空间的局限性**：论文的 CDP 实现基于 x86 的 4-byte pointer（32-bit 地址），使用 8 bit compare bits。在 64-bit 地址空间中，pointer 占 8 字节，一个 64B cache block 中的 pointer 数量减半，compare bits 的选择和 false positive rate 都会变化。论文未讨论向 64-bit 系统的扩展性，而 2008 年时 64-bit 系统已经普及。

3. **Benchmark 选择和 health 的影响**：health benchmark 对平均结果有极大影响（22.5% vs 16%，158.4% 的单点提升），且论文引用了 Zilles 的工作指出 health 可以通过重写程序提升数量级性能。虽然论文同时报告了 w/ 和 w/o health 的结果，但 health 的存在可能误导读者对方案实际效果的判断。此外，15 个 benchmark 的规模偏小，缺少如数据库、图计算等 pointer-intensive real-world 应用。

4. **Coordinated throttling 的阈值选择缺乏系统性分析**：三个阈值（T_coverage=0.2, A_low=0.4, A_high=0.7）被描述为 "empirically determined but not fine-tuned"，但论文未提供 sensitivity analysis 来说明结果对这些阈值的敏感度。在不同系统配置（不同 cache size、memory latency、bandwidth）下，这些阈值可能需要显著调整。

5. **Memory system 建模的时代局限**：单通道 8 bank 的 DRAM 配置、450-cycle memory latency、32B bus 等参数反映了 2008 年前后的系统特征。在现代多通道、high-bandwidth memory 系统中，bandwidth pressure 的特征可能显著不同，coordinated throttling 的收益模式也可能改变。

### 疑问或值得追问的点

- **递归 prefetch 不应用 hint filtering 的设计选择**：论文提到对 CDP prefetch 取回的 block 不应用 compiler hint 而是 prefetch 所有 pointer。这一设计的 sensitivity 如何？是否存在递归 prefetch 中也有大量 harmful PGs 的情况？
- **Coordinated throttling 在 >2 prefetcher 场景的可扩展性**：论文提到"potentially可用于 >2 prefetcher"但标注为 ongoing work。当 prefetcher 数量增多时，每个 prefetcher 需要感知所有 rival 的 coverage，heuristic 的复杂度和有效性如何变化？
- **与 software prefetch 的关系**：ECDP 已经利用了 compiler 的 profiling 能力，为何不直接在编译器中插入 software prefetch instruction 来替代硬件 CDP？论文未讨论这一 alternative 的 trade-off。
- **Cache pollution 的量化分析不足**：论文多次提到 cache pollution 是 CDP 性能下降的主因，并给出了 bisort/mst 理想消除 pollution 后的提升数据，但缺少系统性的 pollution 量化指标（如 pollution-induced miss 占比）。
