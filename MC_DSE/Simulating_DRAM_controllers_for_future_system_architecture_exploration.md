---
title: "Simulating DRAM controllers for future system architecture exploration"
authors: "Andreas Hansson, Neha Agarwal, Aasheesh Kolli, Thomas Wenisch, Aniruddha N. Udipi"
venue: "ISPASS 2014"
year: 2014
---

# Simulating DRAM controllers for future system architecture exploration

## 基本信息

- **发表**：IEEE International Symposium on Performance Analysis of Systems and Software (ISPASS), 2014
- **作者单位**：ARM Ltd. (Cambridge & Austin) + University of Michigan (ACLP)
- **开源**：模型作为 gem5 的一部分开源发布（BSD License）

## 一句话总结

> 提出一种 event-based 的高层 DRAM controller 模型，集成于 gem5，以 7x 仿真加速实现与 DRAMSim2 相当的 system-level 精度。

## 问题与动机

**核心问题**：在 full-system simulation 中，如何高效且准确地建模 DRAM controller 对系统性能的影响？

**问题的重要性**：

1. 从 mobile 到 server 的计算系统日益依赖 massively parallel architectures（many-core CPU、GPU），对 memory bandwidth 和 latency 的需求急剧增长。
2. 使用过于简化的 memory model（fixed latency 或 throughput-based）会严重误导 design space exploration 的结论，多篇前人工作 [6][7][8] 已证实这一点。
3. 新兴 memory 技术（WideIO stacked DRAM、HMC）要求模型具备灵活性以适配不同 DRAM organization。
4. Full-system workload 动辄需要模拟数十亿 cycle，仿真性能是实际可用性的关键瓶颈。

**现有工作的不足**：

- **Cycle-based 模型**（DRAMSim2 [9]、DrSim [14]、USIMM [15]）：对 DRAM 内部状态做 cycle-by-cycle 建模，引入了大量对 system-level 性能无关紧要的细节，导致仿真速度慢、灵活性差，难以适配新 DRAM 接口。
- **Analytical 模型**（[6][7]）：完全抽象掉 controller architecture，无法捕捉 scheduling、page policy 等 controller 决策对性能的复杂影响。
- **已有 event-based 模型**（[5]）：虽采用 event-based 思路，但事件粒度仍基于完整的 DRAM state transition，与 DRAM 细节紧耦合，速度不够快，且无法探索 controller 架构本身。

## 核心方法

### 关键思路

**核心洞察**：system-level 性能主要取决于 controller 的调度决策和少数关键 DRAM timing constraint，而非 DRAM 内部的每一个 cycle 级状态变化。因此，建模重心应从 "模拟 DRAM" 转向 "模拟 controller"，只追踪 bank state、data bus occupancy 和 response timing 这几个关键事件即可以高精度近似完整的 DRAM 行为。

**关键设计原则**：通过精心筛选 DRAM timing parameter 的子集（而非穷举所有数十个 timing），构建 event-based（而非 cycle-based）模型，在不显著损失精度的前提下获得数量级的仿真加速。

### 技术细节

#### 1. Controller Architecture

模型捕捉了一个 generic modern DRAM controller 的架构：

- **Split read/write queues + shared response queue**：与 DRAMSim2 的 unified transaction queue 不同，采用独立的读写队列，更贴近实际 controller 设计（参考 Cadence Wide-I/O controller [17] 和 ARM DMC-342 [18]）。
- **Per-controller buffering**：不做 per-rank 或 per-bank 的 buffering，简化模型。
- **Write handling**：write 请求在进入 write queue 时即返回 response（early write response），降低写延迟对 system-level 的影响。Incoming read 会 snoop write queue，小于 burst size 的 write 会尽可能 merge。
- **无独立 command queue**：page policy 和 arbitration 直接在 read/write request queue 上操作，避免了分离 request queue 和 command queue 带来的不必要复杂度。
- **Address decoding**：支持 RoRaBaCoCh、RoRaBaChCo、RoCoRaBaCh 三种 mapping scheme，channel interleaving 由 gem5 的 crossbar 组件完成。
- **Sub-cache-line access 支持**：CPU cache line 可被拆分为多个 DRAM burst（适配 mobile DRAM 如 LPDDR3 窄 bus width 的场景），controller 内部 merge 处理，对 memory hierarchy 其余部分透明。

#### 2. DRAM Timings 选择

模型只建模最关键的 timing parameter（关键 trade-off：精度 vs 仿真速度）：

| 建模的 Timing | 含义 |
|---|---|
| tRCD, tRAS, tRP | row open/close 的核心时序 |
| tCL, tBURST | column access + data burst 传输（tBURST 隐式建模 tCCD） |
| tWTR | read/write switching penalty |
| tRRD, tXAW (generalized tFAW) | inter-bank activation constraints |
| tREFI, tRFC | refresh 时序（引起 latency spike） |
| Frontend/Backend latency | controller pipeline 延迟 + PHY/IO 延迟 |

**未建模的 timing**：rank-to-rank switching constraints、bank group 引入的差异（DDR4 相关）。

**关键设计选择**：不建模 command/address bus contention，因为其通常不是瓶颈（posted commands with additive latency 等机制使其设计上不构成 bottleneck）。只追踪 data bus occupancy。

#### 3. Scheduling

三层调度策略：

- **Page policy**（4 种变体）：
  - Closed page：每次 column access 后 auto precharge
  - Closed adaptive：如果队列中已有对同一 open row 的 pending access，则保持 row open
  - Open page：保持 row open 直到 bank conflict
  - Open adaptive：提前 precharge（如果有 different row 的 pending access 且无 same row access）
- **Read/write switching**：write drain mode，high water mark 触发强制切换到 write drain，drain 一定最小数量的 write 后才切回 read（除非 write queue 为空）。Low water mark 控制在无 read 时何时开始 drain write。
- **Request scheduling**：FCFS 和 FR-FCFS [21]。FR-FCFS 优先选择 row hit（open page 下），否则选择 first available bank。

#### 4. Event-based 建模技术（核心贡献）

追踪三类关键状态信息：

1. **Bank ready time**：每个 bank 何时可以执行下一条 command
2. **Bank precharge eligibility**：每个 bank 何时满足 tRAS 可以 precharge
3. **Bank activation eligibility**：每个 bank 何时允许被 activate（受 tRRD/tXAW 约束）
4. **Data bus availability**：data bus 何时变为空闲
5. **Refresh due time**：下一次 refresh 的到期时间

基于以上信息，模型计算下一次调度决策的最早时间点，直接跳到该事件，中间无活动的 cycle 全部跳过。调度一次 access 后，更新 bank state 和 bus availability，并计算 response data 的返回时间。

#### 5. System 集成

- 集成于 gem5 full-system simulator，利用 gem5 的 Python 配置框架、statistics framework、port interface。
- 支持任意 cache hierarchy、prefetcher、on-chip crossbar 拓扑。
- Multi-channel 通过多个独立 controller instance + crossbar address interleaving 实现。
- 支持 UMA/NUMA、heterogeneous memory（如 WideIO + LPDDR3 混合）、HMC（16 个 controller instance + crossbar）。

#### 6. Power 建模

使用 Micron power model [24]，收集 page hit rate、data bus utilization（读/写分别统计）、all-banks-precharged time 等统计信息进行 offline 功耗计算。不建模 low-power states 和 DLL/PLL lock/wake-up time（约 500 cycle 量级）。

### 与现有工作的区别

| 维度 | DRAMSim2 [9] | Analytical Models [6][7] | TLM Model [5] | **本文** |
|---|---|---|---|---|
| 建模粒度 | Cycle-based | 方程/分布 | Event-based（DRAM 级） | **Event-based（Controller 级）** |
| Queue 架构 | Unified transaction queue | 无 | - | **Split read/write queue** |
| 仿真速度 | 慢（baseline） | 最快 | 中等 | **比 DRAMSim2 快 7x** |
| Full-system 支持 | gem5, MARSS | gem5（[6]） | 无 | **gem5** |
| DRAM 技术灵活性 | DDR2/DDR3 | - | Wide I/O | **DDR3, LPDDR3, WideIO, HMC** |
| Controller 可探索性 | 有限 | 无 | 无（与 DRAM 紧耦合） | **高（参数化配置）** |
| 开源 | BSD | 无 | 无 | **BSD（gem5 一部分）** |

## 实验评估

### 实验设置

- **仿真平台**：gem5 full-system simulator
- **对比基准**：DRAMSim2（同样集成于 gem5 进行对比）
- **DRAM 配置**（synthetic validation）：DDR3, 2Gbit, 8x8, 666 MHz, single rank, single channel
- **CPU 配置**（full-system）：2 GHz OoO, 6-wide dispatch / 8-wide commit, 40-entry ROB, 32KB I-Cache, 64KB D-Cache, 512KB L2 Cache
- **Workload**：
  - Synthetic: 自研 DRAM-aware traffic generator（可控 page hit rate、bank 数量、R/W ratio）+ linear generator + random generator
  - Full-system: PARSEC benchmark suite（bodytrack, blackscholes, canneal, swaptions, fluidanimate, x264, vips）
  - Case study: canneal on 16 cores with 8MB shared LLC
- **对比方案**：DRAMSim2 with matched DDR3 timing parameters

### 关键结果

1. **Bandwidth 精度**：在 open-page read-only 场景下，两个模型均达到约 90% utilization，bandwidth 曲线趋势高度一致。Closed-page write-only 场景中，本文模型比 DRAMSim2 高约 15% utilization（归因于 write buffering/drain 策略差异带来的更好 bank parallelism 利用）。

2. **Latency 精度**：平均 latency 差异在 1% 以内。Mixed R/W closed-page 场景下，本文模型呈现 bimodal latency distribution（因 write drain 策略导致部分 read 被 write drain 阻塞），而 DRAMSim2 无此现象——这是 controller 架构差异的合理体现，非精度问题。

3. **仿真加速**：synthetic workload 平均 7x 加速，最高 10x。Full-system PARSEC 测试中，overall simulation time 减少 13%-20%（受限于 OoO core 仿真占主导时间）。对于 16-channel HMC 等配置，加速效果达到数量级。

4. **Full-system 保真度**：PARSEC benchmarks 上，IPC、L2 miss latency、DRAM bus utilization 与 DRAMSim2 高度相关（Figure 8 中 ratio 接近 1.0），同时 simulation time ratio 均 < 1（即本文更快）。

5. **Power 精度**：与 DRAMSim2 对比，最大功耗差异 8%，平均差异 3%。

6. **Case Study（DDR3 vs LPDDR3 vs WideIO）**：在 16-core canneal 上，WideIO 因高 bank access timing（tRCD, tRP 均为 18ns）导致最高 access latency；LPDDR3 因窄 bus width（32-bit）需将每个请求拆为两个 burst，引入额外 queuing latency，但第二个 burst 总是 row hit 从而降低了平均 bank latency。

### 结果分析

- Synthetic 实验通过系统性扫描 stride size × bank count × R/W ratio × page policy 的组合空间，验证了模型在不同 traffic pattern 和 controller policy 下的行为正确性。
- Full-system 验证表明，controller 架构差异（split vs unified queue、write drain 策略）在 system-level metric 上的影响极小，验证了 "pruning unnecessary details" 的核心假设。
- Case study 展示了模型通过纯参数变更即可适配不同 DRAM 技术的灵活性——DDR3、LPDDR3、WideIO 三种截然不同的 DRAM organization 无需修改任何代码。
- 论文未做 sensitivity analysis 来量化各个被省略的 timing parameter 对精度的贡献。

## 审稿人视角

### 优点

1. **定位精准，解决实际痛点**：准确抓住了 system-level exploration 需要 "fast enough & accurate enough" 的 DRAM 模型这一核心需求。Controller-centric 而非 DRAM-centric 的建模视角是有意义的洞察，在精度-速度 trade-off 上找到了合理的 sweet spot。

2. **工程影响力极大**：这就是 gem5 中至今广泛使用的 DRAM controller 模型的原始论文。作为开源 infrastructure 贡献，其长期影响远超单篇论文的学术贡献。BSD License + 完整集成于 gem5 生态，使得后续大量体系结构研究直接受益。

3. **Validation 方法论合理**：没有追求与 DRAMSim2 cycle-exact 一致（这是不必要且不合理的目标），而是明确以 system-level metric（IPC、miss latency、bus utilization）的 correlation 作为验证标准。Synthetic traffic generator 的设计（DRAM-aware, 可控 page hit rate 和 bank targeting）是一个有用的工具贡献。

4. **灵活性展示有说服力**：DDR3/LPDDR3/WideIO 三种 DRAM 的 case study 以及 HMC 的可行性讨论，有效论证了 controller-centric 建模的灵活性优势。

### 不足

1. **Scheduling 策略过于基础**：只实现了 FCFS 和 FR-FCFS，缺乏对更先进 scheduling 策略（如 ATLAS [22]、MISE [11]、PAR-BS 等）的支持和评估。虽然论文声称框架 extensible，但未验证添加复杂 scheduler 的难度和对 event-based 模型的影响。FR-FCFS 作为 baseline scheduler 的代表性在 multi-core fairness 场景下是不足的。

2. **省略的 timing 缺乏量化分析**：rank-to-rank switching delay、bank group constraint（DDR4 的 tCCD_L/tCCD_S 区分）等被省略的 timing 对精度的影响未做 sensitivity study。论文仅定性声称 "不显著影响 system-level 性能"，但在 multi-rank 或 DDR4 场景下这一论断可能不成立。

3. **Full-system workload 覆盖不足**：仅使用 PARSEC（shared-memory parallel, 且多为计算密集型），缺乏 memory-intensive 的 single-thread workload（如 SPEC CPU 的 mcf、lbm 等高 MPKI benchmark）和 server workload（如 YCSB、GAPBS）的验证。PARSEC 中多数 benchmark 的 memory 压力较低，难以充分暴露 DRAM controller 模型差异。

4. **Power 建模较为初步**：不支持 low-power states（self-refresh、power-down）和 DLL/PLL wake-up timing，对 mobile 场景（LPDDR 系列的核心使用场景恰恰依赖低功耗状态切换）的适用性打了折扣。仅做 offline power 计算，无法捕捉 power-aware scheduling 的交互效果。

5. **Write handling 差异的影响被低估**：本文模型的 write drain 策略与 DRAMSim2 差异显著（bimodal latency distribution），但论文以 "平均 latency 差异 < 1%" 轻描淡写。在 latency-sensitive workload（如实时系统、GPU texture fetch）中，tail latency 的差异可能非常关键，而论文未评估 tail latency。

6. **Scalability 验证不充分**：声称 16-channel HMC 场景有数量级加速，但未给出具体数据。Multi-core（16 cores）case study 仅跑了 canneal 一个 benchmark，统计意义不足。

### 疑问或值得追问的点

- 在 multi-rank 配置下，省略 rank-to-rank switching delay 对 bandwidth 的影响有多大？尤其在 2-rank 或 4-rank DDR3/DDR4 配置中，tRTRS/tWTRS 约 1-2 个 clock cycle，在高 utilization 场景下可能不可忽视。
- Event-based model 的 event 数量在高 memory 压力下如何增长？是否存在某些 traffic pattern 使得 event 密度接近 cycle-based 模型，从而丧失速度优势？
- 论文发表于 2014 年，当时 DDR4 已接近量产。模型对 DDR4 的 bank group 概念缺乏支持，这是否限制了其 "future system exploration" 的定位？（注：后续 gem5 社区已添加相关支持，但这不属于论文贡献。）
- Sub-cache-line access 的 merge 逻辑在高并发写场景下是否引入额外的 critical path？对 controller 的 frontend latency parameter 设置是否应相应调整？
