---
title: "Fair Queuing Memory Systems"
authors: "Kyle J. Nesbit, Nidhi Aggarwal, James Laudon, James E. Smith"
venue: "MICRO 2006"
year: 2006
---

# Fair Queuing Memory Systems

## 基本信息

- **发表**：The 39th Annual IEEE/ACM International Symposium on Microarchitecture (MICRO'06)，2006
- **单位**：University of Wisconsin – Madison; Sun Microsystems, Inc.

## 一句话总结

> 将网络 fair queuing 理论引入 DRAM memory scheduler，通过 virtual time memory system 建模为每个线程提供带宽 QoS 保障并公平分配剩余带宽。

## 问题与动机

CMP 时代多线程共享 off-chip DRAM memory system，若无管控，aggressive thread 会严重干扰其他线程的性能。论文用一个非常直观的例子说明了问题的严重性：SPEC benchmark vpr 单独运行时 memory latency 为 150 cycles，但与 memory-intensive 的 art 共同调度后，latency 飙升至 1070 cycles，IPC 下降 60%。这说明即便两个线程仅共享 DRAM（各自拥有 private cache），一个 aggressive thread 就能"饿死"其他线程。

现有的 FR-FCFS (First-Ready First-Come-First-Serve) 调度策略在单线程场景下能很好地优化带宽利用率，但在多线程场景下存在两个根本性缺陷：

1. **FCFS 对 bursty thread 不公平**：拥有长 burst cache miss 的线程会占据大量连续 FCFS slot，大幅增加其他线程请求的延迟。
2. **First-Ready 调度引发严重的 priority inversion**：由于 priority chaining，一连串低优先级的 row buffer hit 会持续阻塞高优先级请求（因为它们先 ready），导致高优先级请求迟迟无法被服务。

此外，当时针对多线程 memory scheduling 的研究很少。已有的 QoS memory scheduler 主要面向嵌入式/SoC 场景，依赖设计时已知的 memory access pattern，不适用于通用高性能计算。

## 核心方法

### 关键思路

核心 insight 是将网络领域成熟的 fair queuing (FQ) 理论迁移到 DRAM memory scheduling 中。每个线程被建模为运行在一个**私有的 virtual time memory system (VTMS)** 上——该 VTMS 的所有 SDRAM timing 参数按线程分配到的带宽份额 $\phi_i$ 的倒数进行 time scaling。例如，分配到 $1/2$ 带宽的线程，其 VTMS 中 $t_{CL}$ 翻倍。调度器按 **earliest virtual finish-time first (VFTF)** 优先级服务请求，等价于 earliest deadline first (EDF) scheduling，从而在满足 QoS 的同时公平分配剩余带宽。

### 技术细节

#### 1. Virtual Time Memory System (VTMS) 建模

VTMS 将 SDRAM 的 address bus 和 data bus 抽象为单一资源——memory channel。每个 memory request $m_i^k$（第 $i$ 个线程的第 $k$ 个请求）经过两阶段虚拟服务：

**Bank service**（请求访问 bank $j$）：

$B_j.S_i^k = \max\{a_i^k,\ B_j.F_i^{(k-1)'}\}$

$B_j.F_i^k = B_j.S_i^k + B_j.L_i^k / \phi_i$

**Channel service**：

$C.S_i^k = \max\{B_j.F_i^k,\ C.F_i^{k-1}\}$

$C.F_i^k = C.S_i^k + C.L_i^k / \phi_i$

其中 $C.F_i^k$ 即为该请求的 **virtual finish-time**，用于调度优先级排序。$a_i^k$ 是请求到达 memory controller 的虚拟到达时间，$\phi_i$ 是线程 $i$ 的 service share。

Bank service time $B.L_i^k$ 取决于 bank state：

| Bank State            | $B.L_i^k$                   |
|:--------------------- |:--------------------------- |
| Open - bank conflict  | $t_{RP} + t_{RCD} + t_{CL}$ |
| Closed                | $t_{RCD} + t_{CL}$          |
| Open - row buffer hit | $t_{CL}$                    |

Channel service time 统一为 $C.L_i^k = BL/2$。

将上述公式合并，virtual finish-time 的完整表达式为：

$C.F_i^k = \max\{\max\{Ra_i,\ B_j.R_i\} + B.L_i^k / \phi_i,\ C.R_i\} + C.L_i^k / \phi_i$

其中 $Ra_i$ 是线程 $i$ 最早的 pending request 的到达时间，$B_j.R_i$ 和 $C.R_i$ 分别是 bank $j$ 和 channel 的上一次 virtual finish-time register。

#### 2. 关键设计选择：real clock vs. virtual clock

与网络 FQ 使用 virtual clock 不同，FQ memory scheduler 使用 **real clock**（每周期递增，refresh 期间暂停）。原因在于 memory system 是 **closed system**（应用的 memory request rate 依赖于 memory latency），而网络是 open system（bit rate 独立于网络延迟）。使用 real clock 会惩罚过去消耗了更多带宽的线程，这正是论文所期望的 fairness policy：**将剩余带宽优先分配给历史上消耗最少 excess bandwidth 的线程**。

#### 3. Bank service 不确定性的处理

请求到达 memory controller 时，其精确 bank service requirement 未知（取决于后续调度决定的 bank state）。论文考虑了两种方案：

- **方案 A**：到达时假设平均 bank service time → 会惩罚 row hit 率高的线程，不公平
- **方案 B（采用）**：在请求即将被调度服务时才计算 virtual finish-time，并在 SDRAM command 发射后更新 VTMS register → 更精确但可能导致 out-of-order virtual finish-time assignment

论文选择方案 B，认为尽管存在 reordering，virtual finish-time 仍然尊重线程 VTMS 的带宽约束。

#### 4. 防止 Priority Inversion 的 FQ Bank Scheduling

First-Ready 调度的 priority chaining 问题：低优先级的 row buffer hit stream 持续 ready，阻塞高优先级请求。论文引入 **configurable bound** $x$ 机制：

- Bank closed 状态下，activate 后的前 $x$ 个 cycle 内，bank scheduler 使用标准 FR 策略（允许低优先级 row hit 完成服务，不浪费带宽）
- 超过 $x$ cycle 后，bank scheduler 切换为选择 earliest virtual finish-time 的请求，并等待其第一条 SDRAM command ready

论文选择 $x = t_{RAS}$，这是一个较紧的 bound，偏向 QoS 而可能牺牲少量 data bus utilization。这是一个核心 trade-off：**降低 priority inversion blocking time ↔ 降低 data bus utilization**。

#### 5. 硬件实现

FQ scheduler 在 FR-FCFS 基础上增加 VTMS hardware：

- 每个线程维护一组 VTMS register：每个 bank 一个 finish-time register $B_j.R_i$，一个 channel finish-time register $C.R_i$，一个 service share register $\phi_i$，一个 oldest arrival time register $Ra_i$
- Service share 初始化后，virtual service time（如 $B_j.L_i^k / \phi_i$）成为常数，无需每次重算
- Update logic 和 finish-time logic 仅需几个 adder 和 mux
- 所有线程可共享一份 update logic（因为每个 cycle 最多更新一个线程的 register）

VTMS register 的更新在 SDRAM command issue 时触发：

Bank register 更新：

$B_j.R_i = \max\{a_i^k,\ B_j.R_i\} + B_{cmd}.L_i^k / \phi_i$

Channel register 更新（仅在 read/write 时）：

$C.R_i = \max\{B_j.R_i,\ C.R_i\} + C_{cmd}.L_i^k / \phi_i$

各 SDRAM command 的 service time 定义如下：

| SDRAM Command | Bank service $B_{cmd}.L_i^k$            | Channel service $C_{cmd}.L_i^k$ |
|:------------- |:--------------------------------------- |:------------------------------- |
| Precharge     | $t_{RP} + (t_{RAS} - t_{RCD} - t_{CL})$ | n/a                             |
| Activate      | $t_{RCD}$                               | n/a                             |
| Read          | $t_{CL}$                                | $BL/2$                          |
| Write         | $t_{WL}$                                | $BL/2$                          |

#### 6. QoS 目标的定义

FQ memory scheduler 的 QoS objective：分配到 $\phi_i$ 份额的线程 $i$，其性能不低于在一个频率为 $\phi_i$ 倍物理内存频率的私有内存系统上单独运行的性能。论文承认这不是严格 guarantee（因为请求交织可能改变 row buffer 状态），但对通用高性能计算而言已足够。

### 与现有工作的区别

1. **FR-FCFS [Rixner et al., ISCA'00; Hur & Lin, MICRO'04]**：单线程优化的调度策略，不考虑 QoS，在多线程下 FCFS + First-Ready 导致严重不公平和 priority inversion。FQ scheduler 以 FR-FCFS 为基础，将 arrival time priority 替换为 virtual finish-time priority，并增加 bank-level priority inversion 控制。
2. **Zhu & Zhang [HPCA'05]**：针对 SMT 处理器的 memory scheduling，主要基于 ROB/issue queue/MSHR 占用率调度，不提供 QoS guarantee，也未建立理论框架。
3. **嵌入式/SoC QoS scheduler [Hofstee et al.; Lee et al.; Sonics MemMax]**：提供 hard real-time guarantee 但牺牲灵活性和带宽利用率，依赖设计时已知的 access pattern，不适用于通用高性能场景。

## 实验评估

### 实验设置

- **仿真平台**：IBM Research 开发的 detailed structural simulator（类似 ASIM），抽象层次略高于 latch-level model。默认配置为 IBM 970 单处理器系统，经过验证与 970 设计组 latch-level model 误差在 ±5% 以内。论文使用了修改后的配置（monolithic ROB、unified issue queue、64B cache line）以避免 970 特有约束
- **处理器配置**：4 GHz, 8-wide issue, 128-entry ROB, 32KB private I/D-Cache, 512KB private L2 Cache
- **内存系统**：1 channel, 1 rank, 8 banks DDR2-800; 每线程 16 transaction buffer entries 和 8 write buffer entries, closed page policy
- **Workload**：SPEC CPU 2000 benchmark suite, 20 个 benchmark 的 100M instruction sampled traces
- **对比 baseline**：
  - **FR-FCFS**：标准 first-ready first-come-first-serve
  - **FR-VFTF**：使用 VFTF priority 但不含 FQ bank scheduling（无 priority inversion 控制）
  - **FQ-VFTF**：完整的 FQ memory scheduler

### 关键结果

1. **双核 QoS 隔离实验**（subject thread + art 作为 aggressive background thread）：
   
   - FR-FCFS 下 subject thread 的 normalized IPC 的 harmonic mean 仅 0.62
   - FQ-VFTF 在 19 个 workload 中有 **18 个满足 QoS objective**（normalized IPC ≥ 1.0），唯一未满足的 vpr 也达到 0.94（FR-FCFS 下仅 0.48）
   - FQ-VFTF 相比 FR-FCFS 平均性能提升 **31%**（最高 76%），同时维持 **92%** 的平均 data bus utilization

2. **FR-VFTF vs FQ-VFTF 对比**（隔离 priority inversion 控制的贡献）：
   
   - FR-VFTF 平均 normalized IPC 为 0.87，14/19 个 workload 未满足 QoS
   - FQ-VFTF 平均 normalized IPC 为 1.10，**提升 26%**，证明 FQ bank scheduling 对控制 priority chaining 至关重要

3. **四核异构 workload 实验**：
   
   - FQ-VFTF 在所有 workload 的所有线程上均满足 QoS objective
   - 平均性能提升 **14%**（最高 41%）
   - Normalized data bus utilization 的 variance 从 FR-FCFS 的 **0.2 降至 0.0058**（范围从 0.28~2.1 收窄到 0.73~0.98）

4. **Data bus utilization trade-off**：
   
   - FQ-VFTF 在部分 benchmark（如 swim, mgrid）上 data bus utilization 下降约 10%，原因是 bank conflict 增加
   - Bank utilization 的增加是提供 QoS 的不可避免的副作用

### 结果分析

论文通过 Figure 9 的 normalized latency vs. normalized data bus utilization 散点图提供了深入分析：FR-FCFS 下各线程的带宽分配毫无规律（variance = 0.2），而 FQ-VFTF 下所有线程紧密聚集在目标值附近（variance = 0.0058）。论文还观察到 FQ scheduler 下 aggressive thread 仍然略微获得更多带宽，因为它们拥有更多 memory level parallelism，在被限制带宽时仍能产生更多请求。这一观察反过来支持了论文的 fairness policy 设计（优先将 excess bandwidth 给历史上消耗更少的线程）。

Sensitivity analysis 方面，论文通过三种配置（FR-FCFS / FR-VFTF / FQ-VFTF）的逐步对比，清晰分离了 VFTF priority 和 FQ bank scheduling 各自的贡献。但缺少对 bound $x$ 取值、不同 $\phi_i$ 分配策略、不同内存配置（multi-channel/multi-rank）的 sensitivity study。

## 审稿人视角

### 优点

1. **理论基础扎实**：将网络 fair queuing 理论完整迁移到 DRAM scheduling，建立了 VTMS 抽象模型，QoS objective 定义清晰（等价于 $\phi_i$ 频率的私有内存系统），且与 EDF scheduling 理论的对应关系明确。这种有理论支撑的系统设计在当时的 memory scheduling 领域是开创性的。

2. **问题分解清晰**：通过 FR-FCFS → FR-VFTF → FQ-VFTF 的三级对比，精确分离了 priority algorithm 和 bank scheduling algorithm 各自的贡献，实验设计体现了良好的方法论素养。

3. **对 closed system vs. open system 差异的洞察深刻**：认识到 memory system 与 network 的根本区别（request rate 依赖 latency），并据此设计了不同于 GPS 的 fairness policy 和使用 real clock 而非 virtual clock 的决策，这是非 trivial 的 adaptation。

4. **硬件开销合理**：VTMS register 和 update logic 的设计紧凑，service share 初始化后 virtual service time 成为常数，update logic 可被所有线程共享，实现复杂度低。

5. **QoS 效果显著**：在极端场景（art 作为 background）下将 subject thread 的 harmonic mean normalized IPC 从 0.62 提升到 1.10，同时维持 92% 的高 data bus utilization，说明 QoS 和效率并非不可兼得。

### 不足

1. **实验配置过于简化**：仅 1 channel / 1 rank / 8 banks 的 DDR2-800 系统，论文明确将 multi-channel 留作 future work。然而在实际 CMP 系统中 multi-channel 和 multi-rank 是常态，VTMS 的 channel 抽象是否能自然扩展到这些场景存在疑问——rank switching overhead（$t_{RRD}$）和 inter-channel scheduling 都会引入额外的复杂性。

2. **Workload 选择局限**：仅使用 SPEC 2000 单线程 benchmark 组合为多线程 workload，缺少真正的多线程共享数据应用（如 PARSEC、SPLASH-2）。共享数据应用的 memory access pattern（如同步、false sharing）可能显著影响 FQ scheduler 的效果。

3. **Fairness policy 缺乏形式化验证**：论文承认 fairness policy 的 formal validation 是 future work，但这恰恰是理论贡献的核心部分。在什么条件下 FQ scheduler 能严格满足 QoS？违反 QoS 的 worst case bound 是什么？这些问题未回答。

4. **未考虑 write traffic 的影响**：论文的实验和分析主要关注 read latency，但在实际系统中 read-write switching（$t_{WTR}$ 延迟）和 write drain 策略对带宽分配有重要影响。Write buffer 的静态分区也可能造成带宽浪费。

5. **Priority inversion bound $x = t_{RAS}$ 的选择缺乏充分论证**：论文直接选择了 $t_{RAS}$ 作为 bound 值，但未探索不同 $x$ 值对 QoS 和 data bus utilization 的 trade-off 曲线。这是一个重要的设计参数，应做 sensitivity analysis。

6. **缺少与 static partitioning 等简单方案的对比**：最简单的 QoS 方案是静态 time-slice 或 bandwidth partitioning（如按 round-robin 给每个线程固定 slot），论文未与此类 strawman baseline 对比，难以判断 FQ 方案的复杂度溢价是否值得。

### 疑问或值得追问的点

- VTMS 将 address bus 和 data bus 合并为单一 channel 资源，在 multi-rank 系统中 rank-level timing constraint（$t_{FAW}$, $t_{RRD}$）如何融入 VTMS 模型？这种抽象在 DDR4/DDR5 的 bank group 结构下是否仍然适用？
- 论文使用 closed page policy，若使用 open page policy，row buffer hit 对 virtual finish-time 的影响会更大，FQ scheduler 的 QoS 保障是否会退化？
- $\phi_i$ 的静态等分在实际系统中过于理想化。若 OS 动态调整 $\phi_i$，VTMS register 的状态如何平滑过渡？频繁调整是否导致 transient unfairness？
- 在 QoS 已满足（所有线程 normalized IPC ≥ 1）的情况下，FQ scheduler 的 excess bandwidth 分配策略是否真的优于简单的 proportional sharing？论文的 fairness policy（给历史上消耗最少 excess bandwidth 的线程）在 long-running workload 中是否会产生振荡？
