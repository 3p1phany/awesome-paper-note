---
title: "DRAM Controller with a Close-Page Predictor"
authors: "Vladimir V. Stankovic, Nebojsa Z. Milenkovic"
venue: "EUROCON 2005"
year: 2005
---

# DRAM Controller with a Close-Page Predictor

## 基本信息
- **发表**：EUROCON 2005, Belgrade, Serbia & Montenegro, November 22-24, 2005
- **作者单位**：University of Nis, School of Electronic Engineering

## 一句话总结
> 结合 zero live time predictor 和 dead time predictor，构建 close-page predictor 以自适应决定何时关闭已打开的 DRAM row。

## 问题与动机

DRAM 的访问延迟取决于 row buffer 的状态：若访问命中当前已打开的 row（row hit），延迟仅为 Tca（column access time）；若访问不同 row（row conflict），则需付出完整的 Tpr + Tra + Tca 代价。经典的两种 page policy 各有局限：

- **Open Page Policy**：保持 row 打开，locality 好时表现优，但遇到 row conflict 时会叠加 Tpr 开销。
- **Close Page (Autoprecharge) Policy**：每次访问后立即关闭 row，延迟恒定为 Tra + Tca，但浪费了 row hit 的机会。

理想情况是：在一个 row 的最后一次访问完成后立即关闭它，从而将 Tpr 与后续访问的 Tra 重叠（hide precharge latency）。这需要准确预测"当前 row 是否已经不会再被访问"。作者此前在 [1] 中提出了一个基于 dead time 的简单预测器，本文在此基础上增加了 zero live time predictor 来补全预测能力。

## 核心方法

### 关键思路

核心洞察有两个：（1）许多被打开的 row 在首次访问后就不再被重复访问（即 zero live time 现象），对这类 row 应立即发出 autoprecharge；（2）对于确实有后续访问的 row，可以利用"average dead time 远大于 average access interval"这一统计规律，用上一次 access interval 的倍数作为 timeout 阈值来预测 row 是否已进入 dead time。两个预测器级联工作，先判 zero live time，再判 dead time。

### 技术细节

**整体架构：Close-Page Predictor = Zero Live Time Predictor + Dead Time Predictor**

工作流程如下：
1. 当一个新 row 被 activate 时，首先查询 **zero live time predictor**。
2. 若预测为 zero live time → 本次访问使用带 autoprecharge 的命令，访问完成后立即关闭 row。
3. 若预测为 non-zero live time → 保持 row open，后续由 **dead time predictor** 接管，持续监控是否应关闭。
4. Dead time predictor 判定 row 已进入 dead time → 发出 precharge 命令关闭 row。

#### Zero Live Time Predictor（三种变体）

**ZLT1（1-bit per row）**：为每个 DRAM row 维护 1 bit，记录该 row 上次打开时是否为 zero live time。下次打开时，基于"上次是 zero live time 则预测本次也是"的 last-value prediction 策略。所有 row 初始预测为 non-zero live time（等价于默认 Open Page Policy）。

- 硬件开销：以 4 banks × 4096 rows 为例，需要 16Kbit = 2KB 的 SRAM。

**ZLT2（2-bit saturating counter per row, decrement on miss）**：为每个 row 维护一个 2-bit 饱和计数器（值域 0-3）。zero live time 发生时 +1（最大不超过 3），non-zero live time 发生时 -1（最小不低于 0）。计数器值 ≥ 2 时预测 zero live time，否则预测 non-zero。初始值为 0。

**ZLT2'（2-bit saturating counter per row, reset on miss）**：与 ZLT2 类似，唯一区别是 non-zero live time 发生时将计数器直接 reset 为 0（而非 -1）。这意味着 ZLT2' 对 non-zero live time 的反应更激进——一次 miss 就完全清零信心。

#### Dead Time Predictor

基于 [1] 中的观察：average dead time 远大于 average access interval。

- 为每个 bank 维护一个 **elapsed time counter**（记录自上次访问以来的时间）。
- 维护一个 **全局共享的 access interval register**（记录最近一次两个连续 bank 访问之间的间隔）。
- 当 elapsed time ≥ last access interval × 2 时，预测该 row 已进入 dead time，发出 precharge。
- 乘以 2 通过简单的 1-bit shift 实现，硬件开销极低。

#### 设计权衡

- **ZLT1 vs ZLT2/ZLT2'**：ZLT1 最简单（1-bit per row），但容易在 zero/non-zero live time 交替出现时频繁误判。ZLT2/ZLT2' 通过 hysteresis（2-bit 计数器）过滤噪声，显著减少 misprediction 数量。代价是存储翻倍（2-bit per row）。
- **错误的不对称性**：错误地关闭一个本应保持打开的 row（miss-close）比错误地保持一个本应关闭的 row（miss-open）代价更大。因为 miss-open 时 dead time predictor 仍有机会补救，而 miss-close 后 dead time predictor 无法纠正。这一洞察解释了为何 ZLT2/ZLT2' 虽然在 accuracy 百分比上有时不如 ZLT1，但在 miss-close 次数上远优于 ZLT1（如 m88ksim 在 rgbc 下：ZLT1 有 80 次 miss-close，ZLT2/ZLT2' 仅 1 次）。
- **Dead time predictor 的 boundary level（×2 vs ×4）和 register scope（global vs per-bank）**：[1] 中实验表明两者差异不显著，本文统一采用 ×2 和 global 配置以简化设计。作者也指出这与 single-program 环境有关，multi-program 下 per-bank 可能更有意义。

### 与现有工作的区别

| 方案 | 策略 | 局限 |
|------|------|------|
| Open Page Policy | 始终保持 row open | row conflict 时付出 Tpr + Tra + Tca 的完整代价 |
| Close Page (Autoprecharge) | 每次访问后立即关闭 | 浪费 row hit 机会，延迟恒为 Tra + Tca |
| Dead Time Predictor [1] | 基于 access interval timeout 预测何时关闭 | 无法处理 zero live time 场景——row 打开后若无后续访问，需等到 timeout 才关闭 |
| **本文 Close-Page Predictor** | ZLT predictor + Dead Time Predictor 级联 | 在 [1] 基础上增加 zero live time 预测，覆盖了 [1] 遗漏的场景 |

另外，核心 idea 借鉴了 cache 领域的 dead block prediction 工作 [2][3]（Hu et al., ISSCC 2003; Lai et al., ISCA 2001），将 cache line 的 dead time / live time 概念迁移到 DRAM row buffer 管理上。

## 实验评估

### 实验设置

- **仿真平台**：SimpleScalar Tool Set（Sim-Outorder），集成作者自行编写的 DRAM 模拟程序。
- **处理器配置**：2 GHz superscalar processor，最多 4-wide issue，out-of-order execution，two-level branch predictor。
- **Cache 层次**：L1 I-Cache 16KB direct-mapped 32B line；L1 D-Cache 16KB direct-mapped 32B line；L2 Unified 1MB 4-way set-associative 128B line；全部 write-back。
- **DRAM 配置**：4 banks，4096 rows/bank，1KB row size；Tpr = Tra = Tca = 20 processor cycles（即 10ns @2GHz）；4B burst width。
- **Workload**：SPEC95 中 6 个程序——cc1, compress, ijpeg, li, m88ksim, perl。
- **Address Remapping**：3 种方案——rgbc（经典 page interleaving）、rgrbcx、rbrgcx（后两者来自 [5]，优化 row hit rate）。
- **对比 Baseline**：Open Row Policy (OR)、Dead Time Predictor (DTP) [1]、Close Page Predictor 三变体 (CPP1/CPP2/CPP2')、Ideal（100% 准确率的理想预测器）。

### 关键结果

1. **ZLT2/ZLT2' 在 miss 次数上显著优于 ZLT1**：以 rgbc 下 m88ksim 为例，ZLT1 产生 1 hit / 80 misses，而 ZLT2 和 ZLT2' 仅 0 hit / 1 miss——miss-close 数量减少约 80 倍。

2. **Prediction accuracy 因 workload 和 address remapping 差异显著**：rgbc 下 cc1 和 ijpeg 的 ZLT2 accuracy 达 0.83 和 0.98；但 rgrbcx/rbrgcx 下部分 benchmark（如 ijpeg 在 rgrbcx 下 ZLT1 仅 0.14）accuracy 大幅下降。这说明 address remapping 改变了 row 访问模式，影响 zero live time 的可预测性。

3. **CPP2/CPP2' 几乎在所有配置下优于或持平 DTP**：从 Fig.1-3 的 average latency 来看，CPP2 和 CPP2'（带 2-bit ZLT predictor 的完整方案）在绝大多数 benchmark × address remapping 组合下，延迟低于或等于纯 DTP。而 CPP1 在部分配置下反而恶化 DTP（如 ccl, li 在 rgrbcx 和 rbrgcx 下）。

4. **多数场景下 CPP 接近 Ideal**：Fig.1-3 显示 CPP2/CPP2' 的 average latency 在多数 benchmark 下已非常接近 Ideal predictor 的理论最优值，尤其在 rgbc 配置下。

### 结果分析

- **rgbc vs rgrbcx/rbrgcx**：rgbc 是经典 page interleaving，row hit rate 较低（更多 zero live time），反而使得 zero live time predictor 有更多施展空间且模式更规律。rgrbcx/rbrgcx 通过 remapping 提升了 row hit rate，导致 zero live time 出现频率和模式变化，prediction accuracy 下降。
- **CPP1 的局限**：1-bit predictor 的 thrashing 问题在 row 行为不稳定时暴露明显（频繁 miss-close），而 2-bit hysteresis 有效缓解。
- **未做 sensitivity analysis**：论文没有对 DRAM timing 参数、cache 大小、memory intensity 等进行敏感性分析。

## 审稿人视角

### 优点

1. **问题定义清晰，方法直觉合理**：将 DRAM row 的 page policy 决策分解为"是否 zero live time"和"何时进入 dead time"两个子问题，逻辑上互补且完整。级联式设计使得两个简单预测器各司其职。

2. **对预测错误不对称性的分析有价值**：明确指出 miss-close 比 miss-open 代价更高，并以此论证为何 accuracy 数字不能直接反映预测器优劣（ZLT2/ZLT2' accuracy 看似低但 miss 数大幅减少），这一分析视角在当时是有意义的。

3. **硬件实现开销极低**：整个预测器只需 per-row 1-2 bit SRAM + per-bank counter/comparator + 一个全局 register，适合集成到实际 memory controller 中。

### 不足

1. **实验方法论严重过时且规模不足**：
   - 仅使用 **SPEC95**（1995 年的 benchmark），即使在 2005 年发表时 SPEC CPU2000 已经广泛使用。6 个 benchmark 样本量过小。
   - SimpleScalar 的 DRAM 模型由作者自行编写，缺乏与标准 DRAM 模拟器（如 DRAMSim）的交叉验证，模型准确性存疑。
   - DRAM 配置过于简化：单 channel、4 banks、Tpr=Tra=Tca=20 cycles 的对称 timing 在真实 DRAM 中并不典型（通常 tRCD ≠ tRP ≠ tCL）。

2. **缺少关键 baseline 对比**：
   - 没有与 Close Page (Autoprecharge) Policy 进行对比，这是最基本的 baseline 之一。
   - 没有与同期其他 adaptive page policy 工作对比（如 Rixner et al., ISCA 2000 中的 adaptive open/close page 策略）。

3. **Zero live time predictor 的 per-row 存储可扩展性存疑**：作者以 4 banks × 4096 rows = 2KB 为例说明开销小，但现代 DRAM 可能有 8-16 banks × 数万 rows × 多 rank/channel，存储需求会快速膨胀。论文未讨论 tag-based 或 sampled 替代方案。

4. **Dead time predictor 过于粗糙**：仅用一个全局共享的 last access interval × 2 作为 timeout 阈值，没有 per-bank 或 per-row 的自适应，也没有考虑 access interval 的分布特征。在 multi-program 或 memory-intensive workload 下，这种全局阈值很可能失效。

5. **缺乏对 bandwidth 和 system-level performance 的评估**：仅报告 average DRAM latency，没有 IPC、system throughput、bandwidth utilization 等指标。Page policy 的选择不仅影响延迟，还影响 bank-level parallelism 和带宽利用率，单看延迟不够全面。

6. **论文写作和数据呈现质量一般**：Table I-III 的 OCR 质量差导致部分数据难以辨认；Fig.1-3 分辨率低；缺少 error bar 或 confidence interval。

### 疑问或值得追问的点

- Dead time predictor 使用全局 access interval 而非 per-bank access interval，在单程序环境下差异不大，但作者是否考虑过在 multi-programmed / multi-threaded workload 下 per-bank 或 per-source 的自适应？
- Zero live time predictor 的 per-row 状态在 row 数量很大时如何处理？是否可以用 set-sampled 方式（类似 cache sampling）降低开销？
- 如果将 zero live time prediction 与 close page autoprecharge 结合考虑，是否存在与 DRAM 的 tRAS（minimum row active time）约束的冲突？即预测为 zero live time 后立即 autoprecharge，但 tRAS 可能尚未满足。
- 论文的 "Ideal" baseline 定义为 100% accuracy 的 close-page predictor，但这并非真正的理论最优——真正的最优应该是 Belady-style 的 omniscient policy，同时考虑 prefetch 和 row 预打开。Ideal 的定义偏保守。
