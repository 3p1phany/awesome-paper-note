---
title: "Memory Access Scheduling"
authors: "Scott Rixner, William J. Dally, Ujval J. Kapasi, Peter Mattson, John D. Owens"
venue: "ISCA 2000"
year: 2000
---

# Memory Access Scheduling

## 基本信息

- **发表**：ISCA (International Symposium on Computer Architecture), 2000
- **单位**：Stanford University, Computer Systems Laboratory
- **一作**：Scott Rixner（当时同时为 MIT EE 研究生）

## 一句话总结

> 通过对 pending memory references 进行 out-of-order scheduling，利用 DRAM bank/row/column 三维结构的非均匀访问延迟特性，大幅提升 sustained memory bandwidth。

## 问题与动机

**核心问题**：现代 DRAM 并非真正的 random access device，其内部 bank、row、column 三维结构导致不同访问模式间存在近一个数量级的 bandwidth 差异——同一 row 内连续 column access 只需 1 cycle/word，而跨 row 的 access 需要经历 precharge + row activation 共 6 cycles 的延迟。传统 memory controller 采用 in-order 策略按请求到达顺序逐一服务，完全没有利用这一结构特性。

**为什么重要**：processor 性能年增长 60%，而 memory bandwidth 年增长仅 10%，memory wall 问题日益严重。对于 media processor 而言，streaming data 缺乏 temporal locality、cache footprint 大，无法依赖 cache 缓解带宽压力，memory bandwidth 直接决定系统性能。

**现有工作不足**：

- Stream buffer / prefetch 方案（如 Jouppi 1990）只能隐藏 latency，无法根据 DRAM 3D 结构重排访问顺序。
- McKee 的 Stream Memory Controller (SMC) 仅在 stream 间调度（先发多个同一 stream 的请求再切换 stream），不做 stream 内部 reference 重排。
- CVMS 和 PVA 仅在 command 级做 row/closed 调度，不处理 indirect (scatter/gather) stream。
- 以上方案均无法处理 indirect stream 的随机访问模式。

## 核心方法

### 关键思路

论文将 DRAM 访问调度类比为 superscalar processor 的 instruction scheduling：通过在 memory controller 中 buffer 一组 pending references，按照 DRAM timing/resource constraints 选择最优的 operation 执行顺序（而非按到达顺序），从而最大化 bank-level parallelism 和每次 row activation 的 column access 数量。核心 insight 是：**DRAM 的 precharge/activate 延迟可以被其他 bank 的 column access 所隐藏，而同一 row 的多次 column access 成本极低**，因此调度的目标是 (1) 并行推进多个 bank 的访问，(2) 最大化 column-to-row access ratio。

### 技术细节

#### DRAM 时序模型

以 NEC µPD45128163 SDRAM 为例：4 个 internal bank，每 bank 4096 rows × 512 columns；运行在 125MHz；precharge latency = 3 cycles (24ns)，row activation latency = 3 cycles (24ns)，column access throughput = 1 cycle/word (8ns)。三种共享资源需仲裁：**bank 资源**（precharge/activate 期间独占 3 cycles）、**address lines**（所有操作共享）、**data lines**（read/write 切换需 1 cycle high-Z）。

#### Scheduler 架构

论文提出的 scheduler 架构（Figure 4）包含以下组件：

1. **Per-bank Pending Reference Storage**：每个 bank 维护一组 pending references，每条记录包含 valid、load/store、row address、column address、data、scheduling state（如 age、是否命中当前 active row）。实践中可用共享 buffer 实现（增加 bank address field），论文实验中使用 32-entry 共享 bank buffer。

2. **Per-bank Precharge Manager**：决定何时 precharge 本 bank。策略选择：
   - **Open policy**：仅当有 pending reference 指向其他 row 且无 pending reference 指向当前 active row 时才 precharge。适用于 row locality 高的场景。
   - **Closed policy**：当前 active row 无 pending reference 时立即 precharge。适用于 row locality 低的场景，可利用 auto-precharge 与最后一次 column access 重叠。

3. **Per-bank Row Arbiter**：当 bank 处于 IDLE 态时决定 activate 哪一 row。可选 in-order、priority (ordered/age-threshold/load-over-store)、most pending 等策略。

4. **Global Column Arbiter**：在所有 bank 的所有 pending references 中选一个 column access 发射。可选策略包括 most pending（选 pending reference 最多的 row，尽快释放 bank）和 fewest pending（选 pending reference 最少的 row，缩短 row 占用时间）。

5. **Global Address Arbiter**：在 precharge、activate、column access 三类操作间仲裁共享 address lines。可选 column-first（降低已 active row 的 reference latency）、row-first（增加 bank parallelism）、precharge-first 等策略。

#### 调度算法分层

论文系统性地将调度决策分解为四个独立维度，每个维度可选不同 policy，组合构成完整的 scheduling algorithm：

| 维度 | 选项 |
|------|------|
| Column Access 选择 | in-order / priority (ordered, age-threshold, load-over-store) / most pending / fewest pending |
| Precharge 策略 | open / closed |
| Row Activation 选择 | in-order / priority / most pending |
| Address Arbiter 策略 | column-first / row-first / precharge-first |

论文重点评估了以下算法：

- **In-order (baseline)**：所有决策均基于最老的 pending reference，即传统 FCFS。
- **First-ready**：所有决策使用 ordered priority——选择最老的、当前可执行的 reference。关键改进在于：当最老 reference 的 bank 在 precharge/activate 时，可推进其他 bank 的 reference。
- **col/open, col/closed, row/open, row/closed**：column access 和 row activation 均使用 ordered priority，precharge 分别使用 open/closed，address arbitration 分别使用 column-first/row-first。
- **load/col/open, load/row/open**：在 col/open 和 row/open 基础上增加 load-over-store priority，解决 latency-sensitive 应用（如 FFT）中 store 阻塞 load 的问题。

#### 关键 Trade-off

1. **Open vs. Closed precharge**：open 利于 row locality 高的访问（如 bit-reversed FFT、unit stride），closed 利于 random access 模式（减少未来访问的 activate 等待时间）。col/closed 存在"过早 precharge"风险——column access 被快速消耗完后 bank buffer 为空，导致不必要的 precharge-reactivate。

2. **Column-first vs. Row-first**：两者在大多数 benchmark 上差异很小。Column-first 优先服务已 active row 的 reference（降低 latency），row-first 优先推进 idle bank 的 activate（增加 parallelism）。仅在 FFT 中 col/open 因 store stream 延迟 load stream 而表现更差。

3. **Bank buffer size**：16 entries 足以让所有 application 达到峰值性能；增大到 64 entries 仅对 trace 有边际改善（27% → 30%）。这表明 moderate buffer 即可捕获足够的 reordering 机会。

### 与现有工作的区别

| 维度 | 本文 Memory Access Scheduling | McKee SMC (1995) | CVMS (Corbal 1998) |
|------|------|------|------|
| 调度粒度 | 单个 memory reference 级别 | Stream 间调度（切换 stream 前先多发几个同一 stream 的 ref） | Command 级（base+stride），memory bank controller 间做 row/closed |
| Indirect stream 支持 | 支持（通过 per-reference 调度） | 不支持 | 不支持 |
| Intra-stream 重排 | 支持 | 不支持 | 不支持 |
| 策略空间 | 系统性的多维策略组合 | 单一策略 | 单一 row/closed |

## 实验评估

### 实验设置

- **仿真平台**：基于 Imagine stream processor 的 cycle-accurate 模拟器（非通用模拟器，为定制实现）。
- **处理器配置**：Imagine stream processor @ 500MHz，峰值 20 GFLOPS (SP FP) / 20 GOPS (32-bit int)；64KB Stream Register File (SRF)；2 个 address generator，4 个 interleaved external memory bank controller，2 个 reorder buffer。
- **DRAM 配置**：每个 memory bank controller 连接 2 片 NEC µPD45128163 SDRAM @ 125MHz（32-bit column access width）。单 controller 峰值 500MB/s，系统总峰值 2 GB/s。Bank buffer = 32 entries（默认）。
- **Workload**：
  - **Microbenchmarks**（5 个）：Unit Load、Unit（load+store）、Unit Conflict（同 bank 不同 row）、Constrained Random（64KB 范围）、Random（全地址空间）。无计算开销，仅压力测试 memory。
  - **Applications**（5 个）：FFT (1024-point × 10)、Depth (stereo depth extraction, 320×240)、QRD (192×96 矩阵 QR 分解)、MPEG (MPEG2 encoding, 3 frames 360×288)、Tex (triangle rendering 720×720 with texture mapping)。
  - 每个 application 同时报告"含计算"和"纯 trace（假设计算瞬时完成）"两种模式的结果。
- **Baseline**：In-order scheduling（所有决策仅考虑最老 pending reference），代表当时常见 memory controller 的做法。
- **Metric**：Sustained memory bandwidth (MB/s)，由于每种 scheduling 下应用执行的 memory reference 总数不变，bandwidth 提升直接等价于 execution time 缩短。

### 关键结果

1. **First-ready scheduling**（最简单的 reordering）：microbenchmarks +79%，applications +17%，application traces +40% bandwidth improvement over in-order baseline。Random benchmarks 改善最大（>125%），因为 reordering 显著增加了每次 row activation 的 column access 数。

2. **Aggressive reordering**（col/open, col/closed, row/open, row/closed 四种）：microbenchmarks +106%~144%，applications +27%~30%，application traces +85%~93%。几乎所有 trace 都能达到 peak bandwidth 的 80%+。

3. **Open vs. Closed 策略对比**：random/semi-random 访问（Tex、Constrained Random、Random）明显倾向 closed；bit-reversed FFT 明显倾向 open（因为 bit-reversed stream 对同一 row 有多次时间上分散的访问）。Unit Load 在 col/closed 下反而退化，因为 column access 被快速消耗后触发了不必要的 precharge。

4. **Load-over-store 优化**：针对 FFT 这一 latency-sensitive 应用，load/col/open 和 load/row/open 使 FFT trace 达到 peak bandwidth 的 97%+，而不影响其他 application 的性能。

### 结果分析

- **Bank buffer size sensitivity**：16 entries 是 application 的 sweet spot，Unit 和 Constrained Random 在更大 buffer 下仍有边际收益（因为并发 stream 多，reordering window 更大时可挖掘更多 parallelism）。
- **Compute-bound 场景**：MPEG 在 application 模式下几乎无改善（compute-bound，SRF 有效捕获了 data locality），但其 trace 在 aggressive scheduling 下可达 90%+ peak bandwidth，表明随着未来 processor 算力增长，scheduling 的收益空间会扩大。
- **Column-first vs. Row-first**：几乎所有 benchmark 上无显著差异，说明两者的 trade-off（latency reduction vs. parallelism increase）在这些 workload 下基本持平。
- 论文未做 energy 或 area 分析，但引用了 Hitachi 的 test chip 数据：first-ready scheduler 在 0.18µm 工艺下仅 1.5mm²、26mW @ 100MHz，说明硬件开销可控。

## 审稿人视角

### 优点

1. **系统性的问题分解与设计空间探索**：论文将 memory access scheduling 分解为 precharge、row activation、column access、address arbitration 四个独立决策维度，每个维度定义了清晰的 policy 选项（Table 1），这种模块化的抽象极具启发性，为后续研究（如 FR-FCFS 等）奠定了概念框架。

2. **问题提出的前瞻性**：2000 年 ISCA 上明确提出 DRAM 的 3D 结构特性是可以被 scheduler 利用的关键 observation，并给出了从 in-order 到 aggressive reordering 的完整谱系，在 memory controller 调度这一方向上具有开创性意义。Figure 1 的 56 cycles vs. 19 cycles 例子非常直观有力。

3. **实验设计合理**：同时包含 microbenchmark（压力测试极限）、application（真实性能）、application trace（去除 compute bottleneck 后的潜力评估）三个层次，能够清晰区分当前收益与未来潜力。Load-over-store 优化的引入也展示了针对特定 workload 特征的 fine-tuning 能力。

4. **开创了 memory scheduling 作为独立研究方向的先河**：本文是后续大量 DRAM scheduling 研究（STFM、ATLAS、TCM、PAR-BS、BLISS 等 fairness/QoS-aware scheduling）的直接奠基之作。

### 不足

1. **评估平台过于特殊，通用性存疑**：实验完全基于 Imagine stream processor，这是一个专用的 media/stream 处理器架构，具有非常规则的 streaming 访问模式。论文未在通用 out-of-order processor 上评估，而通用处理器的 memory access pattern 更加不规则、cache filtering 后的 miss stream 特性也截然不同。因此论文的 bandwidth 改善数字（尤其是 93%~144%）能否在通用场景下复现，存在较大不确定性。

2. **缺乏 fairness 和 starvation 的深入分析**：虽然提到了 age-threshold priority 可以防止 starvation，但论文没有量化分析 aggressive reordering 对单个 reference 的 worst-case latency 影响。在多线程/多进程共享 memory 的场景下，这是一个关键问题（后续 Mutlu 等人的 STFM、PAR-BS 工作正是针对此不足）。

3. **策略空间探索不够充分**：虽然 Table 1 定义了丰富的 policy 组合空间，但实际只评估了 6 种算法（in-order, first-ready, col/open, col/closed, row/open, row/closed）。most pending 和 fewest pending 等更精细的 column/row 选择策略未做实验评估，也未分析其潜在收益。

4. **DRAM 模型较简单**：仅建模了单 channel、单 rank 的 SDRAM，未考虑 multi-rank、multi-channel 的调度复杂性（rank switching penalty、channel interleaving 等），也未涉及 DRDRAM 等替代架构的实际评估（尽管在文字中提及了 separate row/column address lines 的优势）。

5. **缺少与 cache 系统的交互分析**：论文聚焦于 cacheless 的 stream processor 场景，完全回避了 cache miss stream 经 filtering 后的特性对 scheduling 效果的影响。对于通用处理器，cache 过滤后的 miss pattern 可能已经具有较好的 row locality（或反之更差），这会显著改变 scheduling 的收益特征。

### 疑问或值得追问的点

- **Most pending column policy 的实际效果如何？** 论文定义了这个策略但未评估。直觉上，选择 pending reference 最多的 row 做 column access 能最大化 column-to-row ratio，可能优于简单的 ordered priority。
- **Auto-precharge 的使用细节**：论文提到当 column access 与 precharge 冲突时可用 auto-precharge 解决，但在 open policy 下 auto-precharge 会破坏 row 保持策略，这一矛盾如何处理？
- **32-entry bank buffer 是所有 bank 共享的**：论文指出这增加了 scheduler logic 复杂度，但未量化 arbitration logic 的面积/时序开销。对于更多 bank 的现代 DRAM（如 DDR4 的 16 bank groups），shared buffer 的 scalability 如何？
- **Application trace 模式假设计算瞬时完成**：这过于理想化，实际中 compute 与 memory access 的 overlap 决定了 memory scheduling 的真实收益上限。一个更现实的 sensitivity analysis 应该参数化 compute-to-memory ratio。
