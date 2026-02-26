---
title: "Dynamic Page Policy Using Perceptron Learning"
authors: "Muhammad M. Rafique, Zhichun Zhu"
venue: "MEMSYS 2022"
year: 2022
---

# Dynamic Page Policy Using Perceptron Learning

## 基本信息

- **发表**：International Symposium on Memory Systems (MEMSYS), 2022
- **机构**：University of Illinois at Chicago

## 一句话总结

> 利用 perceptron learning 综合多维度 memory access 特征，动态预测 DRAM page 的 open/close 决策。

## 问题与动机

**核心问题**：在多核处理器并发执行 data-intensive 应用时，交织的内存请求导致 memory controller 看到的访问模式趋于随机化，静态的 open-page 或 close-page policy 无法适应动态变化的 locality 特征，导致 row-buffer conflict 增多、访存延迟升高。

**为什么重要**：Page policy 的选择直接决定了内存访问延迟的构成——open-page 模式下 page hit 只需 tCL，但 page conflict 需要 tRP + tRCD + tCL（worst case）；close-page 模式下每次访问都是 tRCD + tCL（page miss）。在 HBM 这类拥有数百个 bank 的大规模并行内存系统中，page policy 对性能的影响被进一步放大。

**现有工作不足**：

1. **静态 policy**（open/close）：无法适应应用执行过程中动态变化的 locality。
2. **Access-based 动态 policy**（如 PBRM [3]）：仅依赖有限的历史信息（如最近打开的 row 的 hit count），缺乏对长期访存行为的洞察。
3. **Time-based 动态 policy**（如 INTAH [6]）：用固定的超时机制关闭 page，无法精确捕捉访存模式的变化。
4. **FAPS-3D [9]**：仅基于 bank-level 的 page hit rate 做 epoch 粒度的决策，粒度较粗，无法利用 page-level 的细粒度信息。

## 核心方法

### 关键思路

将 page open/close 决策建模为一个**二分类问题**，利用 perceptron learning 从多个维度的 feature（page-level + bank-level）学习长期的内存访问行为模式，在线训练并动态调整预测。核心 insight 是：单一特征（如 hit rate）不足以准确预测 page policy，需要综合多维信息（utilization、hotness、recency、stride 等）并通过学习确定各特征的权重。

### 技术细节

#### 整体架构

系统由三个核心组件构成：

1. **Page Record Table (PRT)**：32-way set-associative 结构，16 个 set（按 bank ID 索引），存储 page-level 特征。
2. **Bank Record Table (BRT)**：direct-mapped 结构，每个 bank 一个 entry，存储 bank-level 特征。
3. **Weight Tables (WT)**：每个 feature 对应一个独立的 weight table，以 feature 的 counter value 为索引，direct-mapped 结构。每个 weight 为 4-bit（范围 -8 到 +7）。

#### Feature 设计（7 个特征）

**Page-level features（5 个，存储在 PRT）**：

- **Page Utilization**（4-bit）：page 打开后被访问的 column hit 次数，反映 spatial locality。
- **Page Hotness**（4-bit）：page 被访问的总次数，反映该 page 的热度。
- **Page Recency**（5-bit）：page 的近期访问情况。被访问时设为最大值，同 bank 内其他 page 的 recency counter 递减 1。
- **Column Stride**（4-bit）：连续 column 访问之间的步长，大步长意味着下一次访问不太可能是 page hit。
- **Page Hit Count**（4-bit）：saturating counter，hit 时递增、miss 时递减，反映动态变化的 locality。

**Bank-level features（2 个，存储在 BRT）**：

- **Bank Recency**（4-bit）：与 page recency 类似，但跟踪的是 bank 粒度的近期访问。
- **Bank Hit Count**（8-bit）：bank 级别的 hit/miss saturating counter。

#### 预测与训练流程

**预测**：访问发生后，查找 PRT 和 BRT 中对应 entry，将各 feature 的 counter value 作为索引查询对应的 weight table，取出 weight 后求和。Sum ≥ 0 → 预测 open page；Sum < 0 → 预测 close page。若 PRT 中未找到该 page 的 entry，新建 entry 并默认 open page（LRU 替换）。

**关键设计选择**：用 counter value 直接索引 weight table 替代传统 perceptron 的乘法运算，大幅降低计算开销（借鉴了 prefetch filtering [21] 的方法）。

**训练**：

- 若预测 open 且下一次访问同 bank 是 row-buffer hit → 预测正确，不更新 weight。
- 若预测 open 且下一次访问同 bank 是 row-buffer conflict → 预测错误，对应 weight 递减。
- Close page 预测同理，错误时 weight 递增。
- 训练贯穿整个应用执行过程（online learning）。

#### Hardware Overhead

- PRT 512 entries 时，metadata 存储开销为 **2.77 KB/channel**。
- Weight tables 额外需要 **184 Bytes/channel**。
- CACTI 6.0（32nm）估算：面积 ~0.08 mm²，能耗 ~0.022 nJ/access。

### 与现有工作的区别

| 方案              | 决策粒度                | 信息来源                              | 自适应机制                             |
| --------------- | ------------------- | --------------------------------- | --------------------------------- |
| **PBRM** [3]    | Per-row             | History table 记录的 hit count       | 根据预测正确性周期更新                       |
| **INTAH** [6]   | Per-bank            | Timing register + mistake counter | 调整超时时间                            |
| **FAPS-3D** [9] | Per-bank/epoch      | Bank-level hit rate               | 2-bit counter 按 epoch 分类          |
| **DYMPL（本文）**   | Per-page/per-access | 7 个多维度 feature                    | Perceptron online learning，逐次预测训练 |

DYMPL 的核心差异在于：(1) 利用多维度 feature 而非单一指标；(2) perceptron 能自动学习各 feature 的权重，捕获 feature 间的相关性；(3) online learning 持续适应动态变化的访存行为。

## 实验评估

### 实验设置

- **仿真平台**：gem5（cycle-accurate, x86-64），扩展支持多种 page policy。能耗估算使用 DRAMPower 模型。
- **处理器**：8 核 out-of-order @ 3.6 GHz, 8-issue superscalar, 192-entry ROB
- **Cache 层次**：L1 I/D 32KB, L2 256KB, L3 16MB (shared), inclusive, MOESI
- **内存配置**：HBM, 8 GB, 8 channels, 8 DRAM layers, 16 banks/channel, 128-bit bus, 500 MHz (1 Gbps DDR), 1 KB row-buffer, tRCD = tRP = tCL = 15 cycles
- **Scheduling**：FCFS，地址映射 Row:Rank:Bank:Channel:Column
- **Workload**：SPEC CPU2006 + CPU2017 混合，12 组 8 程序 multiprogramming workload，按 MPKI 分为 HM（>20）、LM（1-20）、MX（混合）三类
- **对比 baseline**：CLOSE (S), OPEN (S), PBRM (D), INTAH (D), FAPS-3D (D)
- **仿真方法**：fast-forward 初始化 → warmup 100M 指令（同时训练 perceptron）→ 详细模拟 1B 指令

### 关键结果

1. **整体性能**：DYMPL 相比 CLOSE 平均提升 **19.8%**，相比 FAPS-3D 提升 **5.5%**，相比 OPEN/PBRM/INTAH 分别提升 12.6%/9.9%/7.6%。
2. **内存访问延迟**：DYMPL 相比 CLOSE 平均降低 **33.3%**，FAPS-3D 降低 26.9%，INTAH 降低 21.2%。
3. **Row-buffer conflict**：DYMPL 相比 OPEN 平均减少 **19.4%**（FAPS-3D 为 14.5%，INTAH 为 10.1%）。
4. **能耗**：DYMPL 相比 CLOSE 平均降低 **16.9%**（FAPS-3D 为 13.8%）。

### 结果分析

- **按 workload 类型**：HM workload 收益最大（DYMPL vs CLOSE: +29.9%），因为其对内存子系统依赖最高；LM workload 收益较小（+11.2%）。
- **Queue occupancy**：DYMPL 的 read queue 平均占用率 9.9%（CLOSE 为 14.3%），write queue 15.3%（CLOSE 为 23.9%），conflict 减少使请求处理更高效。
- **Sensitivity — PRT 大小**：256/512/1024/2048 entries 对应性能提升 4.9%/5.5%/5.9%/6.2%（vs FAPS-3D），收益递减，选择 512 作为性价比平衡点。
- **Sensitivity — Feature ablation**：Page utilization、page recency、page hotness、bank hit count、bank recency 影响较大；column stride（-0.6%）和 page hit count（-0.3%）影响较小，可考虑省略以降低开销。

## 审稿人视角

### 优点

1. **问题建模自然**：将 page open/close 决策映射为 perceptron 二分类问题是合理的。Feature 的选择涵盖了 spatial locality（utilization, stride）、temporal locality（recency, hotness）以及 bank-level 的全局信息，维度较全面。
2. **Hardware-friendly 设计**：用 counter value 索引 weight table 替代乘法，计算开销低；存储开销（~3 KB/channel）在 HBM logic die 上可接受。
3. **Sensitivity analysis 较完整**：Feature ablation 和 PRT size 的分析提供了实用的设计空间探索。

### 不足

1. **Scheduling policy 过于简化**：采用 FCFS 调度，而实际 memory controller 普遍使用 FR-FCFS 或更复杂的调度策略。在 FR-FCFS 下，scheduler 本身会优先调度 row-buffer hit 请求，这会显著改变 page policy 的效果。FCFS 下的结论能否推广到 FR-FCFS 场景存疑，且论文完全未讨论这一问题。
2. **训练信号的定义有问题**：论文用"下一次访问同 bank 是否是 row-buffer hit/conflict"作为 ground truth 来判断预测的正误。但这个定义忽略了一个关键问题——如果 page 被 close 了，那下一次访问该 bank 必然是 page miss 而非 conflict，此时无法用 conflict 来衡量 close 决策是否正确。论文对 close prediction 的 ground truth 定义不够清晰。
3. **仅用 IPC 作为性能指标，缺乏公平性指标**：在 multiprogramming workload 下，单纯的加权 IPC（geometric mean of relative IPC）不足以反映 fairness。论文未报告 unfairness 指标（如 maximum slowdown），page policy 的改变可能 starve 某些 thread。
4. **Workload 代表性有限**：仅使用 SPEC CPU 这种计算密集型/传统桌面 benchmark，缺乏对 server workload（如 PARSEC、GAPBS 等共享内存多线程应用）和真实 data-intensive workload（如 graph analytics, ML inference）的评估。SPEC CPU 的 multiprogramming 混合无法真正反映 thread 间的内存争用特征。
5. **HBM 建模细节不足**：论文声称目标平台是 HBM，但 tRCD = tRP = tCL = 15 cycles @ 500 MHz 的时序参数更像 DDR4，与 HBM2/HBM2E 的实际参数（tRCD ~14ns, tCL ~varies）不太匹配。且 HBM 的 pseudo-channel 模式、bank group 等特性均未体现。
6. **发表在 MEMSYS 而非顶会**：MEMSYS 的审稿标准和影响力远低于 ISCA/MICRO/HPCA，论文的实验严谨性和 baseline 覆盖度也反映了这一点。
7. **至少应该与 Ipek et al. (ISCA'08) 讨论对比**。虽然 Ipek 的工作做的是整个 scheduling 而不仅是 page policy，但两者的核心思想高度一致——都是用 ML 从系统状态学习 DRAM 管理策略。Ipek 用 RL 可以隐式地做出包含 page policy 在内的全局最优决策，而 DYMPL 的 perceptron 只做 page open/close 这个局部决策。论文完全没有提到这篇 ISCA 经典工作是一个明显的 related work 缺失
8. **Perceptron 作为线性模型的局限性未讨论**。与 RL 或更复杂的非线性模型相比，perceptron 无法捕获 feature 间的非线性交互。论文应该讨论为什么选择 perceptron 而不是更强的模型（hardware cost 是一个合理的理由，但需要量化分析）。
9. **Online learning 的 exploration-exploitation tradeoff 未讨论**。Ipek 的工作详细分析了 ε-greedy 策略来平衡 exploration 和 exploitation，而 DYMPL 直接用 perceptron 的 deterministic prediction，没有探索机制，可能导致陷入局部最优。

### 疑问或值得追问的点

- Perceptron 的 prediction latency 是否在 critical path 上？论文未分析预测延迟对 memory scheduling 的影响。如果预测需要多个 cycle，可能会延迟 command 的发射。
- Online learning 的收敛速度如何？在 phase change 剧烈的应用中，perceptron 的适应速度是否足够快？
- 7 个 feature 中有些是高度相关的（如 page hotness 和 page hit count，bank recency 和 bank hit count），perceptron 作为线性模型能否有效处理 feature 冗余？是否做过 feature correlation 分析？
- Address mapping 使用 Row:Rank:Bank:Channel:Column，这个映射方案本身对 row-buffer hit rate 有很大影响。换用其他常见映射方案（如把 channel/bank bits 放在更低位）时，DYMPL 的效果如何？
