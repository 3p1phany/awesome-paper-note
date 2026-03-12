---
title: "PROFET: Modeling System Performance and Energy Without Simulating the CPU"
authors: "Milan Radulovic, Rommel Sánchez Verdejo, Paul Carpenter, Petar Radojković, Bruce Jacob, Eduard Ayguadé"
venue: "Proceedings of the ACM on Measurement and Analysis of Computing Systems (POMACS / ACM SIGMETRICS) 2019"
year: 2019
---
# PROFET: Modeling System Performance and Energy Without Simulating the CPU

## 基本信息

- **发表**：Proc. ACM Meas. Anal. Comput. Syst. (POMACS), Vol. 3, No. 2, Article 34, June 2019
- **作者单位**：Barcelona Supercomputing Center (BSC), Universitat Politècnica de Catalunya (UPC), University of Maryland
- **开源**：https://github.com/bsc-mem/PROFET

## 一句话总结

> 基于真实硬件 profiling 和 bandwidth-latency curve，无需 CPU 仿真即可精确预测更换内存系统后的应用性能、功耗和能耗变化。

## 问题与动机

**要解决的问题**：DRAM scaling 逐渐逼近极限，新兴内存技术（如 3D-stacked MCDRAM、HBM 等）的研究需要快速、准确地评估应用在不同内存系统上的性能和能耗表现。

**现有方法的不足**：

1. **硬件仿真器慢且不准**：传统做法是使用 CPU simulator + memory simulator（如 ZSim + DRAMSim2）来评估新内存配置的性能。但 CPU 仿真器存在严重问题：
   - 仿真速度极慢，限制了可探索的设计空间
   - CPU 模型对 out-of-order execution 的建模过于简化
   - data prefetcher 的建模通常是过时的或不准确的
   - 缺乏 virtual-to-physical memory translation
   - 这些不足导致仿真结果与真实硬件之间存在巨大偏差
2. **已有解析模型依赖仿真器做 profiling**：Karkhanis-Smith 模型、Eyerman 的 mechanistic model 等解析模型虽然不需要仿真来做预测，但它们的 application profiling（如获取 MLP、cold/capacity/conflict miss 分类等）仍然需要硬件仿真器，无法完全脱离仿真。
3. **已有工作对内存带宽-延迟关系的处理过于粗糙**：Clapp et al. (2015) 认为不同频率和读写比的内存系统可以用单一 bandwidth-latency curve 近似，但本文实测表明这一结论是错误的。

## 核心方法

### 关键思路

PROFET 的核心洞察是：**应用在不同内存系统上的性能差异，可以通过应用在不同 bandwidth-latency curve 之间的"移动"来理解和预测**。具体而言，每个内存系统有其固有的 bandwidth-latency curve（随负载增加，latency 非线性上升），而应用运行时必然处于该 curve 上的某一点。更换内存系统等价于应用从一条 curve "跳"到另一条 curve，新的工作点由性能模型曲线与目标内存 curve 的交点唯一确定。

这一方法的关键优势在于：application profiling 完全基于真实硬件的 performance counter，CPU 的 prefetcher、OOO engine 等微架构细节已经隐含在实测数据中，无需显式建模。

### 技术细节

#### 整体架构

PROFET 的完整流程包含三个模型（performance、power、energy），需要四类输入：

1. **Memory system profiling**：baseline 和 target 内存系统各自的 bandwidth-latency curve 族
2. **CPU parameters**：ROB size、MSHR size、CPImin（最大理论 IPC 的倒数）
3. **Application profiling**：在 baseline 系统上运行应用，通过 hardware performance counter 采集每 1 秒 sampling segment 的 Cyctot、Instot、MissLLC、BWused、RatioR/W
4. **Memory power parameters**：IDD 电流、电压、timing 参数等（来自内存厂商 datasheet）

#### Memory System Profiling — Bandwidth-Latency Curve 族

这是 PROFET 的基础输入。关键技术点：

- **测量方法**：使用 pointer-chasing microbenchmark 测量 latency，同时运行修改版 STREAM benchmark 施加可控带宽负载。单个内存配置的 profiling 约需 15 分钟。
- **curve 族而非单一 curve**：一个内存系统并非只有一条 bandwidth-latency curve，而是一个 **以读写比为参数的 curve 族**。读写比从 50% reads 到 100% reads 变化时，curve 的形状和位置有显著差异。在高带宽利用率下，50% reads（即 50% writes）比 100% reads 的 latency 高出 100ns (76%)——因为 write 操作会引入额外的 tWR 和 tWTR 延迟。
- **反驳 Clapp et al. 的结论**：本文基于 DDR3、DDR4、MCDRAM 多种频率和细粒度读写比的大量实测数据，证明不同频率的内存有根本不同形状的 bandwidth-latency curve，不能用单一通用曲线近似。

#### Performance Model — 从 In-order 到 Out-of-order

**Step 1: CPI 分解（通用）**

将应用 CPItot 分解为两个分量：

$$
CPI_{tot} = CPI_0 + CPI_{LLC}
$$

其中 CPI0 是假设 100% LLC hit rate 时的 CPI（不受内存延迟影响），CPILLC 是 LLC miss 导致的 stall。

关键假设：更换内存系统不改变 Instot（指令数）和 MissLLC（LLC miss 数），也不改变 CPI0。因此：

$$
CPI_{tot}^{(2)} = CPI_{tot}^{(1)} + \frac{Miss_{LLC}}{Ins_{tot}} \times (Stalls_{LLC}^{(2)} - Stalls_{LLC}^{(1)})
$$

**Step 2: In-order processor（简单情形）**

对于 in-order processor，LLC miss 直接导致 pipeline stall，即 StallsLLC = Penmem。代入得：

$$
IPC_{tot}^{(2)} = \frac{1}{\frac{1}{IPC_{tot}^{(1)}} + \frac{Miss_{LLC}}{Ins_{tot}} \times (Pen_{mem}^{(2)} - Pen_{mem}^{(1)})}
$$

**Step 3: Out-of-order processor（核心贡献）**

OOO processor 在 LLC miss 后可以继续执行独立指令，因此 StallsLLC < Penmem。引入 Insooo（LLC miss 后 OOO 执行的独立指令数）和 MLP（memory level parallelism）：

$$
Stalls_{LLC} = \frac{1}{MLP} \times (Pen_{mem} - CPI_0 \times Ins_{ooo})
$$

**Insooo 的界定**（这是无法直接测量的参数，采用 sensitivity analysis）：

- 下界：Insooo ≥ 0（OOO 可能立即 stall）
- 上界 1：InsROB（ROB 容量，Sandy Bridge 168 entries，KNL 72 entries）
- 上界 2：Penmem × Instot / Cyctot（OOO 完全覆盖 memory penalty 的理论极限）
- 最终上界：两者取最小值

**MLP 的估计**（本文的关键创新之一）：

- 通过公式推导得到 MLP 关于 Insooo 和 CPI0 的表达式：

$$
MLP(Ins_{ooo}, CPI_0) = \frac{\frac{Miss_{LLC}}{Ins_{tot}} \times (Pen_{mem}^{(1)} - CPI_0 \times Ins_{ooo})}{CPI_{tot}^{(1)} - CPI_0}
$$

- CPI0 有界：CPImin ≤ CPI0 ≤ CPItot^(1)
- MLP 上界还受 MSHR size 限制（Sandy Bridge 为 10，KNL 为 12）
- **Point estimate**：假设 LLC miss 在 sampling segment 内均匀分布，得到 MLP_E(Insooo) = (MissLLC / Instot) × Insooo + 1

**Step 4: 最终性能公式**

综合上述分析，target 系统的 IPC 预测为：

$$
IPC_{tot}^{(2)} = \frac{IPC_{tot}^{(1)}}{1 + IPC_{tot}^{(1)} \times \frac{Lat_{mem}^{(2)} - Lat_{mem}^{(1)}}{Ins_{ooo} + Ins_{tot}/Miss_{LLC}}}
$$

注意 Penmem 差值等于 Latmem 差值（LLC hit latency 在 baseline 和 target 上相同，相消）。

#### Performance Estimation 的最终求解 — 曲线交点法

这是整篇论文最精巧的部分。核心问题在于：降低 memory latency → IPC 提升 → 带宽增加 → latency 因 contention 回升，存在反馈效应。

求解方法：

1. 将 IPC-latency 关系（Eq. 17）转换为 bandwidth-latency 关系：$BW_{used}^{(2)} = (BW_{used}^{(1)} / IPC_{tot}^{(1)}) × IPC_{tot}^{(2)}$
2. 将此关系曲线叠加到 target 内存的 bandwidth-latency curve 上
3. 两条曲线的交点即为应用在 target 内存上的工作点
4. 交点唯一存在性保证：内存 curve 单调递增（latency 随 bandwidth 增加），性能模型曲线单调递减（bandwidth 随 latency 增加而降低）
5. 使用 bisection method 求交点
6. 对 Insooo 在其有效范围内 sweep，得到性能预测的 point estimate 和 error bar（min, max）

#### Power Model

基于 Micron 功耗计算指南，将总平台功耗分解为：

$$
P_{tot}^{(2)} = P_{tot}^{(1)} + (P_{mem}^{(2)} - P_{mem}^{(1)})
$$

内存功耗进一步分解为 background power（active standby, precharge power-down, self-refresh 三种状态加权）和 operational power（read/write access energy × access 数量 / 采样时间 + refresh power）。

关键假设：内存 power-down 状态时间比例和 row-buffer hit/miss 率在 baseline 和 target 之间不变。

#### Energy Model

$$
E_{tot}^{(2)} = \sum_{i=1}^{N} P_{tot,i}^{(2)} \times \frac{IPC_{tot,i}^{(1)}}{IPC_{tot,i}^{(2)}} \times \Delta t_i^{(1)}
$$

即每个 segment 的能耗 = 估计功耗 × 调整后的时间（因性能变化导致执行时间变化）。

### 与现有工作的区别

| 对比维度              | PROFET                     | Karkhanis-Smith / Eyerman Mechanistic Model | Clapp et al. (2015)    |
| --------------------- | -------------------------- | ------------------------------------------- | ---------------------- |
| Application profiling | 仅需真实硬件 perf counter  | 需要硬件仿真器                              | 真实硬件               |
| Prefetcher 建模       | 隐含在真实执行中           | 不包含或简化                                | 不详                   |
| Memory latency 处理   | Bandwidth-latency curve 族 | 常数 latency 或仿真得到                     | 单一通用 curve         |
| Read/Write ratio 影响 | 精细建模（curve 族）       | 不考虑                                      | 认为可忽略（本文反驳） |
| 验证方式              | 真实硬件实测               | 仿真器交叉验证                              | 有限验证               |
| MLP 估计              | 基于 perf counter 的新方法 | 需要仿真器数据                              | 简化处理               |

## 实验评估

### 实验设置

- **硬件平台**：
  - Sandy Bridge-EP E5-2670：2 socket × 8 cores, 3.0 GHz, 20 MB L3, 4 channel DDR3-800/1066/1333/1600, 64 GB
  - Knights Landing Xeon Phi 7230：1 socket × 64 cores, 1.3 GHz, 无 L3, 6 channel DDR4-2400 (96 GB) + 8 channel MCDRAM (16 GB), flat mode
- **Workload**：
  - SPEC CPU2006 全套 29 个 benchmark
  - 4 个 UEABS HPC 应用：ALYA, GROMACS, NAMD, Quantum Espresso（仅 Sandy Bridge，KNL 内存容量不足）
  - Sandy Bridge 上 16 copies 全满载；KNL 上受 MCDRAM 容量限制，每个 benchmark 实例数不同（8~64）
- **工具**：LIKWID performance counter suite，Yokogawa WT230 功率计
- **对比 baseline**：ZSim + DRAMSim2 硬件仿真器（Sandy Bridge 平台）

### 关键结果

1. **Performance 预测精度极高**：

   - Sandy Bridge DDR3-800→1600：高带宽 benchmark 平均误差 5.1%，低带宽 benchmark 平均误差 1.6%
   - KNL DDR4→MCDRAM：高带宽 benchmark 平均误差 7%，低带宽 benchmark 平均误差 1.6%
   - 整体平均性能预测误差约 2%
2. **大幅优于硬件仿真器**：

   - Sandy Bridge DDR3-800→1600：PROFET 平均误差 3.6% vs. ZSim+DRAMSim2 平均误差 15.7%
   - 仿真器的误差范围极大（-23.7% 到 45.7%），且趋势方向都是错的（低估高带宽 benchmark 增益，高估低带宽 benchmark 增益）
3. **Power 和 Energy 预测同样精确**（Sandy Bridge）：

   - Power 预测平均误差 < 1%（DDR3 频率翻倍仅导致 ~2% 整体功耗增加）
   - Energy 预测平均误差 < 2%（DDR3-800→1600 平均节能 13%，libquantum 最高达 41%）
4. **速度优势约三个数量级**：

   - PROFET 总计 1040 秒（含 memory profiling 910s + application profiling 127s + model 执行数秒）
   - ZSim+DRAMSim2 需要 242 小时

### 结果分析

- **高带宽 benchmark（>50% BW 利用率）** 对内存配置变化最敏感，也是建模最具挑战性的对象。DDR3-800→1600 可带来最高 80%（libquantum）的性能提升。
- **KNL DDR4→MCDRAM** 的有趣发现：虽然 MCDRAM 带宽为 DDR4 的 4.2 倍，但其 lead-off latency 高 23ns，导致低带宽 benchmark（如 mcf）性能反而下降 9%。PROFET 准确捕获了这一现象。
- **Error bar 宽度与 ROB size 正相关**：KNL 的 72-entry ROB 导致 Insooo 范围更窄，因此预测更精确（error bar 仅 3.2% for HBW vs. Sandy Bridge 的 7.4%）。
- **Sensitivity analysis**：Insooo 作为 free parameter 的 sweep 对低带宽 benchmark 几乎无影响（因为 Lat 差异小或 MissLLC 少），对高带宽 benchmark 影响也可控。
- **Prefetcher 稳定性验证**：Sandy Bridge 上 DDR3-800 与 DDR3-1600 之间，每条指令的 prefetch 数量差异 < 5%，支持了"prefetcher 行为不受内存频率显著影响"的假设。

## 审稿人视角

### 优点

1. **方法论创新且实用性极强**：完全基于真实硬件 performance counter 的 profiling，避免了 CPU simulator 的一切建模误差。这不仅是学术上的创新，更具有很强的工业实用价值——任何拥有现有服务器的团队都可以用 PROFET 快速评估换装新内存的预期收益。
2. **Bandwidth-latency curve 族的发现具有独立学术贡献**：证明了 read/write ratio 对 loaded latency 有显著影响（最高 76%），反驳了 Clapp et al. 的"单一 curve 足够"结论，这一发现对整个内存系统建模领域都有价值。
3. **验证充分且令人信服**：在两个截然不同的平台（mainstream Sandy Bridge vs. emerging KNL with MCDRAM）、多种内存配置、大量 benchmark（SPEC CPU2006 全套 + HPC 应用）上验证，且与真实硬件实测对比。同时与 ZSim+DRAMSim2 做了 head-to-head 对比，展示了量级级别的准确度和速度优势。
4. **曲线交点求解方法优雅**：将 bandwidth-latency curve 和 performance model curve 的交点作为应用在 target 系统上的工作点，并严格证明了交点存在且唯一，数学上完备。feedback loop（性能变化→带宽变化→latency 变化→性能再变化）的处理方式简洁而正确。
5. **完全开源**：代码、所有输入数据、评估结果均公开发布，reproducibility 极好。

### 不足

1. **仅支持单一 CPU 微架构不变的场景**：PROFET 的核心假设是 baseline 和 target 共享同一 CPU，仅内存系统不同。这限制了其适用场景——无法评估同时更换 CPU 和内存的情形，也无法用于还没有物理硬件的全新 CPU 架构探索。
2. **Insooo 无法精确确定**：虽然 sensitivity analysis 的 error bar 在实验中表现窄，但这是因为评估的两个平台 ROB 较小（72 和 168）。对于更现代的处理器（如 Apple M 系列 ROB 超过 600 entries），Insooo 范围可能大幅膨胀，error bar 是否仍然可控？论文未讨论这一 scalability 问题。
3. **Power/Energy model 未在 KNL MCDRAM 上验证**：作者承认缺乏可靠的 MCDRAM 功耗参数。对于新兴内存技术（正是 PROFET 目标受众最关注的场景），power model 的适用性未得到验证。
4. **多核共享资源竞争的建模较为隐含**：虽然实验是在满载（16 cores）下进行的，但 PROFET 通过实测 bandwidth-latency curve 隐含了多核竞争效应。当核数或线程配置改变时（如从 16 核到 8 核），需要重新测量 curve，模型本身无法预测不同并发度下的行为。
5. **假设局限性**：

   - 假设 Instot（指令数）和 MissLLC 在不同内存配置下不变，对于使用 busy-waiting 或 dynamic scheduling 的应用不成立
   - 假设 row-buffer hit/miss ratio 不变——但不同内存频率下 MC scheduling 行为可能改变 page policy 效果
   - 1 秒的 sampling interval 粒度可能对 phase behavior 非常剧烈的应用（如 mcf）不够细
6. **评估平台已较过时**：Sandy Bridge（2011）和 KNL（2016）在论文发表时（2019）已不算新。对更现代的平台（如 DDR5、CXL-attached memory、HBM2E/3）的适用性需要额外验证。虽然作者声称迁移到 KNL 只需改几个参数，但 DDR5 引入的 same-bank refresh、on-die ECC 等新特性是否会影响 bandwidth-latency curve 的特征，尚未讨论。

### 疑问或值得追问的点

- **对 CXL-attached memory 的适用性**：CXL memory 引入了额外的协议层延迟和非确定性延迟抖动，bandwidth-latency curve 的形状可能与传统 DRAM 有质的不同。PROFET 的 curve 族方法是否仍然适用？
- **对 HBM/HBM2E 的验证**：KNL 的 MCDRAM 是早期 3D-stacked memory，与 HBM2E/HBM3 有较大差异。GPU 侧的 HBM 验证将大幅增强论文说服力。
- **Prefetcher sensitivity**：论文声称 prefetcher 行为在不同频率间稳定（差异 < 5%），但这是 Sandy Bridge 时代较简单的 prefetcher。现代 CPU 的 prefetcher 更加激进和自适应，这一假设是否仍然成立？
- **对 LLC partitioning 或 heterogeneous memory tiering 场景的扩展**：现代数据中心越来越多地使用 Intel CAT 做 LLC 分区或 CXL memory tiering，PROFET 能否扩展到这些场景？
- **能否反向使用**：即给定性能目标，反推所需的内存系统参数（bandwidth、latency 要求），用于内存系统设计空间探索？
