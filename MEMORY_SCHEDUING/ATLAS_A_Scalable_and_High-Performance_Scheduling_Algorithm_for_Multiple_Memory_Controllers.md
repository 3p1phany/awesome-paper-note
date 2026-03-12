---
title: "ATLAS: A Scalable and High-Performance Scheduling Algorithm for Multiple Memory Controllers"
authors: "Yoongu Kim, Dongsu Han, Onur Mutlu, Mor Harchol-Balter"
venue: "HPCA 2010"
year: 2010
---

# ATLAS: A Scalable and High-Performance Scheduling Algorithm for Multiple Memory Controllers

## 基本信息

- **发表**：HPCA, 2010
- **作者**：Yoongu Kim, Dongsu Han, Onur Mutlu, Mor Harchol-Balter（Carnegie Mellon University）

## 一句话总结

> 基于排队论中 Least-Attained-Service 思想，以长 quantum 粒度追踪线程 memory intensity 并排序，实现高吞吐且多 MC 可扩展的内存调度。

## 问题与动机

**问题**：CMP 系统中多核共享多个 memory controller（MC），MC 的调度算法需要同时满足三个目标：(i) 最大化系统吞吐、(ii) 可配置性（支持系统软件指定线程优先级）、(iii) 可扩展性（不依赖 MC 间频繁细粒度的信息交换）。

**为什么重要**：随着核心数增长远超 off-chip pin bandwidth 的增长，主存带宽成为主要瓶颈。不合理的调度会导致系统吞吐下降，部分核心长期饥饿。

**现有工作不足**：

- **FCFS / FR-FCFS**：thread-unaware，不区分线程，inherently 偏向 memory-intensive 线程（因其请求更"老"），不保留 bank-level parallelism，吞吐低。FR-FCFS 虽然利用 row-buffer locality，但盲目优先 row-hit 会造成低 row-buffer locality 线程的 starvation。
- **FQM (Fair Queueing Memory)**：试图均分带宽，per-bank 独立决策导致 bank-level parallelism 退化，不考虑长期 memory intensity，无需 MC 间协调但吞吐低。
- **STFM (Stall-Time Fair Memory)**：估算线程 slowdown 并优先被减速最多的线程。但设计面向单 MC，在多 MC 下需要极细粒度的协调（每个 DRAM cycle 级别），且 slowdown 估算依赖 heuristic，多 MC 多线程下准确性退化严重。
- **PAR-BS (Parallelism-Aware Batch Scheduling)**：当时最佳 CMP memory scheduling 算法。但有三大缺陷：(1) batch 粒度太细（平均~2000 cycles），无法捕获完整 memory episode，可能将短 episode 打碎跨 batch 服务；(2) batch 结束时需要在所有 MC 间交换 per-thread 信息来形成一致的 thread ranking，通信量 O(N·M)，且 batch 很短导致协调极其频繁，不可扩展；(3) 以"线程在单 bank 的最大请求数"作为 shortest-job-first 的 heuristic，在多 MC 多线程下由于 load imbalance 而日益不准确。

**核心矛盾**：高吞吐需要 MC 间一致的 thread ranking（coordinated scheduling），但频繁协调不可扩展。

## 核心方法

### 关键思路

ATLAS 的核心洞察有两个：(1) Memory episode（线程等待至少一个内存请求的时间段）的长度服从 Pareto 分布（具有 Decreasing Hazard Rate, DHR 性质），因此已获服务（attained service）越少的线程，其 episode 预期越快结束——这正是排队论中 Least-Attained-Service (LAS) 策略最优的理论前提；(2) 通过将 LAS 的统计窗口从单个 episode 扩展到长 quantum（10M cycles），既能捕获线程的长期 memory intensity 行为，又能将 MC 间协调频率降低到极低水平，从而兼得高吞吐与可扩展性。

### 技术细节

#### 1. Memory Episode 与 Pareto 分布

线程执行交替于 memory episode（等待内存请求）和 compute episode（无内存请求，IPC 高）。论文对 SPEC CPU2006 的 29 个 benchmark 测量发现，其中 26 个的 memory episode length 服从 Pareto 分布（log-log 坐标下线性拟合，R² > 0.96）。Pareto 分布的 DHR 性质意味着：一个 episode 已经运行得越久，它预期还要运行得越久。因此，优先服务 attained service 最少的 episode 等价于优先服务"最快结束"的 episode，近似于理论最优的 SRPT（Shortest-Remaining-Processing-Time）。

#### 2. 从 Per-Episode LAS 到 Per-Thread LAS（长期 intensity 考量）

单纯的 per-episode LAS 只做短期优化，忽视了线程的长期行为。论文举例：memory-intensive 线程 A 有频繁短 memory episode 和极短 compute episode；non-intensive 线程 B 有偶尔的长 memory episode 和很长的 compute episode。Per-episode LAS 会优先 A，但 A 很快又回来竞争内存。正确策略是优先 B，让 B 尽快进入长 compute episode，减少竞争。

ATLAS 将 LAS 扩展到长 quantum（固定长度 time interval，默认 10M cycles）。每个 quantum 内，MC 追踪每个线程的 attained service（AS）。Quantum 结束时用 exponentially-weighted moving average 更新 TotalAS：

$$TotalAS_i = \alpha \cdot TotalAS_{i-1} + (1-\alpha) \cdot AS_i$$

其中 α=0.875（偏向保留历史，但也保持对 phase 变化的适应性）。下一个 quantum 中，TotalAS 越低的线程排名越高（获得更高调度优先级）。

#### 3. ATLAS 调度规则（Rule 1: 每个 MC 的请求优先级）

按优先级从高到低：
1. **TH (Over-threshold-requests-first)**：在 MC 中等待超过 T cycles（默认 100K）的请求最优先，防止 starvation。
2. **LAS (Higher-LAS-rank-thread-first)**：LAS rank 高的线程请求优先，最大化吞吐并保留 per-thread bank-level parallelism。
3. **RH (Row-hit-first)**：Row-hit 请求优先于 row-conflict/closed 请求，利用 row-buffer locality。
4. **FCFS (Oldest-first)**：更老的请求优先。

#### 4. MC 间协调（Rule 2: Quantum 结束时的动作）

1. 每个 MC 将各线程的 local AS 发送到 centralized meta-controller，然后 reset local AS。
2. Meta-controller 汇总各 MC 的 local AS，按公式更新 TotalAS。
3. 基于 TotalAS 排名（TotalAS 越低排名越高）。
4. 广播 ranking 到所有 MC。
5. 各 MC 收到后用新 ranking 进行下一 quantum 的调度。

**关键设计权衡**：Quantum 越长，协调越不频繁（可扩展性好），且更能捕获长 memory episode 和长期 intensity 行为；但太长则无法适应 workload phase 变化。论文实验表明 10M cycles 是较好的平衡点。

#### 5. 系统软件支持

通过 thread weight 机制，系统软件可指定线程优先级。更新 TotalAS 时用 weight 缩放：

$$TotalAS_i = \alpha \cdot TotalAS_{i-1} + \frac{(1-\alpha)}{thread\_weight} \cdot AS_i$$

权重越高的线程"看起来" attained service 越少，因此获得更高优先级。实验显示 ATLAS 能提供 weight 与 speedup 之间的线性关系，优于 PAR-BS 的指数关系。

#### 6. 硬件开销

在 24 核、4 MC、128-entry request buffer、4 banks/MC、quantum 10M cycles 的配置下，ATLAS 相比 FR-FCFS 额外需要 8,896 bits 的存储。关键寄存器包括：per-request 的 over-threshold flag (1 bit)、thread-rank (5 bits)、thread-ID (5 bits)；per-controller 的 QuantumDuration (24 bits)；per-thread-per-controller 的 Local-AS (26 bits)；meta-controller 中 per-thread 的 TotalAS (28 bits)。所有逻辑不在执行关键路径上。

### 与现有工作的区别

| 特性 | ATLAS | PAR-BS | STFM |
|------|-------|--------|------|
| 线程区分依据 | 长期 memory intensity (LAS-based TotalAS) | 短期 request count (per-batch shortest-job-first heuristic) | Slowdown estimation (interference-based) |
| 协调粒度 | ~10M cycles (quantum) | ~2K cycles (batch) | 每个 DRAM cycle |
| 协调信息量 | O(N) per quantum（每线程一个 AS 值） | O(N·M) per batch（每线程在每 MC 的 max-requests + total-requests） | O(N) per DRAM cycle（bank-level parallelism 信息） |
| 理论基础 | 排队论 LAS 在 DHR 分布下最优 | Shortest-job-first heuristic | Fairness-based slowdown equalization |
| 可扩展性 | 高（长 quantum，低频协调） | 低（短 batch，高频协调） | 极低（per-cycle 协调） |

## 实验评估

### 实验设置

- **仿真平台**：In-house cycle-precise x86 CMP simulator，functional front-end 基于 Pin 和 iDNA，详细建模内存系统的 bandwidth limitation、contention、bank/port/channel/bus conflicts。
- **处理器配置**：5 GHz, 128-entry instruction window (64-entry issue queue + 64-entry store queue), 12-stage pipeline, 3-wide fetch/exec/commit (最多 1 条 memory op)；32KB L1 (4-way, 2-cycle), 512KB L2 per core (8-way, 12-cycle, 32 MSHRs)。
- **DRAM 配置**：Micron DDR2-800 timing (tCL=tRCD=tRP=15ns, BL/2=10ns), 4 banks/controller, 2KB row-buffer/bank, single-rank DIMM, 64-bit wide channel (6.4 GB/s peak per controller)。FR-FCFS 作为 baseline controller，128-entry request buffer, 64-entry write buffer, reads over writes, XOR-based address mapping。
- **核心/控制器规模**：主评估 24 核 + 4 MC，扩展评估 4-32 核 + 1-16 MC。
- **Workload**：SPEC CPU2006，26 个 benchmark（排除了 bwaves, gamess, zeusmp），分为 memory-intensive（>10 L2 MPKI）和 non-intensive（<10 MPKI）两类。主评估使用 32 个 workload，每个含 12 个 intensive + 12 个 non-intensive benchmark。每次模拟 200M cycles。
- **对比 baseline**：FCFS, FR-FCFS, STFM, PAR-BS, PAR-BSc（理想化协调版 PAR-BS，假设各 MC 瞬间获得全局信息，不可实现的上界）。
- **性能指标**：Instruction Throughput（所有核心 IPC 总和）和 System Throughput（Weighted Speedup = Σ IPC_shared / IPC_alone）。
- **ATLAS 参数**：quantum = 10M cycles, α = 0.875, T (starvation threshold) = 100K cycles。

### 关键结果

1. **24 核 4 MC 系统**：ATLAS 平均比 PAR-BS 提升 instruction throughput 10.8%、system throughput 8.4%；比不可实现的 PAR-BSc 仍高 7.3%/5.3%；比 FR-FCFS 高 17.1%/13.2%。32 个 workload 上 ATLAS 对 PAR-BS 的 system throughput 提升区间为 3.4%~14.5%，改进一致性好。

2. **可扩展性**：固定 4 MC，随核心数从 4 增至 32，ATLAS 对 PAR-BS 的 system throughput 提升从 1.1% 增长到 10.8%，表明核心越多优势越大。单 MC 系统上 ATLAS 比 STFM（单 MC 最佳 prior work）高 22.4%/17.0%。

3. **协调效果**：ATLAS 的 uncoordinated 版本（各 MC 独立基于 local AS ranking）仍比 uncoordinated PAR-BS 高 7.5%、比 ideally-coordinated PAR-BSc 高 4.5%。协调仅额外带来 1.2%/0.8% 的提升，说明长 quantum 本身就很好地减少了协调需求。

4. **Fairness 代价**：ATLAS 的 maximum slowdown 增加 20.3%，harmonic speedup 降低 10.5%（相比 PAR-BS）。ATLAS 对 memory-intensive 线程不公平，但这些线程对额外延迟更不敏感，且可通过 thread weight 机制补偿。

### 结果分析

**Quantum Length 敏感性**：从 1K 到 25M cycles 变化，system throughput 随 quantum 增大而上升（因为更能捕获长期行为和完整 memory episode），直到 25M cycles 开始略微下降（适应性不足）。很小的 quantum（1K-10K）性能急剧下降，因为 ranking 变化过频繁，破坏 bank-level parallelism 并丧失长期信息。

**History Weight α 敏感性**：α 越大（保留更多历史），系统性能越高（因为 SPEC CPU2006 的 per-thread memory intensity 在长时间内相对稳定）。但过高的 α 会使调度易受恶意程序攻击（冒充低 intensity），因此选择 α=0.875 做平衡。

**Starvation Threshold T 敏感性**：T 过小导致退化为 FCFS（大量请求触发 threshold），T 过大则 starvation-freedom 保证变弱但性能不变（因为 LAS ranking 本身就有减轻 starvation 的效果）。

**Workload 异质性**：ATLAS 在异质 workload（混合 intensive 和 non-intensive）时提升最大（12/24 或 18/24 为 intensive 时，提升 8.4%~7.7%）；同质 workload 时提升较小但仍正向（全 non-intensive 时 3.0%，全 intensive 时 4.4%）。

**Address Mapping**：Block-interleaving 下 ATLAS 提升更大（8.4% vs. PAR-BS），Row-interleaving 下也有 3.5% 提升。ATLAS 更善于利用 block-interleaving 提供的高 bank-level parallelism。

**Cache Size 和 Memory Latency 敏感性**：L2 从 512KB 到 4MB，ATLAS 对 PAR-BS 的提升从 8.4% 降到 6.4%（更大 cache 减少内存压力）。tCL 从 5ns 到 30ns，提升从 4.4% 增到 10.2%（内存延迟越高，调度越重要）。

## 审稿人视角

### 优点

1. **理论基础扎实**：将排队论中 LAS 在 DHR 分布下最优的理论结果迁移到 memory scheduling 场景，并通过对 26/29 SPEC CPU2006 benchmark 的 memory episode length 分布分析验证了 Pareto/DHR 前提的成立。这使得设计不仅仅是 heuristic，而有了坚实的理论支撑。

2. **可扩展性设计优雅**：长 quantum 的设计一石二鸟——既能捕获长期线程行为（比 PAR-BS 的短 batch 更合理），又将 MC 间协调频率降低了 3~4 个数量级。对协调开销的实验分析（100~10000 cycles）表明性能几乎不受影响，进一步证实了设计的鲁棒性。

3. **实验评估全面且严格**：覆盖 4-32 核、1-16 MC、32 种 workload mix、多种 memory intensity 比例、多种地址映射方式、cache size 和 memory latency 敏感性。对比了 5 种 prior work（含不可实现的理想上界 PAR-BSc）。参数敏感性分析（quantum length, α, T）也很充分。

4. **硬件开销低**：额外仅需 ~9K bits 存储，无关键路径逻辑，实现友好。

5. **System software 支持设计巧妙**：通过 thread weight 缩放 attained service，实现了 weight 与 speedup 的线性关系，比 PAR-BS 的指数关系更易于系统软件推理和控制。

### 不足

1. **Fairness 牺牲显著但讨论不够充分**：Maximum slowdown 增加 20.3%，harmonic speedup 降低 10.5%。论文简单地将其归因于"memory-intensive 线程对额外延迟不敏感"并指向 thread weight 机制，但并未量化展示 thread weight 能在多大程度上恢复 fairness，也未讨论在实际系统中系统软件如何合理设置 weight（这本身就是一个非平凡的问题）。对于一个声称可配置的调度器，fairness 的定量恢复能力应作为核心评估内容。

2. **Pareto 分布假设的普适性存疑**：26/29 SPEC CPU2006 benchmark 符合 Pareto，但 SPEC CPU2006 以计算密集型为主，对 data center workload、graph processing、ML 推理等新兴负载的 memory episode 分布特征未做讨论。如果 episode length 分布不再 DHR（例如某些规律性极强的 streaming workload），LAS 的理论最优性将不成立。论文缺乏对 non-Pareto workload 的 robustness 分析。

3. **单一 quantum length 对所有线程/phase 不够灵活**：10M cycles 的固定 quantum 对 phase 变化频繁的 workload 适应性差。论文承认"very small quantum leads to low performance"，但另一面——quantum 太长导致对 phase transition 响应迟钝——的定量分析不够。尤其在 multi-programmed 场景下，不同线程的 phase 变化频率可能差异很大，统一的 quantum 长度是一个粗糙的设计选择。

4. **Meta-controller 的实现细节缺失**：论文提到 centralized meta-controller 收集、计算、广播，但对其物理位置、与 on-chip network 的集成、在 NUMA 架构下的延迟影响等缺乏讨论。虽然协调频率低，但 meta-controller 本身是一个 single point of failure / bottleneck，尤其在 16+ MC 的场景下。

5. **Workload 评估的时代局限性**：仅使用 SPEC CPU2006 multiprogrammed workload，缺少 multi-threaded workload（shared-memory 并行程序）的评估。Multi-threaded 场景下线程间存在同步和数据共享，memory episode 的行为模式可能与独立 benchmark 的 multiprogrammed 混合非常不同。

6. **DDR2 时代的参数配置**：基于 DDR2-800、4 banks/controller，与现代 DDR5（16/32 banks per rank, bank group 结构）差异巨大。Bank group timing constraints 会改变 row-buffer management 和 bank-level parallelism 的 trade-off，ATLAS 的 LAS-based ranking 在现代 DRAM 架构下的效果需要重新验证。

### 疑问或值得追问的点

- 在 memory episode length 分布不满足 DHR 的场景（如 3 个不符合 Pareto 的 SPEC benchmark），ATLAS 的性能退化幅度如何？论文未做隔离分析。
- α = 0.875 的选择看似通过 sweep 得到，但 per-workload 的最优 α 是否差异很大？自适应 α 是否能带来进一步提升？
- Thresholding 机制（T=100K cycles）作为 starvation freedom 的保底，与 LAS ranking 本身的 anti-starvation 效果之间的交互如何？是否存在 threshold 触发反而打破了 LAS 最优性的情况？
- 在多 MC 系统中，thread 的 request 分布在不同 MC 间可能高度不均匀（特别是使用 page-interleaving 时）。Uncoordinated ATLAS 在这种场景下是否会因为各 MC 看到的 local AS 差异过大而做出严重不一致的决策？
- 论文声称"ATLAS's performance benefit increases as the number of cores increases"，但实验最多到 32 核。在 64/128 核场景下，quantum 内 TotalAS 排名的区分度是否足够？当线程数远超 bank 数时，ranking 的有效性是否会退化？
