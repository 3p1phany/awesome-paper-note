---
title: "Prediction Based DRAM Row-Buffer Management in the Many-Core Era"
authors: "Manu Awasthi, David W. Nellans, Rajeev Balasubramonian, Al Davis"
venue: "PACT 2011"
year: 2011
---

# Prediction Based DRAM Row-Buffer Management in the Many-Core Era

## 基本信息
- **发表**：International Conference on Parallel Architectures and Compilation Techniques (PACT), 2011
- **机构**：School of Computing, University of Utah

## 一句话总结
> 提出轻量级的 Access Based Predictor (ABP)，以 per-page 访问次数历史替代 timer 机制来决定 row-buffer 关闭时机，在 many-core 场景下以 20KB 开销获得显著性能提升。

## 问题与动机

在 many-core 时代，多个线程/核心产生的 memory access stream 在 memory controller 处被 interleave，导致单个 stream 原本具有的 spatial locality 被严重稀释，合并后的访问流呈现越来越强的随机性。这直接冲击了传统依赖 locality 的 DRAM 优化策略：

- **Open-page policy**：在单核时代非常有效，但多核 interleaving 导致 row-buffer conflict 急剧增加，反而引入额外的 write-back latency（需要先 precharge 再 activate 新 row）。
- **Closed-page policy**：对随机访问友好，但完全放弃了利用 locality 的可能性，当仍存在部分 locality 时会带来不必要的功耗和延迟开销。
- **Timer-based hybrid policy**（如专利 [2-4] 和学术工作 [5]）：用计时器决定何时关闭 row-buffer，但需要精确预测一个跨越数百到数千 cycle 的 timer 值 N 才能逼近 oracle 性能，这在实践中非常困难。

现有工作的核心不足：几乎所有 hybrid row-buffer management 方案都以 **time** 作为关闭 row-buffer 的底层判据，而非直接跟踪 **access count**——后者才是 locality 更直接的度量。唯一在 per-page 粒度做 access-based prediction 的 prior work（Xu et al. [8], SAMOS 2009）采用了类似 branch predictor 的 two-level 结构，per-page 跟踪的硬件开销过大，作者自己也承认不实用，退而求其次使用 per-bank 或 global 粒度，牺牲了预测精度。

## 核心方法

### 关键思路

核心洞察在于：**access-based prediction 天然比 timer-based prediction 更容易逼近 oracle**。一次完美的 access count 预测就能精确地在最后一次有用访问后关闭 row-buffer，实现最小 conflict penalty；而 timer-based 方法必须猜对一个以 cycle 为单位的时间窗口，预测空间巨大，准确率天然受限。

ABP 的关键设计决策是：**用 one-level 结构替代 Xu09 的 two-level 结构**，在保持 per-page 粒度的同时将硬件开销压缩到可接受范围。

### 技术细节

**ABP 的工作流程：**

1. **首次访问某 DRAM-page 时**：查找 history table。若无对应 entry，则保持 row-buffer open，直到发生 page-conflict（退化为 open-page 行为）。当 row-buffer 被 conflict 关闭时，将本次观测到的 access count 记录到 history table。

2. **history table 中存在 entry 时**：读取预测的 access count，在达到该次数后主动关闭 row-buffer（或因 page-conflict 提前关闭）。

3. **预测结果的反馈更新机制**：
   - **Conflict 发生（预测偏大）**：将 history table 中的 access count 减 1，期望下次更准确。
   - **预测完美（关闭后下一次访问确实是不同 page）**：不更新，保持当前值。
   - **预测过早关闭（关闭后同一 page 又被重新 activate）**：允许 row-buffer 保持 open 直到下次 conflict，然后用 **累计的** access count 更新 history table，避免后续再次 premature closure。

**硬件组织：**

- History table 组织为 **2048-set / 4-way** 的 cache 结构。
- 32 个 DRAM bank 各自拥有独立的 predictor，每个 bank 是 **64-set / 4-way**（64 × 4 × 32 = 8192 entries 总计，与 2048-set / 4-way 对应）。
- 总存储开销仅 **20 KB**。
- Predictor lookup **不在 critical path 上**：预测只需在 RAS（activate）和 CAS（read/write）之后生效，因此 lookup latency 可以被隐藏。
- 平均 hit rate 达到 **92.6%**。

**Xu09 (baseline predictor) 的结构对比：**

Xu09 采用类 branch predictor 的 two-level 结构：第一级是 n-bit shift register (History Register, HR) 记录 row-hit/row-miss 序列；第二级是 History Table (HT)，为所有 2^n 种 pattern 维护 saturating counter。HR 和 HT 可在 global、per-bank、per-page 三个粒度组织。论文中对 Xu09 建模为 per-bank HR + global HT 的配置。

### 与现有工作的区别

| 维度 | ABP (本文) | Xu09 [8] | Timer-based [2-5] |
|------|-----------|----------|-------------------|
| 预测判据 | Access count | Hit/miss pattern → saturating counter | 时间（cycle 数） |
| 预测器结构 | One-level (直接查表) | Two-level (HR + HT) | 计数器/计时器 |
| 跟踪粒度 | Per-page | Per-bank HR + Global HT（per-page 开销过大） | Per-bank 或 Global |
| 存储开销 | 20 KB | 显著更大（文中未给出精确数字，但明确声称 ABP "significantly smaller"） | 较小（仅需 per-bank timer） |
| 更新机制 | 基于预测正确/错误的 access count 增减 | Saturating counter | Timer value 的固定增减 |

## 实验评估

### 实验设置

- **仿真平台**：论文未明确提及具体模拟器名称（仅 2 页 short paper，实验方法论描述极度精简）。
- **硬件配置**：32 个 DRAM bank（从 predictor 组织可推断）。其他配置细节（核心数、频率、channel 数等）未在正文中给出。
- **Workload**：从 Figure 1 的横轴可识别出使用的 benchmark 为 **PARSEC** suite 中的多个程序：Canneal, Vips, Ferret, Streamcluster, Freqmine, Facesim, Bodytrack, Blackscholes，以及 SPECjbb2005 和 Mix1。
- **对比 baseline**：Open-page, Closed-page, Xu09 [8]。
- **评估指标**：Normalized Throughput。

### 关键结果

从 Figure 1 和论文正文可提取以下关键数据：

- ABP 相比 **open-page** 提升 system throughput **12.3%**。
- ABP 相比 **closed-page** 提升 system throughput **21.6%**。
- ABP 与 **Xu09** 性能基本相当（"achieve the same performance"），但存储开销显著更小。
- ABP 的 predictor history table 平均 hit rate 为 **92.6%**。

### 结果分析

从 Figure 1 可以观察到：

- 在所有 workload 上，ABP 和 Xu09 的性能非常接近，验证了 one-level 结构在保持 per-page 粒度的前提下不牺牲预测精度。
- Closed-page 在大多数 workload 上优于 open-page，印证了 many-core 场景下 locality 确实被严重稀释的动机。
- 不同 workload 间性能差异较大（如 Canneal 表现突出），但 ABP 在各 workload 上的表现较为稳定（"low variability across workloads"）。
- 论文未提供 sensitivity analysis（如不同核数、不同 bank 数、不同 predictor table 大小的影响）。

## 审稿人视角

### 优点

1. **洞察清晰且直觉合理**：access count 预测 vs. timer 预测的核心论点非常有说服力——预测一个小整数（access count 通常是个位数）比预测一个跨数百 cycle 的 timer value 天然更容易。这个 insight 虽然朴素，但具有很强的实践指导意义。

2. **硬件友好的设计**：20 KB 的存储开销在 memory controller 中完全可接受，且 predictor lookup 不在 critical path 上（hidden behind RAS + CAS latency），这让方案具备了实际部署的可行性。One-level 结构相比 Xu09 的 two-level 结构大幅简化了硬件复杂度。

3. **反馈更新机制设计巧妙**：对"预测偏大"、"预测完美"、"预测过早"三种情况分别处理，特别是对 premature closure 的累计恢复机制（重新 open 后累计 access count 再更新）避免了振荡，体现了对实际 access pattern 行为的理解。

### 不足

1. **实验评估严重不足**：作为一篇即便是 short paper 的工作，实验部分的缺失令人担忧。没有给出仿真平台、核心数配置、DRAM timing 参数、address mapping 策略等关键信息，实验结果的 reproducibility 基本为零。不清楚所谓的 "multi-core setting" 到底是几核——标题声称 "Many-Core Era" 但可能只测了 4-8 核。

2. **缺乏 sensitivity analysis**：predictor table 大小（64-set/4-way per bank）的选择缺乏论证。为什么不是 32-set 或 128-set？不同 associativity 的影响如何？不同核数下 interleaving 程度变化时 ABP 的表现如何退化？这些都是 reviewer 会追问的核心问题。

3. **与 timer-based 方案缺乏直接对比**：论文声称 access-based 优于 timer-based，但实验中只对比了 open/closed/Xu09，并没有实现任何一个 timer-based hybrid policy 作为 baseline。Rokicki [2]、Kahn [3]、Sander [4] 等专利方案以及 Park [6]、Stankovic [7] 的学术方案都只在 related work 中提及而未实际对比。

4. **功耗分析完全缺失**：row-buffer management 直接影响 DRAM activate/precharge 次数，进而影响功耗。论文在 Introduction 中提到了 "power optimizations" 但通篇没有任何功耗数据。对于一个 row-buffer 策略的工作，这是一个明显的遗漏。

5. **Predictor 的 cold-start 行为和 thrashing 分析缺失**：当 working set 远大于 predictor cache 容量时（8192 entries 对于 many-core 场景可能不够），频繁的 eviction 会导致 predictor 不断 cold-start。92.6% 的 hit rate 是所有 workload 的平均值，但对于 memory-intensive 的 workload，hit rate 可能显著更低，这一点未被讨论。

### 疑问或值得追问的点

- 论文称 predictor 组织为 "2048-set/4-way cache"，同时又说 "each of the 32 DRAM banks having a 64-set 4-way predictor"。64 × 32 = 2048，说明这是 per-bank 独立的 predictor。但如果是 per-bank 独立的，那 per-page 的 tag 匹配是如何实现的？每个 entry 需要存储多少 bit 的 tag + access count？20 KB 的开销计算是否包含 tag 存储？
- Access count 用多少 bit 表示？如果是 saturating counter，上限是多少？对于高 locality 的 workload（如 streaming access），access count 可能很大，是否会溢出？
- "Decrement by 1" 的更新策略在 access count 波动较大时是否收敛过慢？是否考虑过更激进的适应机制（如指数退避）？
- 论文的多核场景中，不同核心之间对同一 bank 的竞争如何影响 predictor 的准确性？是否有 thread-aware 的 predictor 设计空间？
