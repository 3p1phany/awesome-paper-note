---
title: "RBPP: A Row Based DRAM Page Policy for the Many-core Era"
authors: "Xiaowei Shen, Fenglong Song, Haibo Meng, Shuqian An, Zhimin Zhang"
venue: "ICPADS 2014 (IEEE International Conference on Parallel and Distributed Systems)"
year: 2014
---

# RBPP: A Row Based DRAM Page Policy for the Many-core Era

## 基本信息
- **发表**：ICPADS 2014（IEEE International Conference on Parallel and Distributed Systems）
- **单位**：中国科学院计算技术研究所，计算机体系结构国家重点实验室

## 一句话总结
> 通过跟踪每个 bank 的热点 row address 来动态决定 open/close page policy，以低硬件开销适应 many-core 下的 row buffer 局部性衰减。

## 问题与动机

在 single-core 系统中，同一个 core 的连续访存请求大概率命中同一 DRAM row，open-page policy 能充分利用 row buffer locality，带来显著性能收益。但在 many-core 系统中，多个 core 的访存请求在 memory controller 处交织（interleave），导致 row buffer locality 大幅下降——当前 core 打开的 row buffer 很可能被下一个来自其他 core 的请求 conflict miss。

现有方案的不足：

- **Open-page policy (OPP)**：在 many-core 下 locality 大幅降低，保持 row buffer open 反而增加 conflict miss 的 precharge + activate 开销。
- **Close-page policy (CPP)**：虽然避免了 conflict miss penalty，但完全放弃了残余的 locality，每次访问都需要 activate，浪费了 tRCD。
- **Time Based Page Policy (TBPP)** [2][3][4]：基于"miss 请求比 hit 请求到达更晚"的假设，这个假设在 many-core 下并不可靠。
- **Two-Level Predictor Page Policy (TLPPP)** [5][6]：用 branch predictor 风格的两级预测器来预测 open/close，但硬件开销大（每 bank 约 1033 bits）。
- **Access Based Page Policy (ABPP)** [7]：记录每个 page 的连续访问次数来预测，硬件开销大（每 bank 约 1024 bits 的 64-set 4-way cache）。
- **Application-Aware Page Policy (AAPP)** [8][9]：需要修改 OS 的 memory mapping 来隔离不同应用的访存流，实现复杂度高。

## 核心方法

### 关键思路

**Key Observation**：虽然 many-core 下整体 row buffer hit rate 下降，但在任意短时间窗口内，每个 bank 的访存集中在极少数 row 上。论文统计了 8-core 系统上 10 组 SPEC CPU2006 workload，发现在 10,000 DRAM cycles 的窗口内，每个 bank 最热的 8 个 row 占总访问量的平均 73.0%。

**核心 idea**：为每个 bank 维护一个小型的"最常访问 row 寄存器组"（Most Accessed Row Registers, MARR），记录当前时间窗口内的热点 row。当一次访存完成后，如果当前 row 在 MARR 中，则保持 row buffer open（预测后续还会被访问）；否则 close row buffer。

### 技术细节

#### 整体架构

每个 bank 维护以下硬件结构：
1. **一个 n-bit 寄存器**：记录上一次请求的 row address（n = 16，DDR3 规范中 row address 最多 16 bits）。
2. **p 个 (n+2)-bit 寄存器**：即 MARR，每个条目存储一个 row address（n bits）加上一个 2-bit counter。论文推荐 p = 4。

#### MARR 更新算法（Fig. 2 流程）

当一个请求到达某 bank 时：

**Case 1：当前请求的 row address == 上一次请求的 row address（row buffer hit）**
- 查找 MARR：若该 row 已在 MARR 中 → counter 加 1（有上限饱和）
- 若不在 MARR 中 → 将该 row 加入 MARR，counter 初始化为 1；若 MARR 满，用 LRU 替换

**Case 2：当前请求的 row address ≠ 上一次请求的 row address（row buffer miss / conflict）**
- 查找 MARR：若该 row 在 MARR 中 → counter 减 1；若 counter < 0 则从 MARR 中移除
- 若不在 MARR 中 → 不做任何操作

#### Page Policy 决策

在 MARR 更新完成后，检查当前请求的 row address 是否在 MARR 中：
- **在 MARR 中** → open-page policy（保持 row buffer open）
- **不在 MARR 中** → close-page policy（立即 precharge）

#### 硬件开销

总开销 = (p+1) × n + 2p bits。取 p = 4, n = 16：
- (4+1) × 16 + 2 × 4 = 80 + 8 = **88 bits per bank**
- 对比 ABPP 的 1024 bits/bank 和 TLPPP 的 1033 bits/bank，分别减少 91.4% 和 91.5%

#### 设计选择与 Trade-off

- **MARR 条目数 p 的选择**：p 从 2 增加到 8，平均性能提升很小（除了 Group 8）。论文选择 p = 4 作为性价比最优点。这反映了 many-core 下短时间窗口内每个 bank 活跃 row 确实很少。
- **Counter 的设计**：hit 时加 1、miss 时减 1 的非对称更新策略，结合上限饱和和归零移除，实现了一种简单的"置信度"机制——只有持续被命中的 row 才会留在 MARR 中。
- **LRU 替换**：MARR 满时用 LRU 替换，优先淘汰最久未被访问的条目。

### 与现有工作的区别

| 方案 | 预测依据 | 硬件开销 (bits/bank) | 是否需要修改 OS |
|------|---------|---------------------|--------------|
| ABPP [7] | 每个 page 的连续访问计数 | 1024 | 否 |
| TLPPP [5][6] | Branch predictor 风格的 pattern history | 1033 | 否 |
| AAPP [8][9] | 应用级别的 memory intensity 分类 | 较小 | **是** |
| **RBPP** | **热点 row address 跟踪** | **88** | **否** |

RBPP 的核心差异在于它不做 per-page 粒度的预测（开销大），也不依赖 OS 层面的信息，而是直接跟踪 per-bank 的少量热点 row address，用极小的硬件结构实现近似的预测效果。

## 实验评估

### 实验设置

- **仿真平台**：Gem5 [11] + DRAMSim2 [12]，Gem5 配置为 SE mode
- **硬件配置**：
  - CPU：2.0 GHz，8 cores
  - L1 cache（per core）：32KB I-cache / 64KB D-cache，2-way，64B line，LRU
  - L2 cache（shared）：2MB，8-way，64B line
  - Memory：4GB，1 channel，1 rank/channel，8 banks/rank，DDR3-1333 (10-10-10)
  - DRAM 参数来自 Micron MT41K1G4RA-15E datasheet
- **Workload**：从 SPEC CPU2006 中随机选取 8 个 benchmark 组成一组，共 10 组（详见 Table I）
- **仿真方法**：前 100M instructions（最快 core）用于 warm-up cache 和 MC policy，之后运行直到某个 benchmark 完成 200M instructions，统计 warm-up 后每个 core 的第二个 100M instructions
- **对比 baseline**：OPP、CPP、ABPP、TLPPP

### 关键结果

1. **平均访存延迟**：RBPP 比 OPP 降低 14.7%，比 CPP 降低 4.0%（平均值）。
2. **最大延迟降低**：RBPP 相对 OPP 最大降低 17.4%，相对 CPP 最大降低 12.3%。
3. **与先进方案对比**：RBPP 的平均延迟略优于 ABPP 和 TLPPP（"a little less"）。
4. **硬件开销**：RBPP 仅需 88 bits/bank，比 ABPP（1024 bits）减少 91.4%，比 TLPPP（1033 bits）减少 91.5%。
5. **MARR 条目数敏感性**：p 从 2 增加到 8，平均性能提升很小，p = 4 是较优选择。

### 结果分析

- 10 组 workload 的 row buffer hit rate（open-page 下）从 6.3% 到 35.5% 不等，变化范围很大。但无论 locality 高低，RBPP 始终优于 OPP 和 CPP，说明该策略具有一定的 robustness。
- RBPP 与 ABPP、TLPPP 在性能上差距很小（论文描述为"a little less"），但硬件开销降低了一个数量级，这是 RBPP 的主要卖点。
- 论文没有提供 AAPP 的性能对比数据，仅从"无需修改 OS"的角度进行了定性比较。

## 审稿人视角

### 优点

1. **问题定义清晰，motivation 合理**：many-core 下 row buffer locality 下降是一个真实且重要的问题，论文通过 Fig. 1 的热点 row 统计给出了直观的 key observation。
2. **硬件开销极低**：88 bits/bank 的开销确实非常小，比 ABPP/TLPPP 低一个数量级，这对工程实现非常友好，是论文最大的卖点。
3. **方案简洁易懂**：MARR + counter 的设计逻辑清晰，实现复杂度低，容易在实际 memory controller 中部署。

### 不足

1. **实验规模偏小且配置单一**：
   - 只评估了 1 channel / 1 rank / 8 banks 的配置，在实际 many-core 系统中通常有多 channel 多 rank，交织模式更复杂。
   - DDR3-1333 在 2014 年已经不算新，且缺乏对不同 DRAM 频率/timing 的 sensitivity analysis。
   - 8 cores 称为"many-core"有些勉强，通常 many-core 指 16+ 甚至 64+ cores。

2. **Workload 选择和仿真方法论存在问题**：
   - 所有 workload 都来自 SPEC CPU2006 的单线程 benchmark，通过多份拷贝模拟多核。缺少真正的多线程 workload（如 PARSEC、SPLASH-2）和 memory-intensive 的 streaming workload（如 STREAM）。
   - 用 SE mode 而非 FS mode 仿真，可能无法准确捕捉 OS 层面的 memory 行为。
   - 只运行 200M instructions，对于 SPEC CPU2006 来说 coverage 不足，可能无法代表 benchmark 的 steady-state 行为。SimPoint 或 representative phase 的选择更为严谨。

3. **性能提升的归因分析不足**：
   - 论文没有分析不同 workload 组性能差异的原因。为什么有些组提升大、有些组提升小？与 workload 的 memory intensity、row buffer hit rate 之间的相关性未被讨论。
   - 缺少 row buffer hit/miss/conflict 的分解统计，无法理解 RBPP 具体在哪些情况下做出了正确/错误的预测。

4. **与 ABPP/TLPPP 的对比不够 convincing**：
   - 论文声称 RBPP 性能"略优于" ABPP 和 TLPPP，但从 Fig. 4 看差异非常微小。在没有误差条（error bar）或统计显著性分析的情况下，这个结论不够可靠。
   - ABPP 和 TLPPP 的实现细节（如 ABPP 的 set/way 配置、TLPPP 的 history register 长度）是否经过充分调优？如果 baseline 未被公平优化，对比意义有限。

5. **Counter 设计细节缺失**：
   - Counter 的上限值是多少？2-bit counter 意味着上限为 3？这个参数对性能的影响完全没有分析。
   - Row hit 时 +1、row miss 时 -1 的非对称/对称更新策略的 sensitivity 也未被探讨。

6. **缺少对 bandwidth 和 throughput 的评估**：只看了 average access latency，没有评估 IPC、system throughput、bandwidth utilization 等更全面的系统级指标。DRAM page policy 对 bank-level parallelism 和 command scheduling 的影响也未被讨论。

7. **Writing quality 有待提升**：论文中有一些语法错误和表述不够精确的地方（如"not improved at the same space"应为"same pace"），Section VI 的 conclusion 与 abstract 高度重复。

### 疑问或值得追问的点

- MARR 中使用 LRU 替换策略的依据是什么？是否考虑过基于 counter 值的替换（evict counter 最小的条目）？后者可能更合理。
- Row address 匹配需要 16-bit 全匹配，这在 4 个条目下是可接受的。但如果要扩展到更多条目（或更大的 row address space），CAM 的面积开销是否还能忽略？
- 论文中"10,000 DRAM cycles"的观察窗口是怎么确定的？这个窗口大小对 key observation（73% 集中度）是否敏感？
- RBPP 的 counter 更新逻辑与 page policy 决策是否在关键路径上？在高频 memory controller 中能否满足时序要求？
- 如果将 RBPP 与 FR-FCFS 等 command scheduling policy 结合，交互效果如何？论文完全没有讨论 scheduling policy 的影响。
