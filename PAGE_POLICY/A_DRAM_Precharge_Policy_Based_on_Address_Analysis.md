---
title: "A DRAM Precharge Policy Based on Address Analysis"
authors: "Chiyuan Ma, Shuming Chen"
venue: "DSD 2007 (10th Euromicro Conference on Digital System Design Architectures, Methods and Tools)"
year: 2007
---

# A DRAM Precharge Policy Based on Address Analysis

## 基本信息

- **发表**：DSD 2007（10th Euromicro Conference on Digital System Design Architectures, Methods and Tools）
- **单位**：国防科技大学计算机学院

## 一句话总结

> 通过检查 memory controller waiting queue 中的待访问地址，结合 stride 预测，动态决定 bank 的 precharge 时机。

## 问题与动机

**核心问题**：DRAM precharge policy（open page vs. close page）的静态选择无法适应动态变化的访存流，导致性能损失。

**为什么重要**：处理器与 DRAM 之间的速度差距持续扩大，论文引用数据指出处理器可能有超过一半的时间在等待 DRAM。Precharge policy 直接影响每次 DRAM 访问落入 page hit、page empty 还是 page conflict miss，而三者的延迟差异显著（在论文配置下分别为 6、10、14 cycles），因此 precharge 决策对整体性能至关重要。

**现有工作不足**：

- **Close page policy**：每次访问后立即 precharge，所有访问均为 page empty，放弃了利用 page buffer locality 的机会。
- **Open page policy**：访问后保持 page open，能获取 page hit，但在 page conflict miss 时额外付出 precharge latency，性能不可预测。
- **基于历史的动态策略**（如 Stankovic 的 complete predictor [2]、Huan 的 processor directed policy [3]、Alpha 21174 的 history-based policy [4]）：依赖访问历史进行预测，但历史信息在不同应用上效果不稳定，本质上仍是"猜测"而非"确定性判断"。

**关键洞察**：由于处理器与 DRAM 的速度差距，memory controller 的 waiting queue 中通常积压了多条待访问指令。这意味着在当前访问完成时，下一次对同一 bank 的访问请求很可能已经在队列中——可以直接检查而非预测。

## 核心方法

### 关键思路

论文的核心 idea 是：**用 waiting queue 中已知的未来访问地址来做确定性的 precharge 决策，而非基于历史的概率预测**。当 waiting queue 中找不到对当前 bank 的后续请求时，退而使用 stride-based 地址外推来猜测下一次访问是否仍在当前 page 内。

Key observation 有三个：
1. 处理器-DRAM 速度差距导致 waiting queue 中经常有多条排队请求，使得"看未来"成为可能；
2. 同 page 内的连续访问地址往往呈递增/递减的 stride pattern（stride 通常等于 cacheline size），这为 fallback 预测提供了依据；
3. 约 57% 的连续 DRAM 访问落入不同 bank，提供了 precharge latency hiding 的机会。

### 技术细节

#### 整体架构

DRAM controller 包含四个核心组件：

1. **Waiting Queue**：缓存所有等待访问 DRAM 的 load/store 指令及其地址。
2. **Search Engine**：持续顺序扫描 waiting queue，将每条指令的 page address 分发到对应 bank 的 bank controller。工作条件：目标 PAS 未满且 waiting queue 非空。
3. **Bank Controller（×8）**：每个 bank 一个，包含：
   - **BSR（Bank Status Register）**：记录 bank 状态（idle/active）、上一次访问的 page address、stride 值（当前地址 - 上一次地址）。
   - **PAS（Page Access Statistics Queue）**：深度为 4 的队列，每个 entry 包含 valid bit、page address、counter。
4. **Interface Logic**：与 DRAM 的物理接口。

#### PAS 队列的工作机制

Search engine 将 page address 送入对应 bank 的 PAS：
- 若该地址与 PAS 中上一个 entry 的 page address 相同 → 上一个 entry 的 counter + 1（表示同 page 的连续访问次数）；
- 若不同 → 创建新 entry，valid = 1，counter = 0。

这样 PAS 实际上是对未来访问的一个"run-length encoding"式的摘要。

#### Precharge 决策流程（Figure 6 的流程图）

当一次 DRAM access 完成后：

1. **检查 PAS 首项的 counter**：
   - counter ≠ 0 → 当前 page 马上还会被访问 → **keep page open**，counter - 1；
   - counter = 0 → 进入下一步；
2. **检查 PAS 是否还有其他 valid entry**：
   - 有 → 下一次访问将去不同 page → **precharge bank**，删除当前 PAS entry；
   - 没有（PAS 为空，说明 waiting queue 中没有对此 bank 的后续请求）→ 进入 fallback；
3. **Stride-based fallback 预测**：
   - 计算 `current_address + stride`，判断结果是否仍落在当前 page 内；
   - 是 → **keep page open**；
   - 否 → **precharge bank**。

#### 设计权衡

- **PAS 深度选择（4 entries）**：深度过大增加面积和 search engine 延迟，深度过小则无法捕捉较远的未来访问。论文选择 4，但未给出 sensitivity analysis。
- **确定性 vs. 推测性**：当 PAS 有信息时做确定性决策，PAS 为空时退化为 stride-based 推测——这是一种 graceful degradation 设计。
- **硬件开销**：每个 bank controller 增加一个 4-entry PAS 和扩展的 BSR（加 stride 字段），外加一个共享的 search engine 和 adder。论文声称开销 trivial。

### 与现有工作的区别

| 方法 | 决策依据 | 本质 | 局限 |
|------|---------|------|------|
| Open/Close Page | 静态策略 | 无自适应能力 | 无法适配动态 workload |
| Stankovic [2] complete predictor | 历史访问模式 | 概率预测 | 历史不代表未来 |
| Huan [3] processor directed | 处理器提供的未来访存行为 | 需要 ISA/compiler 支持 | 侵入性强 |
| **本文** | waiting queue 中的实际地址 + stride fallback | 确定性判断 + 推测 | 依赖队列深度和访存压力 |

关键差异：本文直接利用 memory controller 内部已有的 waiting queue 信息，不依赖历史、不需要 ISA 修改，是一种纯 controller 端的微架构优化。

## 实验评估

### 实验设置

- **仿真平台**：SimpleScalar 3.0，修改了 cache 和 memory 模块
- **处理器配置**：
  - 频率 2GHz
  - Instruction window = 4
  - Out-of-order issue
  - L1 I-Cache: 16KB, 32B block, 4-way
  - L1 D-Cache: 16KB, 32B block, 4-way
  - L2 Cache: 128KB, 32B block, 4-way
- **DRAM 配置**：
  - 频率 200MHz（processor:DRAM = 10:1）
  - 8 banks
  - Active latency: 4 DRAM cycles
  - Access latency: 6 DRAM cycles
  - Precharge latency: 4 DRAM cycles
  - Page size: 8KB（由文中 cacheline/page 计算推断）
- **Workload**：SPEC CPU2000 中的 9 个 benchmark（ammp, art, bzip2, equake, gcc, gzip, mcf, mesa, twolf），每个 fast-forward 1B 指令后模拟 1B 指令
- **对比 baseline**：close page policy、open page policy

### 关键结果

1. **整体 CPI 降低**：相比 close page 平均降低 15.7%，相比 open page 平均降低 4.3%。
2. **Memory 指令 CPI 降低**：相比 close page 平均降低 27.3%，相比 open page 平均降低 6.2%。
3. **最佳 case（ammp）**：CPI 相比 close page 降低 59%，相比 open page 降低 10.7%，因为 ammp 访存密集且地址呈强 stride 规律。
4. **Access type 分布变化**：相比 open page，本方法维持了几乎相同的 hit rate，但大幅降低了 conflict miss rate，将其转化为 empty access（empty 延迟 10 cycles < conflict miss 延迟 14 cycles）。

### 结果分析

**效果好的场景**：访存密集型程序（ammp、art），队列中积压请求多，PAS 能提供更多确定性信息；地址呈 stride pattern 明显的程序，fallback 预测准确率高。

**效果有限的场景**：mcf 访存少且 locality 差，三种策略 CPI 差异不大。

**性能上限分析**：论文承认即使方法能大幅降低 conflict miss rate，整体提升仍受限于两个因素：(1) 连续访问同 bank 不同 page 时 precharge latency 无法隐藏；(2) 只有约 57~60% 的连续访问去往不同 bank，才有 precharge hiding 的机会。

论文未做 sensitivity analysis（如 PAS depth、不同 DRAM timing、不同 cache 配置的影响）。

## 审稿人视角

### 优点

1. **思路简洁且直觉合理**：利用 waiting queue 中已有的信息做确定性决策，idea 自然，实现简单，hardwired 逻辑开销小。
2. **两级决策的 graceful degradation**：PAS 有信息时做确定性判断，无信息时用 stride 推测，设计上有层次感。
3. **对访存特征的分析到位**：Section 3 对地址 stride 规律、bank 分布比例的统计分析为方法设计提供了 workload characterization 基础。

### 不足

1. **实验配置严重过时且不典型**：L2 Cache 仅 128KB（即使在 2007 年也偏小），且无 L3，导致 DRAM 访问压力人为偏高，可能夸大了方法的收益。Instruction window = 4 也远小于同期处理器（如 Alpha 21264 的 80-entry ROB），极大限制了 MLP（Memory-Level Parallelism）。
2. **缺乏 sensitivity analysis**：PAS 深度、DRAM timing 参数、cache 大小、bank 数量等关键参数均未做敏感性分析，无法判断方法的 robustness。
3. **DRAM 模型过于简化**：未考虑 rank、channel、refresh、tFAW/tRRD 等现实约束，8 bank 单 channel 的配置在建模精度上存疑。
4. **Benchmark 覆盖不足且方法论有问题**：仅 9 个 SPEC CPU2000 程序，无多线程 workload，无 multi-programmed workload。论文声称该方法在高访存压力下效果好，却没有用真正 memory-intensive 的 multi-core/multi-thread 场景验证。
5. **与 FR-FCFS 等调度策略的关系未讨论**：论文只讨论了 precharge policy，但完全没有涉及与 command scheduling policy（如 FR-FCFS）的交互。实际 controller 中 precharge 决策与调度策略是紧耦合的，单独讨论 precharge policy 的实际意义有限。
6. **Stride-based fallback 过于简单**：仅使用单步 stride 外推，对非线性访问模式（如 linked list traversal、hash table lookup）基本无效，而这恰恰是 open page policy 表现最差的场景。
7. **Search engine 的时序开销未分析**：Search engine 需要顺序扫描整个 waiting queue 并更新 PAS，其延迟是否会在 critical path 上影响 precharge 决策的及时性，论文未讨论。
8. **论文写作质量一般**：发表在 DSD（非体系结构顶会），related work 覆盖面窄，缺少对 Rixner (ISCA 2000) 等经典 DRAM scheduling 工作的引用和对比。

### 疑问或值得追问的点

- PAS 深度为 4 的选择依据是什么？是否做过 sweep？深度为 2 或 8 时性能变化如何？
- 当 waiting queue 很浅（如低访存压力程序）时，PAS 大概率为空，方法退化为纯 stride 预测——此时与简单的 stride-based predictor 相比有多少额外收益？
- 论文提到方法在 in-order access 条件下工作，对 out-of-order access "needs further discussion"——但现代 memory controller 几乎都做 request reordering（如 FR-FCFS），在这种场景下 PAS 的语义是否还成立？
- Search engine 的扫描与 PAS 更新是流水线化的还是串行的？在高访存压力下是否会成为 bottleneck？
