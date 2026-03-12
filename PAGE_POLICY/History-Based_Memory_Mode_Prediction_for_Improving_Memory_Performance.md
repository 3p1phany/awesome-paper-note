---
title: "History-Based Memory Mode Prediction for Improving Memory Performance"
authors: "Seong-Il Park, In-Cheol Park"
venue: "IEEE ISCAS 2003"
year: 2003
---

# History-Based Memory Mode Prediction for Improving Memory Performance

## 基本信息
- **发表**：IEEE International Symposium on Circuits and Systems (ISCAS), 2003
- **作者单位**：KAIST, Department of EECS

## 一句话总结
> 利用 per-bank/per-row 两位饱和计数器记录 page hit/miss 历史，预测下一次访问是否 page hit，从而决定 bank 保持 row-active 还是 auto-precharge 回到 idle。

## 问题与动机

在 SoC 系统中，off-chip SDRAM 是主要的性能瓶颈。SDRAM 的 bank 存在两个稳定状态——idle 和 row-active。当 bank 处于 row-active 状态时，若后续访问命中同一 row（page hit），只需付出 column access latency（t_CL）；若未命中（page miss），则需要额外的 precharge + row-activation 开销（t_RP + t_RCD + t_CL），代价最大。因此，memory controller 需要在每次访问完成后决定：是让 bank 保持 row-active（赌下一次是 page hit），还是立刻 auto-precharge 回 idle（赌下一次是 page miss 或访问其他 row）。

**现有工作的不足：**

1. **静态调度方法** [10]-[13]：通过编译时或设计时静态安排访存地址序列来减少 page miss，仅适用于图像/视频处理等访存模式规律且可提前预知的应用，无法处理通用计算中不规则的访存模式。
2. **动态模式控制 [14]（Miura et al., ISLPED 2001）**：在 page hit 时将 SDRAM 从 idle 切换到 row-active，并在连续 page miss 次数超过阈值时才切回 idle。该方法在 in-row 访问占主导时有效，但当访存模式不规则、in-row 访问不占主导时，频繁的模式切换带来大量 precharge/row-activation 开销。其本质问题是：它仅根据"当前连续 miss 次数"做反应式决策，缺乏对历史模式的学习能力。

## 核心方法

### 关键思路

借鉴分支预测中经典的两位饱和计数器思想：为每个 bank（或每个 row）维护一个两位状态机，根据 page hit/miss 的历史记录来预测下一次访问是否为 page hit，进而决定 bank 应保持 row-active 还是通过 auto-precharge 转入 idle 状态。核心 observation 是：内存访问的 page hit/miss 行为具有时间局部性，历史上频繁 hit 的 bank/row 未来也倾向于 hit。

### 技术细节

**1. 两位饱和计数器状态机**

每个计数器有四个状态：
- **SH (Strongly Hit, 11)**：强命中
- **WH (Weakly Hit, 10)**：弱命中
- **WM (Weakly Miss, 01)**：弱未命中
- **SM (Strongly Miss, 00)**：强未命中

状态转移规则：
- Page hit → 计数器递增（向 SH 方向移动）
- Page miss → 计数器递减（向 SM 方向移动）

**2. 模式控制决策**

- 当计数器处于 SH 或 WH 状态时：发出不带 auto-precharge 的 read/write 命令 → bank 保持 row-active
- 当计数器处于 SM 或 WM 状态时：发出带 auto-precharge 的 read/write 命令 → bank 回到 idle

**3. Hit/Miss 判定**

通过 Address Registers 保存每个 bank 上一次访问的 row address，将当前访问的 row address 与之比较，判定本次访问是 page hit 还是 page miss，并据此更新对应的状态机。

**4. 两种粒度的预测器**

- **Per-row counter**：每个 row 一个两位计数器。对于 N-bit row address、M 个 bank 的 SDRAM，需要 M × 2^N 个计数器。预测精度最高，但面积开销巨大（例如 13-bit row address + 4 banks = 4 × 8192 = 32768 个两位计数器）。
- **Per-bank counter**：每个 bank 仅一个两位计数器，总共只需 M 个计数器（本文配置下仅 4 个）。面积开销极小，预测精度略有下降但仍优于 baseline [14]。

**5. 控制器架构**

论文 Fig. 7 给出了完整的控制器架构，包含以下模块：
- Address Alignment：地址解析为 bank/row/column
- Hit/Miss Decision：与 Address Registers 中存储的上次 row address 比较
- History FSM：维护两位饱和计数器
- Mode Control：根据 FSM 状态决定是否附带 auto-precharge
- Command Generator + I/O Control：生成最终的 SDRAM 命令

**6. 关键 Trade-off**

Per-row counter 提供更精确的预测（平均 74.2% hit prediction ratio），但面积开销为 O(M × 2^N)；Per-bank counter 将面积降至 O(M)，预测精度降至 69.3%，但在 latency 层面反而因为某些 benchmark 上避免了 per-row 的过拟合而表现接近甚至略优。论文最终推荐 per-bank 方案作为面积-性能的最佳平衡点。

### 与现有工作的区别

| 方案 | 决策依据 | 粒度 | 适用性 |
|------|---------|------|--------|
| Always idle | 无预测，总是 precharge | - | 保守，page hit 时浪费 t_RCD |
| Always active | 无预测，总是保持 row-active | - | 激进，page miss 时付出 t_RP + t_RCD |
| Miura et al. [14] | 连续 page miss 次数超阈值才切换 | Per-bank | 反应式，滞后性强 |
| **本文 (per-bank)** | 两位饱和计数器历史预测 | Per-bank | 预测式，能学习 hit/miss 模式 |
| **本文 (per-row)** | 两位饱和计数器历史预测 | Per-row | 预测最精确，面积开销大 |

与 [14] 的核心区别：[14] 是一个简单的阈值计数器，只看"连续 miss"的长度来做反应式切换；本文引入了类似 branch predictor 的两位饱和计数器，能在 hit 和 miss 交替出现的模式下更稳健地预测，不会因为偶尔一次 miss 就立刻放弃 row-active 状态。

## 实验评估

### 实验设置

- **仿真平台**：Trace-driven simulation，MIPS R3000 simulator 产生 data memory trace
- **SDRAM 配置**：13-bit row address, 9-bit column address, 4 banks; t_RP = 3 cycles, t_RCD = 3 cycles, t_CL = 2 cycles (read) / 0 cycles (write)
- **Workload**：5 个 SPEC92 benchmark 的 data memory trace：008.espresso, 023.eqntott, 056.ear, 085.gcc, 090.hydro2d
- **对比 baseline**：Always idle, Always active, Previous dynamic scheme [14], Per-row counter, Per-bank counter

### 关键结果

1. **Hit prediction ratio**：Per-row counter 平均 74.2%，Per-bank counter 平均 69.3%，均显著优于 [14] 的 61.2%（per-bank 比 [14] 高 8.1 个百分点）。
2. **总 latency（per-bank counter vs. always idle）**：平均降低 19.0%（normalized average 81.0% vs. 100.0%）。
3. **总 latency（per-bank counter vs. [14]）**：normalized average 81.0% vs. 86.8%，进一步降低约 5.8 个百分点。
4. **Per-row counter latency**：normalized average 77.8%，是所有方案中最优的，但面积开销远大于 per-bank。
5. **特殊案例 090.hydro2d**：[14] 的 latency（1035732）甚至劣于 always idle（1087215 → [14] 虽然 latency 数值更小，但比 always active 的 948270 更差），说明 [14] 在该 workload 下的模式切换策略失效；而本文 per-bank（944796）接近 always active 的表现，鲁棒性更好。

### 结果分析

论文的分析较为简略。从数据可以观察到：

- **eqntott** 和 **gcc** 两个 benchmark 上所有 history-based 方案表现最好，说明这两个程序的访存模式具有较强的局部性，page hit 比例本身较高（per-row 达到 86.9% 和 81.3%）。
- **hydro2d** 是最难的 benchmark，所有方案的 hit prediction ratio 都最低（per-row 仅 60.0%）。有趣的是，在这个 benchmark 上 always active 策略反而是最优的（948270），说明该程序的 page miss 代价相对不高、或者 miss 后再次访问同 row 的概率不低，保持 row-active 是更好的策略。
- 论文没有做 sensitivity analysis（如不同 t_RP/t_RCD 参数、不同 row address 位宽、不同 bank 数量的影响），也没有分析具体哪些访存模式导致预测失败。

## 审稿人视角

### 优点

1. **思路清晰，方法简洁有效**：将 branch prediction 中经典的两位饱和计数器迁移到 DRAM page mode 预测，idea 自然且实现成本极低（per-bank 方案仅需 4 个两位计数器）。
2. **提供了两种粒度的设计选择**：Per-row 和 per-bank 方案形成了一个面积-精度的 trade-off spectrum，给设计者提供了灵活性。
3. **对 SoC/嵌入式场景有实际意义**：2003 年的 SoC 场景下，memory controller 的面积预算紧张，per-bank 方案几乎零面积开销却能带来 19% 的 latency 降低，工程价值明确。

### 不足

1. **实验方法论严重过时且规模不足**：
   - 仅使用 SPEC92 的 5 个 benchmark，workload 多样性严重不足。
   - 使用 MIPS R3000 simulator 产生 trace，这是一个非常老旧的单发射顺序执行处理器，无法反映现代 out-of-order 处理器的访存行为。
   - Trace-driven simulation 无法捕捉 memory controller 决策对处理器执行的反馈效应（即 timing 不准确）。

2. **缺乏与 open-page / close-page policy 的系统性对比**：论文实际上在做的就是 adaptive open/close page policy，但没有从这个角度进行讨论。Always idle 就是 close-page policy，always active 就是 open-page policy，本文方案是一种 adaptive policy。这个框架性的认识缺失使得论文的定位不够清晰。

3. **Latency 模型过于简化**：
   - 没有考虑 bus contention、bank conflict、refresh 等实际因素。
   - 假设每次访问独立计算 latency，没有建模 command queue、reordering 等现代 MC 的关键特性。
   - 没有考虑多 requestor（多核或 SoC 中多个 master）的场景，而论文 Fig. 1 明确画出了 SoC 有多个 functional block 共享内存。

4. **Per-bank counter 的局限性未被充分讨论**：Per-bank 粒度意味着同一 bank 内不同 row 的 hit/miss 行为被混在一起。当一个 bank 内同时存在高 locality 和低 locality 的 row 时，per-bank counter 会被"污染"。论文没有分析这种 interference 的影响。

5. **没有功耗分析**：论文在 introduction 提到了 power consumption，但实验部分完全没有功耗数据。对于目标场景（portable wireless devices），功耗是核心关注点，这是一个明显的遗漏。

6. **缺少 sensitivity analysis**：没有对 SDRAM timing 参数、地址配置、counter 位宽（为什么是 2-bit 而不是 3-bit？）等做任何敏感性分析。

### 疑问或值得追问的点

- Per-bank counter 在 Table 2 中 090.hydro2d 上的 latency（944796）竟然略低于 per-row counter（942354）几乎持平，但 hit prediction ratio 却差距明显（59.7% vs. 60.0%）。这说明更高的 prediction accuracy 并不总是带来更低的 latency，可能存在 misprediction penalty 不对称的问题（predict hit but miss 的代价 > predict miss but hit 的代价），论文没有分析这一点。
- 两位饱和计数器的初始状态是 reset 到 SM（从 Fig. 5 和 Fig. 6 看），这意味着冷启动时所有 bank 都倾向于 auto-precharge。对于短时间内频繁访问同一 row 的场景（如 cache line fill），冷启动阶段可能需要多次 hit 才能将计数器"热身"到 SH/WH 状态，初始几次访问会白白付出 precharge + activation 开销。
- 论文的核心 idea 与后来广泛研究的 adaptive page policy 高度相关。从 2003 年的视角看，这个工作有一定的先驱性，但方法本身过于简单，在后续的 DRAM scheduling 研究中很快被更复杂的方案（如 FR-FCFS 中的 page policy 决策）所涵盖。
