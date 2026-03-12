---
title: "Page Policy Control with Memory Partitioning for DRAM Performance and Power Efficiency"
authors: "Mingli Xie, Dong Tong, Yi Feng, Kan Huang, Xu Cheng"
venue: "ISLPED 2013 (Symposium on Low Power Electronics and Design)"
year: 2013
---

# Page Policy Control with Memory Partitioning for DRAM Performance and Power Efficiency

## 基本信息
- **发表**：ISLPED (Symposium on Low Power Electronics and Design), 2013
- **单位**：北京大学微处理器研究开发中心

## 一句话总结
> 结合 bank partitioning 的流隔离特性，按应用的 memory intensity 和 row-buffer locality 动态分配 page policy，兼顾 DRAM 性能与功耗。

## 问题与动机

在 CMP 共享内存系统中，DRAM 同时面临性能和功耗两方面的挑战。现有工作存在以下不足：

1. **Bank partitioning 忽视功耗**：Mi et al. [15]、Jeong et al. [16] 等提出的 bank partitioning 通过将 bank 分配给不同 core 来消除 inter-thread interference，有效恢复了 row-buffer locality，但完全没有考虑 DRAM power management。
2. **Page policy 研究脱离应用特征**：Zheng and Zhu [17] 指出 page policy 对 DRAM 功耗有显著影响且最优策略是 application-dependent 的，但他们的评估是静态的、不感知应用行为的。
3. **CMP 中应用特征被掩盖**：在共享内存的 CMP 系统中，不同线程的 memory access stream 相互交织，单个应用的 memory intensity 和 spatial locality 特征被混淆，难以针对性地优化 page policy。

关键观察：bank partitioning 恰好提供了流隔离的基础，使得每个应用的内存访问特征得以保留，为 per-application page policy 提供了前提条件。这一点是本文的核心 motivation。

## 核心方法

### 关键思路

本文的核心洞察是：bank partitioning 在消除 inter-thread interference 的同时，保留了各应用独立的 memory access 特征（memory intensity 和 spatial locality），这为动态 page policy 分配创造了条件。不同应用对 page policy 的偏好不同——memory-intensive + high locality 的应用适合 open-page 以利用 row-buffer hit，而 non-intensive 应用适合 close-page 以让 rank 尽早进入 low power mode 节省 background power。

### 技术细节

#### 整体架构

方案分为两部分，分别运行在 memory controller（硬件）和 OS（软件）层面：

- **AAPP (Application-Aware Page Policy)**：硬件侧，memory controller 负责 profiling、分类和 page policy 分配。
- **PABP (Power-Aware Bank Partitioning)**：软件侧，OS 通过 page coloring 将 memory-intensive 和 non-intensive 应用映射到不同 rank，分离 hot rank 和 cold rank。

整个方案以固定长度的 interval（100K memory cycles）为周期运行，当前 interval profiling，下一个 interval 应用新策略。

#### AAPP 的三步流程

**Step 1 — Profiling**：在每个 interval 内收集两个指标：
- **MAPI (Memory Accesses Per Interval)**：衡量 memory intensity，用 18-bit counter 统计。
- **RBH (Row-Buffer Hit rate)**：衡量 spatial locality，通过 per-bank shadow row-buffer index 记录上一次访问的 row address，统计 shadow row-buffer hit 次数（SRBH），计算 hit rate。

**Step 2 — Grouping**：使用两个阈值进行二级分类：
- 若 MAPI < MAPIt → non-intensive group
- 若 MAPI ≥ MAPIt 且 RBH < RBHt → memory-intensive, low RBL group
- 若 MAPI ≥ MAPIt 且 RBH ≥ RBHt → memory-intensive, high RBL group

默认参数：MAPIt = 200, RBHt = 0.5。

**Step 3 — Policy Assignment**：
| Memory Intensity | Row-buffer Hit Rate | Page Policy | 收益来源 |
|---|---|---|---|
| Low | — | Close-page | 节省 background power（bank 快速 precharge → rank 进入 low power mode） |
| High | High | Open-page | 利用 row-buffer locality 降低 latency |
| High | Low | Close-page | 减少 bank conflict 延迟 |

设计理由：non-intensive 应用的 DRAM 功耗中 background power 占 90% 以上 [17]，close-page 可以让 bank 在 column access 后立即 precharge，增大 rank idle 概率，从而进入 power-down mode。对于 memory-intensive + low locality 的应用，open-page 反而会带来大量 bank conflict（precharge → activate → column access），close-page 直接省去了等待中的 precharge 步骤。

#### PABP 的设计

PABP 的目标是将 memory-intensive 和 non-intensive 应用映射到不同 rank，使 cold rank 尽可能长时间保持 idle。

算法：假设 n 个应用、m 个 rank（k = n/m），按 MAPI 排序后，最 intensive 的 k 个应用映射到一个 rank（hottest rank），次 intensive 的 k 个映射到次热的 rank，依此类推。实现上通过 OS page coloring 来控制物理页到 bank set 的映射。

Page fault handler 的分配策略（优先级递减）：
1. 分配 mapped bank set 中的 free page
2. 分配同一 rank 中其他 bank set 的 free page（保证 hot/cold 分离）
3. 分配其他 rank 的 free page

论文观察到跨 bank set 分配的情况很少（< 10%），因此不做 page migration。

#### 硬件开销

对于 4-core、2-rank、32768 rows/bank 的系统，总硬件开销仅 388 bits：
- MAPI counter: 4 × 18 = 72 bits
- Shadow row-buffer index: 2 × 8 × log₂(32768) = 240 bits
- SRBH counter: 4 × 18 = 72 bits
- Page policy register: 4 bits

### 与现有工作的区别

1. **vs. Jeong et al. [16] (HPCA'12)**：[16] 结合 bank partitioning 与 sub-rank 来节省功耗，sub-rank 通过减少单次访问涉及的 device 数量来降功耗，但需要修改 DIMM 硬件且会降低 peak bandwidth。本文方案不修改 DIMM，正交于 sub-rank，两者可叠加使用（实验验证了 combined improvement 达 62.64% power efficiency）。

2. **vs. Muralidhara et al. [18] (MICRO'11, MCP)**：MCP 将可能互相干扰的应用映射到不同 memory channel，但无法完全消除 inter-thread interference（同 channel 内仍有干扰），且不考虑功耗。本文 AAPP 基于 bank partitioning 后的完全隔离，做更精细的 per-application page policy 调整，且可以和 MCP 组合使用。

3. **vs. Zheng and Zhu [17] (IEEE TC'10)**：[17] 首次揭示 page policy 与 low power mode 协同对功耗的影响，但评估是 static 的，不能 runtime 感知应用行为。本文将其洞察动态化，结合 bank partitioning 的流隔离优势实现 per-application adaptation。

## 实验评估

### 实验设置

- **仿真平台**：gem5 + DRAMSim2
- **处理器配置**：4 cores, 3 GHz, out-of-order, Alpha ISA
- **Cache 层次**：L1 32KB I-Cache + 32KB D-Cache (8-way, 64B line, LRU per core); L2 4MB shared (16-way, 64B line)
- **内存配置**：1 channel, 2 ranks/channel, 8 banks/rank, DDR3-1066 (tCL-tRCD-tRP = 8-8-8), 4GB capacity
- **Memory Scheduler**：FR-FCFS
- **参数设置**：interval = 100K memory cycles, MAPIt = 200, RBHt = 0.5
- **Workload**：SPEC CPU2006 benchmarks, gcc 4.3.2 -O3 交叉编译。按 MPKI 和 RBH 分为 memory-intensive / non-intensive × high RBH / low RBH 四类，组合出 12 个 4-program workload（4 memory-intensive, 4 mixed, 4 non-intensive）
- **Baseline / 对比方案**：Base (balanced bank partitioning + open-page), PABP alone, AAPP alone, PABP+AAPP, sub-rank [16], All+sub-rank
- **Metrics**：Weighted Speedup (WS), DRAM power consumption (Micron methodology [4]), Power Efficiency = WS / DRAM Power

### 关键结果

1. **Memory-intensive workloads 性能提升**：PABP+AAPP 平均提升 system throughput 6.46%，其中 AAPP 贡献 5.85%。WL4（全 low locality 组成）从 AAPP 获益最大，达 12.87%。

2. **Non-intensive workloads 功耗大幅降低**：PABP+AAPP 平均降低 DRAM power consumption 40.81%。AAPP 单独即可降低 35.80%，因为 close-page 让 bank 快速 precharge，rank 大量时间处于 power-down mode。

3. **Overall power efficiency 显著改善**：PABP+AAPP 在 memory-intensive / mixed / non-intensive 三组分别提升 power efficiency 3.44% / 33.40% / 69.63%，所有 workload 平均 35.49%。

4. **与 sub-rank 互补**：两者结合后，所有 workload 平均 power efficiency 提升达 62.64%，power saving 达 35.16%，throughput 提升 3.17%。

### 结果分析

**PABP 在 memory-intensive group 效果有限**：因为所有应用都是 memory-intensive，PABP 无法有效分离 hot/cold rank。但对 mixed group 效果显著（功耗降低 16.32%，最高 23.49%），因为能清晰地将 intensive 和 non-intensive 应用分到不同 rank。

**AAPP 对 memory-intensive + low locality 效果最好**：WL4 全部由 low spatial locality 的 memory-intensive benchmark 组成，AAPP 为它们分配 close-page，避免了大量 bank conflict，throughput 提升 12.87%。

**Non-intensive group 存在微小性能损失**：PABP+AAPP 降低 throughput 0.55%，原因是更频繁进入 power-down mode 引入了额外的 exit latency 和 activate latency。但功耗收益远大于性能损失。

**Sensitivity analysis**：
- **Interval length**：100K memory cycles 是最优平衡点。50K 过短导致 profiling 不稳定，400K 过长无法捕捉行为变化。
- **MAPIt 阈值**：随 MAPIt 增大，更多 intensive 应用被错分为 non-intensive，引入额外 activate/exit latency。MAPIt = 600 时 PE 开始下降，因为 close-page 引入的 operation power 超过节省的 background power。
- **RBHt 阈值**：PE 对 RBHt 不敏感且保持较高水平，但 throughput 在 RBHt = 0.5 时最优。
- **Core 数量扩展性**：4/8/16 core 下趋势一致，方案具有 scalability。

## 审稿人视角

### 优点

1. **问题建模清晰，insight 有价值**：本文准确抓住了 bank partitioning 提供流隔离后为 per-application page policy 创造条件这一关键联系，将 performance optimization 和 power management 自然地统一在一个框架内。这个 insight 虽然事后看来直觉上合理，但在当时确实是第一个将 bank partitioning 与动态 page policy 联系起来的工作。

2. **方案极低开销且实用性强**：硬件仅需 388 bits，软件侧复用已有的 page coloring 机制，不需要修改 DRAM DIMM（对比 sub-rank），整体 complexity 很低，具备实际部署的可行性。

3. **与 sub-rank 的正交性验证充分**：实验证明 AAPP/PABP 与 sub-rank 的收益是 additive 的，说明本文方案捕捉到了不同维度的优化机会（延长 rank idle time vs. 减少 access 涉及的 device 数）。

### 不足

1. **实验规模偏小，配置过于简单**：仅评估了 1 channel / 2 rank / DDR3-1066 的配置，4 个 core。现代系统通常是多 channel、多 rank，且 core 数远超 4。虽然 sensitivity study 涵盖了 8/16 core，但没有增加 channel 数量的实验。单 channel 下 rank switching overhead 的影响可能与多 channel 场景有较大差异。

2. **Bank partitioning 的前提假设过强**：方案严格依赖 bank partitioning（等分 bank 给 core），这在 core 数较多时会严重限制每个 application 可用的 bank 数（如 16 core / 2 rank / 8 bank per rank → 每个 core 仅 1 bank），bank-level parallelism 几乎丧失。论文对这个 scalability 问题的讨论不够深入。

3. **分类阈值的鲁棒性存疑**：MAPI 和 RBH 的阈值是固定的（MAPIt = 200, RBHt = 0.5），sensitivity study 显示 MAPIt 对性能影响较大（MAPIt = 600 时 WS 从 +3.0% 变为 -2.5%）。论文没有提出自适应阈值调整机制，对 workload 组成变化的鲁棒性有顾虑。

4. **Page policy 粒度是 per-application 而非 per-bank**：同一应用的不同 bank 可能展现不同的 row-buffer locality 特征（尤其是应用的访问 pattern 有 phase behavior 时），per-application 的粒度可能错过 finer-grained 的优化机会。

5. **缺少对 self-refresh mode 的讨论**：论文仅使用 power-down mode，将 self-refresh 留给 future work。对于 non-intensive workload 中 rank idle 时间可能非常长的情况，power-down 并非最优选择，self-refresh 可以带来更大的功耗节省。

6. **Workload 代表性和公平性评估不足**：仅使用 SPEC CPU2006 单线程 benchmark 的多程序组合，缺少多线程 workload（如 PARSEC）和真实服务器 workload 的评估。此外没有评估 fairness metric（如 maximum slowdown），仅报告了 weighted speedup。

### 疑问或值得追问的点

- PABP 的 page coloring 重新映射会触发大量 page fault 和 TLB flush，论文声称 overhead negligible 引用的是 [26] 的实验，但 [26] 的场景是否与 periodic remapping 一致？如果每个 interval（100K memory cycles ≈ 几十微秒量级）都可能触发重新映射，累积开销是否仍可忽略？
- Memory-intensive + low RBL 组分配 close-page 的理由是减少 bank conflict latency，但 close-page 也会增加 activate 次数（每次访问都需要 activate），这个 trade-off 是否在所有 low locality 场景下都成立？论文缺少对这一子类的单独深入分析。
- 论文的 power model 使用 Micron 的方法 [4] 但基于 DRAMSim2 的事件统计，DRAMSim2 对 power-down mode 的建模精度如何？进入/退出 power-down 的时机和 latency 建模是否准确？
