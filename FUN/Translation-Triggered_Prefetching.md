---
title: "Translation-Triggered Prefetching"
authors: "Abhishek Bhattacharjee"
venue: "ASPLOS 2017"
year: 2017
---

# Translation-Triggered Prefetching

## 基本信息
- **发表**：ASPLOS 2017（Xi'an, China）
- **作者**：Abhishek Bhattacharjee, Department of Computer Science, Rutgers University

## 一句话总结
> 利用 DRAM page table walk 的结果，在 memory controller 中提前 prefetch replay 数据到 row buffer 和 LLC，消除 TLB miss 后的二次 DRAM 访问开销。

## 问题与动机

这篇论文聚焦于大内存服务器场景下 virtual memory 地址翻译对 DRAM 性能的影响。核心观察如下：

**观察 1：Page table walk 占据大量 DRAM 带宽。** 在 big-data workload（图分析、稀疏矩阵、机器学习等）中，20-40% 的 DRAM 引用是 page table walk 访问。这些应用使用高达 3-4TB 的地址空间，访问模式极度不规则，导致 TLB miss 非常频繁。

**观察 2：Page table walk 访问 DRAM 后，replay 数据访问几乎 100% 也要访问 DRAM。** 论文发现，当一个 memory reference 的 page table walk 需要访问 DRAM 时，98%+ 的情况下后续 replay（即真正的数据访问）也会 miss 到 DRAM。直觉很简单：如果一个 translation 冷到连 on-chip cache 都不在，那它指向的数据页必然更冷。

**现有工作的不足：** 过去的研究要么优化 TLB miss rate（如 superpage、大 TLB），要么优化 page table walk latency（如 MMU cache），但都忽略了 page table walk 完成后 replay 访问的开销。Figure 1 显示 replay 的 DRAM 访问（DRAM-Replay-Access）占总运行时间的 10-30%，几乎与 page table walk 本身的 DRAM 开销相当。

## 核心方法

### 关键思路

TEMPO 的核心洞察是：当 memory controller 处理一个 leaf page table entry 的 DRAM 读取时，它已经拥有了计算 replay 数据物理地址所需的全部信息。因此可以在 page table walk 完成、translation 回填 TLB 的这段 slack window 内，提前将 replay 数据 prefetch 到 DRAM row buffer 和 LLC 中。这是一种 **non-speculative** 的 prefetch——目标地址由 memory controller 精确计算，不存在 misprediction。

### 技术细节

#### 整体架构

TEMPO 需要修改两个硬件组件：page table walker 和 memory controller。不需要任何 OS 或应用层修改。

#### Page Table Walker 修改

1. **标记 leaf PT 请求**：当 page table walker 发出 leaf page table entry（例如 4KB 页的 L1 PT）的内存请求时，附加一个 identifier bit 标记这是一个可以触发 prefetch 的 page table 访问。

2. **携带 cache line 信息**：原始 memory reference 请求的是虚拟地址中特定 page 内的某个 cache line。page table walker 在发出 leaf PT 请求时，额外附加 6 bits（对于 4KB page，64 个 cache line 需要 6 bits 编码）来告知 memory controller replay 需要的是哪个 cache line。

#### Memory Controller 修改

Memory controller 增加以下功能：

**① Page table access 检测与触发（PT? 比较器）：** 当请求到达 memory controller 时，通过比较器检查 identifier bit，识别 leaf page table 访问。

**② Transaction Queue 处理：** 由于 page table 请求比普通请求多了 6 bits 的 cache line 信息，直接加宽整个 Tx Q 会增加约 25% 的面积。TEMPO 采用更聪明的方案：将一个 page table 请求拆成两个 Tx Q entry——第一个是正常的 page table 地址请求，第二个临时缓存 replay cache line 信息。

**③ Prefetch Engine（简单 FSM）：** 当 DRAM array 返回 page table entry 后，Prefetch Engine 从中提取物理页号（8 字节 PTE 中的 physical page number），与 Tx Q 中缓存的 cache line 信息拼接，得到 replay 的完整物理地址。然后：
   - 向 DRAM 发送 read 请求，将包含目标数据的 row 激活到 row buffer（row buffer prefetch）
   - 将目标 cache line 发送到 LLC（LLC prefetch）

#### Prefetching Timeliness 分析

prefetch 操作与 translation 回填 cache/TLB 以及 replay 流水线前端操作并行执行，存在一个 slack window。Intel Haswell/Skylake 上这个窗口通常 120+ cycles，而 DRAM array 到 row buffer 的 prefetch 需要 60-100 cycles，再到 LLC 额外 20-30+ cycles，通常能在窗口内完成。即使部分重叠，row buffer hit 仍然有效。

#### 与 Memory Scheduler 的交互设计

**单个 page table 请求排队时：** TEMPO 不立即发起 prefetch，而是在当前 row buffer 内容保持 open 10 cycles，等待可能到来的、映射到同一 row 的其他 page table 请求，以提升 row buffer hit rate。实验显示 10-cycle delay 提升 1-4% 性能。

**多个 page table 请求排队时：** TEMPO 扫描 Tx Q，优先调度映射到同一 DRAM row 的 page table 请求（利用 page table entry 在物理内存中的空间局部性），然后再调度 prefetch 请求，同样按 row 分组。Figure 8 详细展示了这一调度策略。

**与 BLISS fairness scheduler 的交互：**
- Counter increment：prefetch 请求以普通请求一半的权重计入 BLISS 的 per-CPU counter（非 prefetch 权重 2，prefetch 权重 1）
- Stream switching：page table 访问后，其 prefetch 必须在切换到其他应用流之前完成；prefetch 完成后再等待 15-cycle grace period 以保持 prefetched row buffer 内容 open

**与 sub-row buffer 的交互：** 在 8 个 1KB sub-row buffer 的架构中，TEMPO 分配 2 个 sub-row 专用于 post-translation prefetch，兼顾 prefetch 效果和其他访问的服务。

#### 硬件开销

- Page table walker：增大 0.5%
- Memory controller：增大 3%
- On-chip network bandwidth：因更大的消息尺寸有微小增加
- 使用 Verilog 实现，Synopsis 32nm 工艺综合

### 与现有工作的区别

**vs. Traditional cache prefetcher（如 IMP [MICRO'15]）：** IMP 等 prefetcher 预测未来地址，存在 misprediction 风险。TEMPO 是 non-speculative 的，目标地址由 memory controller 精确计算。更重要的是，TEMPO 与 IMP 是正交的，组合使用时性能提升更大（高达 40%），因为 IMP 成功 prefetch 非 page table 数据后，page table 和 replay 访问成为新的瓶颈。

**vs. TLB/MMU cache 优化（如 CoLT [MICRO'12], MMU caches [MICRO'13]）：** 这些工作减少 TLB miss rate 或加速 page table walk，但不处理 walk 完成后 replay 的 DRAM 延迟。TEMPO 是互补的——即使 TLB miss 仍然发生，TEMPO 消除 replay 的 DRAM penalty。

**vs. Superpage 技术（如 transparent hugepage [Linux], libhugetlbfs）：** Superpage 减少 TLB miss 数量。但论文实验显示，即使超过 50% 的内存使用 2MB superpage，3-4TB 规模的 big-data workload 仍有大量 DRAM page table 访问。甚至使用 1GB page 时 TEMPO 仍能提供 5%+ 的性能提升。

## 实验评估

### 实验设置

- **仿真平台**：两步法。第一步使用修改版 Pin 在真实 Intel Skylake + 4TB 内存系统上采集包含虚拟和物理地址的 memory trace。第二步将 trace 送入自研 cycle-level 仿真框架（建模 Skylake 风格 out-of-order core + 受 DRAMSim 启发的 DRAM timing simulator）。TEMPO 硬件使用 Verilog 实现并通过 Synopsis 32nm 综合。
- **硬件配置**：32 core, 4GHz, 3-wide issue, 8 MSHRs/core, 128-entry inst. window; 32KB L1$, 64-entry L1 TLB, 1024-entry L2 TLB; 32MB 32-way LLC; 64-entry read/write queue, FR-FCFS + adaptive row policy; 4TB DDR3-1600, 800MHz, 1 rank/channel, 8 banks/rank, 64K rows/bank, 8KB row buffer, tRCD/tRAS = 12/30 cycles
- **Workload**：8 个 big-data workload（mcf, canneal, xsbench, graph500, lsh, spmv, symgs, illustris），均使用 3-4TB 内存；另外评估了全部剩余 SPEC 和 PARSEC workload 以验证无害性
- **对比 baseline**：
  - Row buffer policy：adaptive, open, closed
  - Memory scheduler：FR-FCFS [ISCA'00], BLISS [TPDS'16]
  - Row buffer organization：单 8KB row buffer, 8×1KB sub-row buffer（FOA/POA [ICS'12]）
  - Cache prefetcher：IMP [MICRO'15]
  - Page size：4KB only, 4KB+2MB transparent hugepage, 4KB+2MB libhugetlbfs, 4KB+1GB libhugetlbfs

### 关键结果

1. **性能提升 10-30%，能耗降低 1-14%**（vs. FR-FCFS + adaptive row baseline）。xsbench 等 DRAM page table 访问频繁的 workload 获得接近 30% 的性能提升。

2. **与 IMP prefetcher 组合时性能提升高达 40%**，比无 prefetcher 时的 10-30% 更高。这是因为 IMP 消除了大量非 page table 的 cache miss，使得 page table walk 和 replay 成为更突出的瓶颈。

3. **75%+ 的 replay 访问获得 LLC hit**，大部分 LLC miss 的 replay 仍获得 row buffer hit。仅有极少数（pathological case）TEMPO 无法覆盖。

4. **对小内存 workload 无害**：所有 SPEC/PARSEC 小 footprint workload 性能提升 1-2%，能耗降低约 1%，无负面影响。

5. **即使使用 1GB superpage，TEMPO 仍提供 5%+ 性能提升**。在仅使用 4KB page 时，性能提升可达 25%+。

### 结果分析

**Sensitivity analysis 覆盖全面：**

- **Superpage 分布**（Figure 13）：通过 memhog 制造不同程度的 memory fragmentation，控制 superpage 比例。Superpage 越多，TEMPO 收益越小，但即使在 libhugetlbfs 2MB 场景下仍有 8-25% 提升。
- **Row buffer policy**（Figure 14）：三种策略下 TEMPO 均有效。canneal 在 open-row 下收益最大（18%）因为多线程共享空间相邻数据；illustris 在 closed-row 下表现更好因为其访问局部性极差。
- **Anticipation delay**（Figure 15）：row buffer 保持 open 等待相邻 page table 请求的最佳延迟为 10 cycles，超过 15 cycles 性能下降。
- **BLISS prefetch weight**（Figure 16 左）：prefetch 权重为 0.5 时综合表现最佳。
- **Grace period**（Figure 16 右）：15 cycles 的 grace period 对 fairness metric 最优。
- **Sub-row buffer 分配**（Figure 17）：8 个 sub-row 中分配 2 个给 prefetch 为最佳平衡点，约 15-20% weighted speedup 提升。

## 审稿人视角

### 优点

1. **观察敏锐且有定量支撑。** 论文首次系统量化了 page table walk 后 replay 访问的 DRAM 开销（10-30% runtime），并发现 98%+ 的 DRAM page table walk 后必然跟随 DRAM replay 访问。这个 observation 非常强，为后续设计提供了坚实基础。

2. **设计 elegantly simple 且 non-speculative。** 与传统 prefetcher 依赖地址预测不同，TEMPO 利用 page table entry 内容精确计算 prefetch 目标，完全没有 misprediction 问题。硬件开销极小（MC 面积仅增 3%），且不需要 OS 或应用修改，工业界可行性很高。

3. **评估极为全面。** 论文系统评估了与多种 row buffer policy、memory scheduler、sub-row buffer 组织、cache prefetcher、superpage 配置的交互，覆盖了实际系统中可能遇到的各种组合。这种 thoroughness 在同类 DRAM 优化论文中少见。

4. **有 RTL 实现和面积/时序综合。** 不仅是仿真层面的评估，还用 Verilog 实现并通过 Synopsis 32nm 综合验证了硬件可行性，增强了说服力。

### 不足

1. **Trace-driven 仿真方法的局限性。** 论文使用 Pin 采集 trace 后离线 feed 给仿真器，这意味着 prefetch 对程序执行行为的反馈（如 prefetch 改变了 cache 状态后对后续指令 scheduling 的影响）无法被捕获。对于一个声称 10-30% 性能提升的优化，execution-driven simulation（如 gem5 full-system）会更有说服力。

2. **Workload 代表性有一定局限。** 虽然使用了 8 个 big-data workload，但它们都是极端 memory-intensive 的场景（3-4TB footprint）。论文没有评估中等规模 footprint（如几十 GB 到几百 GB）的 cloud workload（如 memcached、Cassandra、OLAP 查询），这些在实际数据中心中更常见，且可能有不同的 page table walk 特征。

3. **Multi-channel/multi-rank 场景讨论不足。** 论文假设 1 rank/channel 的简单配置。现代服务器通常有多 channel、多 rank 配置，page table entry 和其指向的 data page 可能分布在不同 channel，此时 prefetch 需要跨 channel 发起，对 timeliness 和带宽的影响需要额外分析。

4. **与虚拟化场景的交互未讨论。** 在虚拟化环境下，two-dimensional page table walk 将 4 级变为最多 24 次内存访问，TEMPO 的触发条件和 prefetch 逻辑需要重新设计。论文完全没有讨论这一重要场景。

5. **10-cycle / 15-cycle 等 magic number 的普适性存疑。** 这些 timing 参数（anticipation delay、grace period）在不同内存技术（DDR4/DDR5/HBM）、不同频率和时序下可能需要重新调优，但论文未提供自适应机制。

### 疑问或值得追问的点

- 论文中 96%+ 的 DRAM page table walk 访问都是 leaf PT（L1 PT），这意味着 MMU cache 对 upper-level PT 的命中率非常高。那么在 MMU cache 更小或更大的配置下，这个比例会如何变化？TEMPO 的触发频率和收益是否敏感于 MMU cache size？
- 在 NUMA 系统中，page table 和 data page 可能位于不同 NUMA node，prefetch 需要跨 node 发起。这种情况下 TEMPO 的 timeliness 和带宽影响如何？
- 论文提到 Tx Q 拆分为两个 entry 的设计，这是否会在高负载时导致 Tx Q 有效容量减半，反而增加 queuing delay？论文声称这比加宽 Tx Q 更优（避免 25% 面积增加），但没有给出 Tx Q 占用率的分析。
- TEMPO 的 row buffer prefetch 本质上是为 replay 数据做 ACT，这会不会与其他 pending request 产生 bank conflict，反而增加其他请求的延迟？论文的 fairness 评估仅限于 BLISS，未分析对单个 bank 级别的 conflict 增加情况。
