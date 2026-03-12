---
title: "Parallelism-Aware Batch Scheduling: Enhancing both Performance and Fairness of Shared DRAM Systems"
authors: "Onur Mutlu, Thomas Moscibroda"
venue: "ISCA 2008"
year: 2008
---

# Parallelism-Aware Batch Scheduling: Enhancing both Performance and Fairness of Shared DRAM Systems

## 基本信息
- **作者**：Onur Mutlu, Thomas Moscibroda（Microsoft Research）
- **发表**：ISCA 2008

## 一句话总结
> 通过 request batching 保证公平性与 starvation freedom，结合 parallelism-aware 的批内调度策略保持线程的 bank-level parallelism，同时提升 CMP 系统的公平性和吞吐量。

## 问题与动机

在 CMP 系统中，多个 core 共享 DRAM memory system。线程间的干扰（inter-thread interference）不仅会造成 bank conflict、row-buffer conflict 和 bus conflict，更关键的是会**破坏单个线程的 intra-thread bank-level parallelism (BLP)**。本来可以在多个 bank 中并行服务的请求被序列化（serialized），导致线程的 stall time 大幅增加。

论文的核心 motivation 来自一个关键观察：现有的所有 DRAM scheduler（包括 FR-FCFS、NFQ、STFM）都不感知线程的 bank-level parallelism。即使 DRAM throughput 被最大化了，单个线程的 stall time 仍可能因为其并发请求被交错服务而显著增加。论文 Figure 2 给出了一个经典的概念性例子：两个 core 各有两个请求分别访问 Bank 0 和 Bank 1，conventional scheduler 可能将 T0-Req0 与 T1-Req0 并行、再将 T1-Req1 与 T0-Req1 并行，导致两个 core 各 stall 2 个 bank latency；而 parallelism-aware scheduler 先并行处理 T0 的两个请求、再并行处理 T1 的两个请求，使得 T0 只 stall 1 个 bank latency，平均 stall time 从 2 降至 1.5。

现有工作的不足：
- **FR-FCFS**：row-hit-first 策略偏向高 row-buffer locality 的 memory-intensive 线程，导致严重的不公平和 starvation。
- **NFQ**（Nesbit et al., MICRO 2006）：基于网络 fair queueing 思想按 bank 均衡带宽分配，但不考虑 row-buffer state 和 bank parallelism，且存在 idleness problem。
- **STFM**（Mutlu & Moscibroda, MICRO 2007）：通过估计线程的 memory slowdown 来公平化调度，但 slowdown 估计不总是准确（尤其对高 BLP 线程），且不感知 bank parallelism。实现上需要除法器等复杂逻辑。

## 核心方法

### 关键思路

PAR-BS 的设计基于两个正交的 idea：(1) **Request Batching (BS)** 将请求分批处理，保证每个线程在每个 batch 中都有请求被服务，从而实现 starvation freedom 和公平性；(2) **Parallelism-Aware Within-Batch Scheduling (PAR)** 在 batch 内部通过 rank-based thread prioritization 保持线程的 bank-level parallelism，从而降低平均 stall time、提升系统吞吐量。关键 insight 是：batching 提供了一个天然的调度粒度，在其中可以安全地进行 thread-unfair 但 high-performance 的优化而不会显著损害整体公平性。

### 技术细节

#### 1. Request Batching (BS)

Batching 的核心机制：
- 每个请求在 request buffer 中有一个 **marked bit**，标记是否属于当前 batch。
- **Batch 形成时机**：当 buffer 中没有 marked request 时（即当前 batch 全部服务完毕），形成新 batch。
- **Marking 规则**：对每个线程，在每个 bank 中最多 mark `Marking-Cap` 个最老的 outstanding request。这些被 mark 的请求构成新的 batch。
- **调度规则**：marked request 始终优先于 unmarked request。但如果某个 bank 没有 marked request，则可以服务该 bank 的 unmarked request（避免浪费带宽）。

`Marking-Cap` 是一个关键系统参数，控制 batch 的大小：
- 太小（如 1）：破坏 row-buffer locality（频繁跨 batch 边界导致 row conflict），也限制了 bank parallelism 的发掘空间。
- 太大（或不设上限）：memory-intensive 线程在 batch 中插入过多请求，non-intensive 线程的请求容易 miss 一个 batch 的形成，被迫等待整个 batch 完成。
- 论文实验表明 `Marking-Cap = 5` 在公平性和吞吐量之间取得最佳平衡。

#### 2. Parallelism-Aware Within-Batch Scheduling (PAR)

Batch 内的请求优先级排序（Rule 2），从高到低：
1. **Marked-requests-first (BS)**：marked 优先于 unmarked。
2. **Row-hit-first (RH)**：row-hit 优先于 row-conflict/closed。
3. **Higher-rank-first (RANK)**：高 rank 线程的请求优先于低 rank 线程。
4. **Oldest-first (FCFS)**：同等条件下老请求优先。

**Thread Ranking（Max-Total 方案）**：每次新 batch 形成时计算线程 rank：
- **Max rule**：计算每个线程在所有 bank 中 marked request 数量的最大值（max-bank-load）。max-bank-load 越小，rank 越高。
- **Total rule（tie-breaker）**：若 max-bank-load 相同，marked request 总数（total-load）越少的线程 rank 越高。

这本质上是一个 **shortest-job-first** 策略：rank 高的线程请求少且分散在不同 bank，可以快速并行完成。由于所有 bank 使用**统一的 rank 顺序**，同一线程的请求会被倾向于在所有 bank 中同时服务，从而保持该线程的 bank-level parallelism。

#### 3. Thread Priority 支持

PAR-BS 提供灵活的线程优先级机制：
- **Priority-Based Marking**：优先级为 X 的线程每 X 个 batch 才 mark 一次其请求。最高优先级线程（level 1）每个 batch 都参与。
- **Priority-Based Within-Batch Scheduling**：在 Rule 2 的 BS 和 RH 之间插入 `PRIORITY—Higher-priority-threads-first` 规则。
- **Purely Opportunistic Service**：最低优先级 L 的线程请求永远不被 mark，仅在系统空闲时被调度。

#### 4. 硬件开销

在 FR-FCFS 基础上增加的硬件状态（以 8-core、128-entry request buffer、8 banks 为例）：仅 **1412 bits** 额外存储。主要包括每个 request 的 marked bit、Thread-ID、每线程每 bank 的请求计数器（用于 Max rule）、每线程总请求计数器（用于 Total rule）、全局 marked request 计数器和 Marking-Cap 寄存器。所有额外逻辑（marking、ranking）仅在新 batch 形成时触发，不在 DRAM 调度的 critical path 上。相比 STFM 需要除法器等复杂逻辑，PAR-BS 仅需简单的比较和 priority encoder。

### 与现有工作的区别

| 特性 | FR-FCFS | NFQ | STFM | PAR-BS |
|------|---------|-----|------|--------|
| Starvation freedom | 否（高 locality 线程可长期占用 bank） | 有限（存在 idleness problem） | 有限（slowdown 估计不准时可能短期 starvation） | **是（batch 严格保证）** |
| Bank parallelism awareness | 否 | 否（按 bank 独立均衡，无跨 bank 协调） | 否（按 slowdown 估计优先化，不感知 BLP） | **是（rank-based 跨所有 bank 统一排序）** |
| 实现复杂度 | 低 | 中（需 virtual deadline 计算） | 高（需除法器估计 slowdown） | **低（基于简单计数器和比较）** |
| 公平性机制 | 无 | 基于 fair queueing bandwidth 分配 | 基于 stall-time slowdown 均衡 | **基于 batch 粒度的进度保证 + SJF** |

NFQ 的核心问题是将网络 fair queueing 直接应用于 DRAM，但 DRAM 有 row-buffer state 和 banked structure，这与无状态网络链路有本质区别。STFM 依赖对线程 alone-running stall time 的估计来计算 slowdown ratio，但对高 BLP 线程（如 mcf）的估计经常不准确，导致这些线程被过度惩罚。

## 实验评估

### 实验设置
- **仿真平台**：cycle-accurate x86 CMP simulator，functional front-end 基于 Pin 和 iDNA。详细建模 memory system 的 bandwidth limitation、contention、bank/port/channel/bus conflicts。
- **处理器配置**：4 GHz, 128-entry instruction window (64-entry issue queue + 64-entry store queue), 12-stage pipeline, 3-wide fetch/exec/commit, 每核 32KB L1 + 512KB L2。
- **DRAM 配置**：Micron DDR2-800 timing (tCL=15ns, tRCD=15ns, tRP=15ns, BL/2=10ns)，8 banks，2KB row-buffer per bank，128-entry request buffer，XOR-based address-to-bank mapping。
- **带宽扩展**：1/2/4 channel 分别对应 4/8/16 core（每 channel 6.4 GB/s peak）。
- **Workload**：SPEC CPU2006（25 个 benchmark）+ 2 个 Windows 桌面应用（Matlab, xml-parser）。每个 benchmark 运行 150M instructions，选自 representative execution phase。根据 memory intensity、row-buffer locality、BLP 分为 8 个 category。
- **Workload 组合**：4-core 100 种组合，8-core 16 种，16-core 12 种。
- **对比 baseline**：FR-FCFS, FCFS, NFQ (FQ-VFTF with priority inversion prevention), STFM (α=1.10, IntervalLength=2^24)。

### 关键结果

1. **4-core 平均（100 workload）**：PAR-BS 相比 STFM（previous best），unfairness 改善 **1.11X**（从 1.36 降至 1.22），weighted-speedup 提升 **4.4%**，hmean-speedup 提升 **8.3%**，AST/req 降低 **7.1%**（从 301 降至 281 cycles）。

2. **8-core / 16-core 系统**：PAR-BS 优势持续扩大。8-core 上 unfairness 改善 1.08X、weighted-speedup 提升 4.3%、hmean-speedup 提升 6.1%。16-core 上 unfairness 改善 1.11X、weighted-speedup 提升 3.2%、hmean-speedup 提升 5.1%。

3. **Worst-case request latency**：PAR-BS 的 worst-case latency 显著低于 NFQ 和 STFM。4-core 上 PAR-BS 为 13866 cycles vs. STFM 20305 / NFQ 19801。这是因为 batching 天然地 bound 了任何请求的最大等待时间，而 NFQ/STFM 可能为了公平性而将某些线程的请求延迟很长时间。

4. **高 BLP 线程的保护**：在 Case Study I 中，mcf（BLP=4.75）在 NFQ/STFM 下 AST/req 分别恶化到 193/174 cycles，而 PAR-BS 仅为 146 cycles。Case Study III（4 copies of lbm）中，PAR-BS 将 AST/req 从 222 (FR-FCFS/STFM) 和 322 (NFQ) 降至 199 cycles，即使在完全公平的场景下也提升了 8.6% 的吞吐量。

### 结果分析

**Marking-Cap sensitivity**：论文详细分析了 cap 从 1 到 20 以及无上限（no-c）的影响。Cap=1 时 row-buffer locality 严重破坏（强制交错不同线程的请求），Cap 过大时 non-intensive 线程被延迟。Cap=5 是 sweet spot。论文提到可通过 adaptive Marking-Cap 进一步改进。

**Batching 策略对比**：论文评估了 full batching（PAR-BS 采用）、static batching（固定时间间隔）、eslot batching（允许迟到请求加入 batch）。Full batching 最优：static batching 对时间间隔敏感且无 starvation 保证；eslot batching 对 memory-intensive 线程惩罚过大。

**Within-batch scheduling 策略对比**：Max-Total ranking 最优。去除 ranking（使用纯 FR-FCFS 或 FCFS within batch）导致 throughput 下降 4.7%-10.7%。Random/round-robin ranking 也显著劣于 SJF-based ranking。即便只保留 batching 不做 parallelism-aware scheduling（round-robin within batch），效果也接近 STFM——说明 batching 本身就是一个强大的 fairness framework。

**PAR-BS 效果不显著的场景**：当线程本身 BLP 很低时（如 4 copies of matlab），parallelism-aware 调度的优势不明显（因为没有可利用的 bank parallelism）。此时 PAR-BS 的优势主要来自 batching 的公平性。

## 审稿人视角

### 优点

1. **问题定义精准，insight 深刻**。论文抓住了 shared DRAM controller 中一个被忽视的关键问题——intra-thread bank-level parallelism 的破坏。Figure 2 的 motivation example 简洁而有力地说明了即使 DRAM throughput 被最大化，个体线程的 stall time 也可能因为 BLP 破坏而增加。这个 observation 在 2008 年是 novel 的，对后续的 memory scheduler 设计产生了深远影响。

2. **设计优雅，两个组件正交且互补**。Batching 负责公平性和 starvation freedom，PAR 负责 throughput optimization。两者解耦使得设计空间的探索和分析非常清晰。Batching 本身作为一个通用 fairness framework 可以与任何 within-batch 策略组合，这增强了工作的通用性。

3. **实验非常全面，分析深入**。100 种 4-core workload 组合、3 种 case study 覆盖不同场景（memory-intensive、non-intensive、uniform BLP）、详尽的 sensitivity analysis（Marking-Cap、batching 方式、within-batch 策略），以及 8-core/16-core 的 scalability 评估。对 NFQ 和 STFM 的失败场景分析也很透彻（NFQ 的 idleness problem、STFM 的 slowdown 估计不准问题）。

4. **硬件开销极低，实现简单**。仅需 1412 bits 额外存储，不需要复杂算术运算（vs. STFM 的除法器），scheduling logic 不在 critical path 上。这使得 PAR-BS 具有很强的实际可实现性。

### 不足

1. **Marking-Cap 是静态参数，缺乏自适应机制**。论文承认 adaptive Marking-Cap 可能更好但未实现。实际负载中，workload phase 变化可能需要动态调整 cap 值。固定 cap=5 在某些特定 workload 组合下可能不是最优的。

2. **评估的系统规模和 DRAM 配置较为简单**。仅评估了单 rank、单 DIMM 配置，且 channel 数随核数线性扩展（4 core = 1 channel, 8 core = 2 channel, 16 core = 4 channel）。这种理想化的带宽扩展在实际系统中不一定成立。缺乏对多 rank、多 DIMM 场景的评估，而这些场景中 rank switching overhead 和 timing constraint 会引入额外的复杂性。

3. **BLP 的定义和利用方式相对粗糙**。Max-Total ranking 用 max-bank-load 作为 "job length" 的 proxy，但这是一个静态的、batch 形成时的 snapshot，不能反映请求的动态到达模式。对于请求分布高度不均匀的线程（某些 bank 集中、某些 bank 稀疏），max-bank-load 可能不是最佳的 ranking metric。

4. **缺乏对 write request 的深入分析**。论文简单地将 read 优先于 write，但在 write-intensive workload 中，write drain 的行为和 read-write switching overhead 可能显著影响 PAR-BS 的效果。

5. **Batch boundary 的 locality 损失未被充分量化**。论文定性提到 batch 边界会破坏 row-buffer locality（marked row-conflict 优先于 unmarked row-hit），但未给出 row-buffer hit rate 的具体变化数据来量化这一损失与 parallelism gain 之间的 trade-off。

### 疑问或值得追问的点

- Marking-Cap 的最优值是否与 core 数量、DRAM 配置（bank 数量、channel 数量）相关？论文的 sensitivity analysis 仅在 4-core 上进行，8/16-core 是否应该使用不同的 cap？
- SJF-based ranking 是否会在长期运行中系统性地 starve memory-intensive 线程？虽然论文认为 batching 保证了公平性，但在极端情况下（大量 non-intensive 线程 vs. 少数 intensive 线程），SJF 可能持续将 intensive 线程排在低 rank。
- 在 multi-channel 配置中，PAR-BS 是 per-channel 独立运行还是跨 channel 协调？论文似乎采用了 lock-step channel 配置（相当于增加 bus width），但独立 channel 配置下的行为值得探讨。
- 论文使用 DDR2-800 参数，对于后续的 DDR3/DDR4/DDR5 中更多的 bank (bank group) 和更复杂的 timing constraint，PAR-BS 的 Max-Total ranking 是否仍然有效？Bank group 的引入使得 intra-bank-group 和 inter-bank-group 的 parallelism 有不同的 cost，这可能需要更细致的 ranking 策略。
