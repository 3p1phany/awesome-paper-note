---
title: "Multiple Sub-Row Buffers in DRAM: Unlocking Performance and Energy Improvement Opportunities"
authors: "Nagendra Gulur, R Manikantan, Mahesh Mehendale, R Govindarajan"
venue: "ICS 2012"
year: 2012
---

# Multiple Sub-Row Buffers in DRAM: Unlocking Performance and Energy Improvement Opportunities

## 基本信息
- **发表**：ICS (International Conference on Supercomputing), 2012
- **机构**：Texas Instruments (India) & Indian Institute of Science, Bengaluru

## 一句话总结
> 将 DRAM bank 的单一大 row buffer 拆分为多个小 sub-row buffer，在几乎不改 JEDEC 标准的前提下同时提升性能和降低能耗。

## 问题与动机

本文要解决的核心问题是：在 multicore 场景下，DRAM 的单一大 row buffer 导致 **row buffer locality 严重退化**，同时带来 **高能耗**。

**为什么重要？**
- DRAM 能耗占系统总能耗的 30% 以上，其中 Activate/Precharge 操作是主要能耗来源。
- Multicore 架构下，来自不同 core 的请求在 memory controller 处交织（interleave），导致同一 bank 的 row buffer 被频繁替换，row hit rate 极低。
- 传统的大 row buffer（1024–2048 columns，跨所有 device 约 8KB–16KB）是为单核空间局部性设计的，在多核场景下成为"开销大、收益低"的结构。

**现有工作的不足：**
1. **Memory scheduling 方案**（FR-FCFS、PARBS、TCM 等）：通过请求重排提升 row hit rate，但需要复杂的 controller 逻辑，且不直接解决能耗问题。
2. **Smaller row buffer 方案**（如 Udipi et al., ISCA 2010）：减小 row buffer 降低能耗，但以 hit rate 下降和性能损失为代价。
3. **地址重映射方案**（page coloring、permutation-based interleaving）：软件层面优化，对硬件改动小但效果有限。
4. **以上方案均只解决性能或能耗中的一个维度**，缺乏同时改善两者的方案。

## 核心方法

### 关键思路

**Key Observation 1**：多核 workload 虽然空间局部性差，但在 DRAM page/row 层面表现出显著的 **temporal locality**。从 stack distance histogram 看，仅增加到 2 个 row buffer，hit rate 就能翻倍以上（如 milc、gromacs、soplex 等 memory-intensive 程序尤为显著）。

**Key Observation 2**：小 row buffer（256 columns）能捕获大 row buffer（1024 columns）超过 **90%** 的 spatial locality hits。这意味着缩小 row buffer 的空间局部性损失很小。

**核心 idea**：将每个 bank 的 1 个大 row buffer（1×1024 columns）替换为 4 个小 sub-row buffer（4×256 columns），在 buffer 总容量不变的前提下，用多个 buffer 捕获 temporal locality，用小 buffer 维持 spatial locality，同时因为每次 Activate/Precharge 操作的数据量减小而降低能耗。

### 技术细节

#### 1. Sub-Row Activation（子行激活机制）

核心挑战：DRAM 需要同时知道 row address（来自 RAS 命令）和部分 column address（用于 sub-row 选择），但 JEDEC 标准中 RAS 和 CAS 是分时复用的，中间有 tRCD（10–15ns）的间隔。

论文讨论了三种方案：
- **增加 pin**：最直接，增加 2–3 个 sub-row-select pin。但 pin count 增长缓慢，不可行。
- **Posted-RAS/CAS**：RAS 和 CAS 背靠背发送，DRAM 内部锁存。引入 1 cycle 额外延迟，被否决。
- **Double-Address-Rate（DAR）**：利用 DDR 的双倍数据率能力，将其扩展到地址线。RAS 在时钟上升沿发送，sub-row-select 在下降沿发送，零额外延迟。**这是论文采用的方案**。

在 DRAM array 内部，实现方式类似 [12]（Udipi et al.）的 hierarchical wordline：`row_select` 线横跨整行但仅参与 sub-row 解码（负载很小），在选中的 sub-row 处激活传统 wordline 读取整个 sub-row。

#### 2. Row-Buffer Selection（行缓冲选择机制）

引入多个 row buffer 后，memory controller 需要承担额外职责：
- **记录**每个 bank 中哪些 sub-row 当前在 row buffer 中（避免重复 Activate）
- **分配**新激活的 sub-row 到哪个 buffer
- **驱逐**时确保被替换的 sub-row 被 Precharge

设计决策：**所有管理逻辑放在 memory controller 端**，DRAM 端保持简单。Controller 维护类似 cache 的 metadata：valid bits、row/sub-row tags、recency bits。

对三种操作的信令方案：
- **Activate**：buffer selection 不在 timing critical path 上（因为要等 bitline 充放电），用 Posted-Buffer-Selection（RAS 后一个 cycle 发送 buffer selection bits）即可。
- **Precharge**：timing critical，用 DAR 方案同时传递 sub-row selection 和 buffer selection bits。
- **Column Access**：timing critical，同样用 DAR 方案。

#### 3. Row-Buffer Allocation（行缓冲分配策略）

默认采用 **LRU** 策略。此外论文提出两种高级策略：

**Fairness-Oriented Allocation：**
- 动机：多核场景下，低请求到达率的 core 因其 row 被其他 core 的请求驱逐，受到不成比例的 interference。
- 方法：为每个 core 维护 shadow row buffer 以估算 standalone hit rate，与实际 shared hit rate 对比。若差值 d ≥ 0.5（threshold），该 core 被分类为 Type-1（受干扰严重），为其分配 dedicated buffer。
- 分配规则：1 个 Type-1 core → 1 个 dedicated buffer + 3 shared；2 个 Type-1 → 各 1 个 dedicated + 2 shared；≥3 个 Type-1 → 退化为 LRU。
- 粒度：per-bank 分类，同一 core 在不同 bank 可能有不同分类。周期性重新分类以适应 phase change。

**Performance-Oriented Allocation：**
- 为每个 core c 和 bank b 计算 `rate_product[c][b] = num_memory_requests[c][b] × row_miss_rate[c][b]`。
- 高 rate_product 的 core 分类为 Type-1，获得 exclusive buffer（最多 2 个）+ shared access to remaining。
- 当 Type-1 core 数超过 buffer 数的一半时退化为 LRU。

#### 4. Early Precharge Scheduling

- 对 LRU buffer 提前执行 precharge（closed page policy），其余 buffer 保持 open page policy。
- 利用 idle cycle 插入 LRU buffer 的 precharge 操作。
- 理由：LRU buffer 最可能被驱逐，提前 precharge 可以隐藏未来 miss 时的 precharge latency。
- Trade-off：牺牲少量可能的 row hit（LRU buffer 被提前关闭），换取 miss latency 降低。

#### 5. Intra-Bank Parallelism (intraBP)

- 允许对一个 row buffer 的 column access 与另一个 row buffer 的 activate/precharge **并行执行**。
- 传统单 row buffer 时，bank 内所有操作严格串行；多 buffer 后，bank 内部电路本身已具备并行能力。
- 效果：提高 data bus 利用率，降低平均访问延迟（约降低 19%），特别有利于 bank-level parallelism 较低的程序。

### 面积开销

**DRAM 端**（以 4 sub-row buffers per bank 为例）：
- Sub-row select 线 + AND 门：4.9%（用 hierarchical wordline + CACTI 建模）
- Buffer selection 解复用器 + buffer_select 线：1.9%（2×2 布局）
- 总 buffer 容量不变（buffer-capacity-neutral）
- **总面积开销：6.8% per bank**

**Memory Controller 端**：
- 4 GB RAM（4 ranks × 8 banks）的 metadata 开销：512 bytes（baseline 已需 128 bytes）
- 可忽略不计

### 与现有工作的区别

| 维度 | MSRB (本文) | PARBS [Mutlu, ISCA'08] / TCM [Kim, MICRO'10] | Udipi et al. [ISCA'10] |
|------|-----------|----------------------------------------------|----------------------|
| 改动位置 | DRAM array（微小）+ MC | 仅 MC（复杂调度） | DRAM array |
| 性能 | +35.8% WS (quad) | +8.5% / +10.5% WS | 轻微下降 |
| 能耗 | −42% (quad) | 不直接优化 | 降低（小 row buffer） |
| 核心机制 | 多 buffer 捕获 temporal locality | 请求重排提升 row hit | 减小 row buffer 降能耗 |
| 与调度方案兼容 | ✓（正交，可叠加） | N/A | 不涉及 |

关键差异：MSRB 是从 **DRAM 微架构层面**出发，而调度方案是在 controller 层面做请求重排。两者正交互补——TCM+MSRB 和 PARBS+MSRB 的组合效果远超单独使用任一方案。与单纯缩小 row buffer 不同，MSRB 通过多 buffer 补偿了缩小带来的 hit rate 损失，实现了性能和能耗的双赢。

## 实验评估

### 实验设置
- **仿真平台**：M5 simulator + in-house DRAM simulator（精确建模 timing）
- **处理器**：3.2 GHz OOO, Alpha ISA
- **Cache 层次**：L1I 32KB DM / L1D 32KB 2-way / L2 shared（1/4/8/16 cores 对应 1/4/8/16 MB）
- **DRAM**：DDR3-1600H, BL=8, CL-nRCD-nRP=9-9-9, 4 个 1GB x16 device per rank, 8 banks/device, 65536 rows/bank, 1024 columns/bank
- **Memory Controller**：on-chip, 64-bit interface, 256-entry command queue, FR-FCFS scheduling, open-page policy, rank-bank-row-column interleaving
- **Controller 数量**：1/4/8/16 cores 对应 1/1/2/4 controllers
- **Workload**：SPEC CPU 2000 + SPEC CPU 2006 的多程序混合，按 L2 MPKI 混合不同 memory intensity（12 组 quad-core、8 组 eight-core、4 组 sixteen-core）
- **模拟方法**：每程序 fast-forward 9B insts → warm-up 500M insts → detailed 250M insts
- **功耗估计**：Micron PowerCalculator，IDD0 从 120mA 保守降至 95mA，column access power 增加 5%
- **对比 baseline**：1×1024（单大 row buffer）、PARBS、TCM、32-Bank（假设的高 bank 数配置）、closed page policy
- **Metrics**：Weighted Speedup (WS)、Harmonic Speedup (HS)、Fairness（min slowdown / max slowdown）

### 关键结果

1. **性能提升**：MSRB (4×256) 在 quad/8/16-core workloads 上分别提升 weighted speedup **35.8%**、**14.5%**、**21.6%**（对比 1×1024 baseline）。Row buffer hit rate 从 0.2 提升到 0.6（quad-core 平均）。

2. **能耗降低**：DRAM 能耗在 quad/8/16-core workloads 上分别降低 **42%**、**28%**、**31%**。Activate power 从平均 357mW 降至 127mW。能耗降低来自两个因素：小 row 的 ACT/PRE 能耗更低 + row miss 次数减少。

3. **远超调度方案**：PARBS 和 TCM 仅提升 8.5% 和 10.5% WS，而 MSRB 提升 35.8%。且 MSRB 可与它们叠加——TCM+MSRB 和 PARBS+MSRB 获得更高的 speedup。32-Bank 假设配置仅提升 5.9%，说明增加 bank 数不如多 buffer 有效。

4. **附加优化**：Early Precharge 额外提升 1.8%，IntraBP 额外提升 4.7%，两者组合额外提升 **5.9%**（均在 MSRB 基础上）。Fairness-Oriented Allocation 提升 fairness **43%**，Performance-Oriented Allocation 提升 WS 至 **40.9%**（比 MSRB 的 35.8% 再增 1.9%）。

### 结果分析

- **单核也有效**：MSRB 在单核 SPEC workloads 上平均提升 7.1% IPC，459.GemsFDTD 提升 148%，462.libquantum 提升 21%。
- **Sensitivity**：2×512 配置的 hit rate 为 0.48（vs 4×256 的 0.60），WS 提升 34.5%（vs 35.8%），说明增加 buffer 数比增大 buffer size 更有效。
- **Early Precharge 的 trade-off**：在 Q5 workload 中 Early Precharge 反而降低性能（loss of row hits > latency gain），但与 IntraBP 组合后恢复。Q9 则大幅受益于 Early Precharge（+30%）。
- **单 controller 时 MSRB 更有效**：8 核单 controller 时 MSRB 提升 16% WS（vs 双 controller 的 14.5%），因为 controller 竞争更激烈时 row buffer locality 退化更严重。
- **Performance-Oriented Allocation** 对 memory-intensive benchmark 的提升可达 25%，且不影响非 memory-intensive 程序的性能。

## 审稿人视角

### 优点

1. **问题抓得准，motivation 清晰有力**：stack distance histogram 的实验数据非常有说服力地论证了"temporal locality 存在但被单 buffer 浪费"和"小 buffer 几乎不损失 spatial locality"这两个关键 observation。这直接驱动了 MSRB 的设计。

2. **实用性强，工程可行性好**：论文严肃对待了 JEDEC 标准的约束，提出的 Double-Address-Rate 方案不需要额外 pin，DRAM 内部改动仅 6.8% 面积开销，controller 端开销可忽略，总 buffer 容量不变。这不是一个纯学术方案，而是有产业落地潜力的设计（作者本身来自 TI）。

3. **同时改善性能和能耗**：这是本文最大的卖点。大多数 DRAM 优化方案在性能和能耗之间做 trade-off，而 MSRB 通过 row buffer reorganization 实现了双赢。能耗降低的两个来源（小行 + 少 miss）分析清晰。

4. **正交性好、扩展性丰富**：与已有调度方案正交且可叠加，Early Precharge 和 IntraBP 是 MSRB 独有的优化机会，Fairness/Performance-Oriented Allocation 展示了 design space 的灵活性。

5. **对比实验较为全面**：与 PARBS、TCM 两个 state-of-the-art scheduler 以及 32-Bank 假设配置进行了对比，说服力较强。

### 不足

1. **功耗建模过于简化**：使用 Micron PowerCalculator 进行功耗估算，IDD0 从 120mA "保守估计"降至 95mA，缺乏 circuit-level 验证。Sub-row activation 的额外解码逻辑、buffer selection 解复用器、DAR 信令的功耗均未精确建模。Column access power 增加 5% 的假设缺乏依据。对于一篇声称"同时降低能耗"的工作，功耗模型的粗糙是明显短板。

2. **面积开销估计可信度有限**：论文自己在脚注中承认 CACTI 与商业 DRAM 设计差异可能很大，但仍以此为主要面积评估工具。6.8% 的 per-bank 面积开销在 DRAM 这种极度 density-optimized 的设计中并不算小，实际可行性存疑。

3. **Workload 和仿真方法论的局限**：
   - 仅使用 SPEC CPU 2000/2006 的多程序混合，缺乏真正的多线程 workload（如 PARSEC、SPLASH-2）和服务器 workload 的评估。多程序混合的 bank 访问模式与真实多线程程序差异较大。
   - 250M instructions 的 detailed simulation 对于评估 DRAM 层面效果偏短。
   - M5 + in-house DRAM simulator 的可复现性不如使用公开工具（如 DRAMSim2/3、Ramulator）。

4. **IntraBP 的实现细节不够充分**：论文承认 IntraBP 需要"minor DRAM interface changes to permit multiple outstanding accesses to each bank"，但对具体需要什么改动、timing constraint 如何满足、是否违反 JEDEC 标准等关键问题语焉不详。这与论文"minimal changes to JEDEC"的主张存在矛盾。

5. **Allocation 策略的探索较浅**：Fairness-Oriented 和 Performance-Oriented Allocation 的 threshold 选择缺乏系统性的 sensitivity analysis。Shadow row buffer 的概念直接借用了 STFM [3] 的思路但未充分 credit。策略仅在 quad-core 上评估，8/16-core 场景下的表现未知。

6. **缺乏与 sub-ranking/mini-rank 方案的直接能耗对比**：Minirank [13] 和 MC-DIMM [29] 也是以降低能耗为目标的 DRAM 改造方案，论文仅在 related work 中文字提及，未进行定量对比。

7. **DAR 方案的可行性论证不足**：将 DDR 的双倍率传输从 data pin 扩展到 address pin 在工程上并非 trivial——address pin 的信号完整性要求、setup/hold time margin、PCB routing 等均未讨论。

### 疑问或值得追问的点

- 在现代 DRAM 标准（DDR4/DDR5）下，bank group 的引入和更深的 prefetch 使得 row buffer 的角色发生了变化。MSRB 的收益在新标准下是否依然显著？DDR5 的 same bank group timing penalty 是否会影响 IntraBP 的效果？
- 4×256 的配置是否是最优的？论文仅对比了 2×512，但 8×128 等更极端配置的 trade-off 分析缺失（论文仅提到 256 以下会对单核有明显损失，但未给出详细数据）。
- Row buffer management 放在 controller 端意味着所有 bank 状态都要在 controller 维护。当 rank/bank 数量进一步增大时（如 HBM 的 16/32 banks per channel），metadata 和管理复杂度的 scalability 如何？
- Early Precharge 的触发条件（idle cycle）在高负载场景下是否难以找到空闲周期？这是否限制了该优化在 memory-intensive workload 上的效果？
- 论文的能耗模型完全忽略了 controller 端的功耗增加。考虑到 shadow row buffer（Fairness-Oriented Allocation）和复杂的 metadata 管理，这部分功耗是否不可忽视？
