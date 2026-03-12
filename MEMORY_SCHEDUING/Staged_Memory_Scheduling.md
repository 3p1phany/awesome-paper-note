---
title: "Staged Memory Scheduling: Achieving High Performance and Scalability in Heterogeneous Systems"
authors: "Rachata Ausavarungnirun, Kevin Kai-Wei Chang, Lavanya Subramanian, Gabriel H. Loh, Onur Mutlu"
venue: "ISCA 2012"
year: 2012
---

# Staged Memory Scheduling: Achieving High Performance and Scalability in Heterogeneous Systems

## 基本信息
- **发表**：ISCA (International Symposium on Computer Architecture), 2012
- **作者单位**：Carnegie Mellon University, Advanced Micro Devices (AMD)
- **关键词**：Memory Scheduling, CPU-GPU Heterogeneous Systems, Memory Controller, Fairness

## 一句话总结
> 将 memory controller 的功能解耦为三级流水线（batch formation → batch scheduling → DRAM command scheduling），以低复杂度实现异构 CPU-GPU 系统中的高性能与公平性调度。

## 问题与动机

### 核心问题
在 CPU-GPU 集成异构系统中，GPU 产生的海量 memory request 严重干扰 CPU 应用的内存访问，导致 CPU 性能下降和饥饿现象。现有的 application-aware memory scheduling 算法（设计于 CPU-only 场景）在面对 CPU-GPU 混合流量时表现 non-robust。

### 问题的根源——Visibility 不足
GPU 应用的 memory intensity 远超 CPU 应用（论文数据显示最高内存密度的 CPU 应用 mcf 的 MPKI 仅为最低内存密度 GPU 应用 Bench01 的 42.8%）。当 CPU 和 GPU 共享 MC 的 request buffer 时，GPU 的大量请求占据了绝大部分 buffer entries，使得 MC 无法充分观察各 CPU 应用的内存行为特征（row-buffer locality、BLP 等）。

增大 request buffer 可以改善 visibility，但带来严重的硬件实现挑战：
- 每个 buffer entry 需要 CAM 来比较 row index 与当前 open row
- 需要大规模 priority encoder / comparison tree 来选择最高优先级请求
- DRAM timing constraint 检查逻辑需覆盖每个 buffer entry
- 本质上等价于一个大规模 out-of-order scheduler，而现代 CPU 的 instruction scheduler 通常也只有 32–64 entries

### 现有工作的不足
论文系统分析了三种 state-of-the-art 调度器在 CPU-GPU 场景下的表现：

1. **FR-FCFS**：优先 row-buffer hit，GPU 因高 RBL 被持续优先，CPU 性能和公平性极差
2. **ATLAS**：基于 attained service 排序，由于 CPU 应用 memory intensity 低而被持续优先，GPU 帧率严重下降
3. **TCM**：基于 memory intensity 将应用分为 latency-sensitive 和 bandwidth-sensitive 两个 cluster，但在 CPU-GPU 场景下，GPU 极高的 intensity 导致 clustering threshold 偏移，部分 high-intensity CPU 应用被错误分类到 low-intensity cluster，造成严重不公平

## 核心方法

### 关键思路
**Key Insight**：传统 monolithic MC 将 row-buffer locality 优化、application-aware 优先级调度、DRAM timing/command 调度三个功能耦合在一个大型 centralized request buffer 上实现。SMS 的核心思想是将这三个功能 **解耦** 到三个独立的、更简单的 stage 中，每个 stage 使用分布式 FIFO 结构，从而在保持调度质量的同时大幅降低硬件复杂度。

**Key Observation**：GPU 应用同时具有高 memory intensity、高 RBL、高 BLP 的特征组合（与 CPU 应用截然不同），这使得基于 row-buffer hit 优先的调度策略会系统性地偏向 GPU，而基于 memory intensity 分类的策略又会因 GPU 的极端 intensity 而失效。

### 技术细节

#### 整体三级架构

```
Stage 1: Batch Formation (per-source FIFOs)
    ↓ ready batches
Stage 2: Batch Scheduler (SJF / Round-Robin)
    ↓ drained requests
Stage 3: DRAM Command Scheduler (per-bank FIFOs, in-order)
```

#### Stage 1 — Batch Formation
- **结构**：每个 source（CPU core 或 GPU）一个独立 FIFO
- **功能**：将来自同一 source、访问同一 DRAM row 的连续请求聚合成一个 batch
- **Batch 就绪条件**（满足任一即可）：
  - 新到达请求访问不同的 row
  - batch 中最老请求的 age 超过阈值（threshold age）
  - FIFO 已满
- **Batch formation 严格按到达顺序**（in-order），论文实验表明 out-of-order grouping 仅带来 <5% 的性能差异，且可能引入不公平
- **Memory intensity 分类与 bypass 机制**：
  - 低 intensity（<1 MPKC）：threshold age = 0，请求直接 bypass 前两个 stage，进入 DCS
  - 中 intensity（1–10 MPKC）：threshold age = 50 cycles
  - 高 intensity（>10 MPKC）：threshold age = 200 cycles
  - GPU：threshold age = 800 cycles
  - 全局轻载 bypass：当 DCS 中总请求数 <16 时，所有应用请求直接进入 DCS

注意：SMS 使用 MPKC（misses per kilo cycles）而非 MPKI 来分类 memory intensity，因为 MC 通常无法获取各应用的 instruction count，这避免了额外的实现开销。

#### Stage 2 — Batch Scheduler
- **两状态机**：pick → drain → pick → ...
- **Pick 状态**：从所有 source FIFO 中选择一个 ready batch
  - 以概率 *p* 使用 **SJF 策略**：选择 in-flight request 数量最少的 source 的最老 ready batch（倾向于 latency-sensitive 的低 intensity 应用）
  - 以概率 *1−p* 使用 **Round-Robin 策略**：轮询各 source FIFO（保证 bandwidth-intensive 应用的带宽）
- **Drain 状态**：将选中 batch 的请求逐周期出队到 Stage 3 的 per-bank FIFO
- ***p* 是可配置参数**：
  - *p* 高（如 0.9）→ 倾向 CPU（SJF 更频繁，低 intensity 应用优先）
  - *p* 低（如 0）→ 倾向 GPU（round-robin 更频繁，GPU 的长 batch 更常被选中）
  - 可由系统软件静态或动态配置，甚至可通过新增 ISA 指令实时调整

#### Stage 3 — DRAM Command Scheduler (DCS)
- **结构**：每个 DRAM bank 一个 FIFO（DDR3 每 rank 8 个 bank → 8 个 FIFO）
- **调度策略**：严格 in-order，仅考虑每个 per-bank FIFO 的 head entry
- **DRAM timing/protocol 检查**：仅针对 8 个 head entry（而非传统方案中数百个 buffer entry），复杂度大幅降低
- **Bank 间仲裁**：round-robin
- **设计理由**：batch formation 已保证 row-buffer locality，batch scheduler 已完成 application-aware 优先级决策，DCS 无需再做 reordering

#### 硬件资源开销
| Stage | 结构 | 规模 |
|-------|------|------|
| Stage 1 | CPU core FIFO | 16 × 10 entries = 160 entries |
| Stage 1 | GPU FIFO | 1 × 20 entries = 20 entries |
| Stage 2 | Memory request counters | 16 × 5-bit (CPU) + 1 × 10-bit (GPU) |
| Stage 3 | Per-bank FIFO | 8 × 15 entries = 120 entries |
| **总计** | | **300 entries aggregate** |

但关键在于：任意时刻 SMS 仅需考虑 17 个 batch head（Stage 2）和 8 个 FIFO head（Stage 3），而非 300 个 entry 全部参与比较。

### 与现有工作的区别

| 特性 | FR-FCFS | ATLAS | TCM | **SMS** |
|------|---------|-------|-----|---------|
| 请求缓冲区 | Centralized, fully-associative | Centralized | Centralized | **Distributed FIFOs** |
| Row-buffer locality | 全局 row-hit 优先 | 全局 row-hit 优先 | 全局 row-hit 优先 | **Stage 1 per-source batch** |
| Application 优先级 | 无 | Attained service 排序 | Intensity-based clustering | **SJF/RR 概率混合** |
| GPU 处理 | 因高 RBL 被隐式优先 | 因高 attained service 被持续降权 | 与高 intensity CPU 混入同一 cluster | **通过 *p* 参数显式控制** |
| 硬件复杂度 | 大型 CAM + priority encoder | 更复杂的 ranking logic | Clustering + shadow RBL tracking | **简单 FIFO + MIN tree** |
| 可配置性 | 无 | 无 | 无 | **SJF 概率 *p* 动态可调** |

关键差异：
1. **vs. ATLAS**：ATLAS 在 CPU-GPU 场景下几乎一直 deprioritize GPU（因 GPU attained service 极高），导致 GPU 帧率严重下降且 CPU 间不公平（strict rank order）。SMS 通过 batch-level 调度避免了 strict ranking。
2. **vs. TCM**：TCM 的 clustering threshold 在 GPU 极端 intensity 下失效，导致 misclassification。SMS 不依赖全局 clustering，而是通过 SJF 自然区分 latency-sensitive 和 bandwidth-intensive 应用。
3. **vs. FR-FCFS**：FR-FCFS 的 row-hit 优先策略在 GPU 高 RBL 下等同于持续优先 GPU。SMS 将 row-buffer locality 优化限定在 per-source batch 内部，避免跨应用的 row-hit 偏向。

## 实验评估

### 实验设置
- **仿真平台**：In-house cycle-level simulator（非公开模拟器）
- **CPU 配置**：16 个 x86-like 3-wide out-of-order cores，128-entry ROB，3.2GHz，32KB L1 private + 8MB L2 shared (16-way LRU)
- **GPU 配置**：类似 AMD Radeon HD 5870，20 cores × 16 SIMD FUs，最大吞吐 1600 ops/cycle，800MHz
- **内存系统**：DDR3-1600，4 channels / 1 rank / 8 banks per channel，2KB row buffer，tRCD/tCAS/tRP = 12.5ns，MC request buffer = 300 entries
- **Workload**：
  - CPU：26 个 SPEC CPU2006 benchmarks，按 MPKI 分为 Low（≤1）、Medium（1–15）、High（>15）三类
  - GPU：8 个 GPU benchmarks（3DMark + 商业游戏 trace），MPKI 范围 173.5–419.7
  - 105 个多程序工作负载（16 CPU + 1 GPU），7 种 intensity mix（L, ML, M, HL, HML, HM, H）
- **对比 Baseline**：FR-FCFS, CFR-FCFS (CPU-prioritized FR-FCFS), ATLAS, TCM, CTCM (CPU-prioritized TCM)
- **性能指标**：
  - CPU Weighted Speedup（Eqn. 1）
  - GPU Frame Rate (FPS)
  - CGWS = CPU_WS + GPU_weight × GPU_Speedup（Eqn. 3）
  - Unfairness = max slowdown（Eqn. 4，使用 harmonic mean 报告）

### 关键结果

1. **CPU 性能**：SMS₀.₉ 在 105 个工作负载上平均提升 CPU weighted speedup **22.1%（vs. ATLAS）** 和 **35.7%（vs. TCM）**，同时 GPU 帧率降低 18.1%/26.7%，但仍维持在 >30 FPS（除 HM 类别外）。

2. **公平性**：GPUweight=1 时，SMS₀.₉ 的 unfairness 比 FR-FCFS/ATLAS/TCM 分别改善 **244.6%/47.6%/205.7%**。这得益于 SJF 概率调度避免了 strict ranking 和 misclassification。

3. **硬件效率**：在 300-entry request buffer 配置下，SMS 比 FR-FCFS **面积减少 46.3%，漏电功耗减少 66.7%**（180nm 标准单元综合）。ATLAS/TCM 的 ranking logic 更复杂，SMS 的优势更大。

4. **可配置性**：通过调节 *p*（0 到 0.9），SMS 可在 CPU 优先（p=0.9）和 GPU 优先（p=0）之间灵活切换。定义 SMS_Max 为每个 GPUweight 下最优 *p* 的配置，SMS_Max 在所有 GPUweight 值下均提供最优 CGWS。

### 结果分析

#### Sensitivity Analysis
1. **Threshold Age**：增大 threshold age → GPU 帧率提升（更长 batch，更好 RBL），但 CPU 性能下降（latency-sensitive 应用等待更久）。减小 threshold age → batch 变短，RBL 下降，GPU 性能损失大于 CPU（因 SMS₀.₉ 本就优先 CPU）。
2. **DCS FIFO Size**：减小 DCS FIFO → GPU 被更多限制（难以同时填满多个 bank FIFO），CPU 性能和公平性提升。增大至 20 entries 以上收益极小。
3. **Core Count Scalability**：从 2 到 16 核，SMS 始终优于 baseline。16 核时公平性优势最大（47.4% vs. ATLAS），因为核数增多导致 per-core 带宽下降，interference 加剧，SMS 的 throttling 效果更明显。
4. **Channel Scalability**：8 channel 时 SMS 的 GPU 帧率损失收窄至 13.4%（vs. FR-FCFS），而 CPU 性能提升 54.7%。2 channel 时 GPU 帧率降至 30 FPS 以下，但可通过降低 *p* 缓解。

#### CPU-only 场景
SMS 在纯 CPU 16 核系统中，性能比 ATLAS/TCM 仅低 8.8%/4%，但公平性提升 1.2%/25.7%。主要性能损失来自"H"类全高 intensity 工作负载，因为 ATLAS 的 attained service 机制在同质化 intensity 下天然公平，TCM 则通过 misclassification 获得性能但牺牲公平性。

## 审稿人视角

### 优点

1. **问题定义清晰，动机充分**：论文精确地识别了 GPU 流量对 MC visibility 的影响机制——不仅是 buffer 被占满，更关键的是 GPU 同时具有高 intensity + 高 RBL + 高 BLP 的特征组合，使得所有基于 row-hit 优先或 intensity clustering 的策略都失效。Figure 1–3 的 motivation 分析非常有说服力。

2. **架构设计思想有原创性**：将 MC 功能 decouple 为三个 stage 的思路非常优雅，每个 stage 只需解决一个子问题，且使用简单的 FIFO 结构。这种 "separation of concerns" 的设计哲学不仅降低了硬件复杂度，还提供了清晰的 composability——各 stage 的策略可以独立替换和优化。

3. **可配置性是实用亮点**：SJF 概率 *p* 提供了一个简单而有效的 CPU/GPU 性能 tradeoff 旋钮，可由系统软件动态调整，适应不同应用场景。这在实际产品化中非常有价值。

4. **评估全面**：105 个工作负载、7 种 intensity mix、多种 core/channel 配置的 sensitivity study、CPU-only 场景验证、面积/功耗建模，覆盖面很广。CGWS 指标的引入也为异构系统评估提供了灵活框架。

5. **硬件实现可行性强**：有 Verilog 综合结果支撑，面积和功耗数据具有一定说服力（虽然是 180nm）。

### 不足

1. **GPU 模型的代表性存疑**：GPU trace 来自 AMD 内部的 proprietary simulator，不可复现。8 个 GPU benchmark 的多样性有限（主要是 3DMark 和游戏），缺少 GPGPU compute workload（如 Rodinia、Parboil）。GPU 内存请求在经过 GPU 内部 cache hierarchy 过滤后直接注入，这种 trace-driven 方法忽略了 GPU 调度行为与 memory latency 之间的反馈效应。

2. **In-order DCS 的局限性被低估**：论文声称 DCS 的 in-order 设计不会影响性能，因为 "batch scheduler 已经做了 application-aware 决策"。但在实际场景中，当多个 batch 被依次 drain 到同一 bank 的 FIFO 时，可能出现同一 bank 的 FIFO 中存在来自不同 row 的请求交替排列的情况（来自不同 batch），此时 in-order DCS 无法做 row-hit 优化。论文对此场景的分析不够深入。

3. **Batch formation 的 in-order 约束可能浪费 row-buffer locality**：论文承认 out-of-order batch formation 仅有 <5% 的性能差异，但未详细分析在 GPU 应用中（具有高度规律的空间局部性）这一约束的影响。对于某些 CPU 应用（如 stride access pattern），in-order batching 可能无法捕获非连续的 row-buffer hit 机会。

4. **SJF 策略的 "shortest" 定义粗糙**：SJF 以 total in-flight request count 作为 "job size" 的代理指标，但这忽略了不同应用请求的 bank distribution 差异。一个有 5 个请求分散在 5 个 bank 的应用，其实际占用带宽和干扰程度远低于 5 个请求集中在 1 个 bank 的应用。更精细的 intensity metric（如 per-bank request density）可能提供更好的调度决策。

5. **Bypass 机制的阈值选择缺乏理论依据**：低/中/高 intensity 的 MPKC 阈值（1, 10）和 threshold age（0, 50, 200, 800 cycles）完全基于经验调优，论文未提供选择这些值的方法论或分析其对不同工作负载的鲁棒性。

6. **公平性评估指标单一**：仅使用 max slowdown 作为 unfairness 度量。缺少 per-application slowdown 的分布分析（如 CDF），无法判断 SMS 是否在改善最差情况的同时恶化了中等情况的应用体验。

7. **与 source throttling 的对比缺失**：论文提到 Ebrahimi et al. [6] 的 source throttling 是互补方案，但未做定量比较。在 CPU-GPU 场景下，source-side 限制 GPU 注入速率可能是一种更简单的替代方案，论文应当讨论为何 MC-side 的 staging 优于 source-side throttling。

### 疑问或值得追问的点

- **Batch formation 的 per-source FIFO 深度（CPU 10, GPU 20）是否在所有场景下都够用？** 当 GPU 应用的 memory intensity 极高时（如 Bench03, MPKI=419.7），20-entry FIFO 是否会成为瓶颈？Back-pressure 机制如何工作？
- **SJF 概率 *p* 的动态调整策略**：论文提到可以通过 ISA 指令动态调整 *p*，但未给出具体的 adaptive 算法。在实际系统中，如何根据 workload phase 自动确定最优 *p*？
- **多 GPU 场景的扩展性**：论文仅考虑 1 个 GPU 的情况。在现代 chiplet 架构中，可能存在多个 GPU tile 或加速器共享内存，SMS 的 SJF 策略是否仍然有效？
- **Write request 的处理**：论文几乎未讨论 write scheduling。在 GPU workload 中 write 流量占比可观（如 frame buffer update），write-to-read turnaround (tWTR) 的处理对 row-buffer locality 的影响值得深入分析。
- **面积/功耗数据的参考价值**：180nm 标准单元综合且未做优化（flip-flop based buffer），与实际产品级 MC（通常使用 SRAM-based 结构和 advanced node）差距较大，这些数据仅能定性说明 SMS 的复杂度优势。
