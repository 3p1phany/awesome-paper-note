---
title: "Self-Optimizing Memory Controllers: A Reinforcement Learning Approach"
authors: "Engin İpek, Onur Mutlu, José F. Martínez, Rich Caruana"
venue: "ISCA 2008"
year: 2008
---

# Self-Optimizing Memory Controllers: A Reinforcement Learning Approach

## 基本信息
- **发表**：ISCA (International Symposium on Computer Architecture), 2008
- **作者**：Engin İpek (Cornell / MSR), Onur Mutlu (MSR), José F. Martínez (Cornell), Rich Caruana (Cornell)
- **单位**：Cornell University + Microsoft Research

## 一句话总结
> 将 DRAM command scheduling 建模为 reinforcement learning 问题，使 memory controller 能在线学习并自适应优化长期性能。

## 问题与动机

### 问题本质
CMP 时代 off-chip bandwidth 是性能瓶颈，而现有 memory controller 只能利用理论峰值带宽的很小比例。论文以 SCALPARC 为例：FR-FCFS scheduler 的 data bus utilization 仅 46%，而理想 scheduler 可达 85%以上，导致 55% 的性能损失。

### 现有工作的不足
传统 memory controller 设计是 **ad hoc** 的，由人类专家基于经验选取少量属性、制定固定调度策略。这种方式存在两个根本缺陷：

1. **无法做长期规划（no long-term planning）**：无法预判当前调度决策对未来 DRAM cycle 的连锁影响。例如，一个看似浪费的 precharge command 可能为后续更好的 bandwidth utilization 铺路。
2. **无法学习和泛化（no learning & generalization）**：固定策略无法从历史调度经验中学习，也无法适应动态变化的 workload 行为（如 phase change、context switch）。

### 为什么 DRAM scheduling 困难
- DDR2 SDRAM 有超过 50 种 timing constraints（local per-bank 如 tRCD，global across-bank 如 tWTR）
- 调度决策具有长期后果：发出一条命令会因 timing constraints 阻塞后续命令，同时影响各 core 的 instruction window unblocking，进而改变未来的 memory reference stream
- 最终收益取决于处理器未来行为，而这不在 scheduler 控制范围内

## 核心方法

### 关键思路
将 memory controller 设计为一个 **reinforcement learning agent**，它通过与系统的持续交互，自动学习最优的 DRAM command scheduling policy。核心 insight 是：DRAM scheduling 天然可以建模为一个 **infinite-horizon discounted MDP**——scheduler 在每个 DRAM cycle 观察系统状态、执行动作（发出 DRAM command）、收到 reward（是否利用了 data bus），目标是最大化长期累积 reward（即长期 data bus utilization）。

### 技术细节

#### 1. MDP 建模

**Reward 设计**：
- 发出 Read 或 Write command（实际占用 data bus）→ reward = 1
- 发出 Precharge / Activate / NOP → reward = 0
- 关键：reward = 0 不意味着 scheduler 不会选择这些命令；scheduler 优化的是 **长期累积 reward**，如果 precharge 能带来未来更好的 bus utilization，它会学会选择 precharge。

**State 表示（6 个属性，从 226 个候选中通过 feature selection 选出）**：
1. Transaction queue 中的 read 数量（load/store miss）
2. Transaction queue 中的 write 数量（writeback）
3. Transaction queue 中属于 load miss 的 read 数量
4. 对于 load miss 请求，该 load 在其 core 动态指令流中相对于其他 load 的顺序（通过 sequence number 确定，类似 Alpha 21264 的 inum）
5. Transaction queue 中等待当前命令所引用 row 的 write 数量
6. Transaction queue 中等待当前命令所引用 row 的 load miss 数量（且这些 load miss 在各自 core 中具有最老的 sequence number）

**设计意图**：
- 属性 1-2：帮助学习 read/write balance，避免 write buffer stall
- 属性 3：检测低并发状态（多 core 因 load miss 阻塞），学会优先处理 load miss
- 属性 4：学习 load miss 间的优先级
- 属性 5：学习 write burst 以摊销 tWTR 开销
- 属性 6：近似 critical request 数量，open 一个有多个 critical request 的 row 可同时 unblock 多个 core

**Action 空间（6 种）**：
1. Precharge
2. Activate
3. Write
4. Read (满足 load miss)
5. Read (满足 store miss)
6. NOP（用于在无合法命令时更新 Q-value）

区分 load miss read 和 store miss read 允许 scheduler 学会在有利时优先处理 load miss。

#### 2. Q-Learning 算法：SARSA Update

Q-value 更新采用 **SARSA** (on-policy TD learning)：

```
Q(s_prev, a_prev) ← (1 - α) · Q(s_prev, a_prev) + α · [r + γ · Q(s_current, a_current)]
```

- α = 0.1（learning rate）
- γ = 0.95（discount factor）
- 初始化：所有 Q-value 乐观初始化为 1/(1-γ) = 20，鼓励早期充分探索

每个 DRAM cycle 的操作流程（Algorithm 1）：
1. 发出上一 cycle 选中的命令
2. 观察 reward
3. 获取当前所有合法命令
4. 以概率 ε = 0.05 随机选命令（exploration），否则选 Q-value 最大的命令（exploitation）
5. 用 SARSA rule 更新上一 cycle 的 Q-value

#### 3. CMAC 函数逼近：解决 State Space 爆炸

直接用 table 存储所有 state-action pair 的 Q-value 不可行（>10 亿个）。论文采用 **CMAC (Cerebellar Model Articulation Controller)** 解决泛化问题：

- 多个 coarse-grain Q-value table，彼此以随机常数偏移
- 给定 state，从所有 table 中读取对应 entry 并求和得到 Q-value 估计
- **自适应分辨率**：邻近状态点在多数 table 中共享 entry（强泛化），远离的点仅在少数 table 中共享（弱泛化，可区分）
- **Hashing**：将 state attributes 通过 hash function 映射到 CMAC index，使存储不再随属性数指数增长
- Hash 冲突类似于 branch predictor 中的 destructive aliasing，但影响有限（scheduler 在任何时刻只在 state space 的局部小区域活动）

#### 4. 硬件实现

**5-stage pipeline**（运行在 CPU 频率，非 DRAM 频率）：
- Stage 1：获取合法命令 + state attributes，生成 state-action pair
- Stage 2：用 state-action pair 索引 CMAC arrays（index = 高位 attribute 拼接 → XOR with action-dependent constant → hash）
- Stage 3-4：求和各 CMAC array 的值，得到 Q-value estimate
- Stage 5：比较 Q-values，选最大值

**吞吐量**：2-wide pipeline，每 CPU cycle 处理 2 个 Q-value。对 4GHz CPU + DDR2-800 (400MHz DRAM)，每 DRAM cycle 可 clock 10 次 → 可评估最多 12 个候选命令（经验表明 64-entry queue 中合法命令极少超过此值）。

**硬件开销**：
- 8192 个 Q-value × 16-bit fixed point × 2 pipe ways = **32KB SRAM**
- 每个 CMAC array：256 entries，1R1W port，共 32 arrays per pipe way
- 一个 pipelined 16-bit fixed-point multiplier（用于 SARSA update，不在决策关键路径上）
- 少量 counter 用于计算 state attributes

#### 5. 正确性保障

三项机制防止 livelock：
1. 有合法命令时不允许选 NOP
2. 只能 activate 有 pending request 的 row（不能随意 activate）
3. 新 activated 的 row 必须先发出至少一条 read/write 才能 precharge

Anti-starvation：pending 超过 10,000 cycle 的 request 强制优先完成。
Refresh：RL scheduler 不控制 refresh，refresh interval 到期时暂停 RL、执行 refresh、再恢复。

#### 6. Feature Selection 方法

从 226 个候选属性中，使用 **forward stepwise feature selection**：
- 第 1 轮：分别评估 226 个单属性 RL scheduler，选出最佳单属性
- 第 2 轮：在最佳单属性基础上，分别加入剩余 225 个属性，选最佳二属性组合
- 重复 N 轮，最终选出 6 个属性
- 训练用 benchmark：MG, FFT, RADIX, SWIM（仿真最快的 4 个）

### 与现有工作的区别

| 对比项 | FR-FCFS (Rixner et al., ISCA 2000) | Adaptive History-based (Hur & Lin, MICRO 2004) | **本文 RL-based** |
|--------|--------------------------------------|------------------------------------------------|------------------|
| 策略类型 | 固定优先级（CAS > RAS, older > younger） | 在多个预设策略间切换 | 在线学习，策略空间远大于人工设计 |
| 长期规划 | 无 | 无 | 有（通过 discount factor γ 实现） |
| 自适应性 | 无 | 有限（仅切换预设策略） | 完全自适应（SARSA 持续更新） |
| 泛化能力 | 无 | 无 | 有（CMAC 函数逼近） |
| 考虑命令类型 | 仅 CAS vs RAS | 仅 read vs write | 区分 precharge/activate/write/read(load)/read(store)/NOP |

与 Fair Queueing (Nesbit et al., MICRO 2006) 的区别：FQ 仅解决 fairness 问题（平均仅 2% 提升），而 RL 直接优化长期系统性能（19% 提升）。

## 实验评估

### 实验设置

- **仿真平台**：SESC simulator（heavily modified version）
- **硬件配置**：
  - 4 core, 2-way SMT per core (共 8 hardware threads)
  - 4GHz, 4-wide issue, 96-entry ROB
  - 4MB shared L2 (8-way, 64B block, 32-cycle uncontended latency)
  - DDR2-800: 400MHz DRAM bus, 6.4GB/s peak, single rank, 4 chips/rank, 4 banks/chip
  - 64-entry transaction queue, open page policy, page interleaving
  - 关键 timing: tRCD=5, tCL=5, tWL=4, tWTR=3, tRP=5, tRAS=18, tRC=22, BL=8
- **Workload**：9 个 parallel applications (8 threads each)
  - SPLASH-2: OCEAN, FFT, RADIX
  - SPEC OpenMP: SWIM-OMP, EQUAKE-OMP, ART-OMP
  - NAS OpenMP: MG, CG
  - Nu-MineBench: SCALPARC
  - 均使用 gcc/Fortran -O3 编译，run to completion
- **对比 baseline**：
  1. In-order scheduler
  2. FR-FCFS (state-of-the-art baseline)
  3. Optimistic scheduler（移除所有 timing constraints 除 CAS-to-CAS，作为 upper bound）

### 关键结果

1. **性能提升**：RL-based controller 在 4-core 单通道配置下，平均比 FR-FCFS 提升 **19%**（最高 33% on SCALPARC，最低 7% on FFT），达到了 optimistic scheduler 所能提供的性能提升的 **27%**。

2. **带宽利用率**：DRAM data bus utilization 从 FR-FCFS 的 46% 提升到 RL 的 **56%**（提升 22%），而 optimistic 上界为 80%。

3. **等效带宽节省**：单通道 RL 系统的性能提升量是双通道 FR-FCFS 系统性能提升量的 **50%**（19% vs 39%），即 RL 能以更低成本达到近似于加倍带宽的一半效果。双通道 + RL 进一步提升至 58%。

4. **多控制器可扩展性**：在 8-core/2-channel 和 16-core/4-channel 配置下，RL 分别比 FR-FCFS 提升 15% 和 14%，多个独立 RL agent 无需显式协调即可成功收敛。

### 结果分析

**Queue Occupancy 与 Miss Penalty 的良性循环**：
RL scheduler 的 transaction queue 平均 occupancy 为 28（FR-FCFS 仅 10），平均 L2 load miss penalty 从 824 cycle 降至 562 cycle。这形成良性循环：更好的调度 → 更低 miss penalty → core 更快 unblock → 更多 request 进入 queue → 更高 bank-level parallelism 和 row buffer locality → 更好的调度。

**额外 state information 的贡献**：
将 RL 使用的额外信息（read/write 区分、load/store 区分、sequence number）加入 FR-FCFS 形成 "Family-BEST" 策略，仅提升 5%。RL 的 19% 提升中，大部分来自 CMAC 的高表达力 + 在线自适应学习，而非单纯的额外信息。

**Online vs Offline RL**：
Offline RL（在所有 benchmark 训练数据上训练后固化策略）仅提升 8%，远低于 online RL 的 19%。原因：(1) online RL 可适应 phase change；(2) offline RL 在 non-stationary environment 下难以找到好策略。

**参数敏感性**：
- γ：γ=0 时仅优化即时 reward，性能差；γ=1 时 Q-value 不收敛；γ=0.95 最优
- ε：较小值（0-0.05）效果好；ε=1（纯随机）性能严重下降；ε=0 对当前 benchmark 无害，但可能不利于应对 context switch

## 审稿人视角

### 优点

1. **问题建模优雅且深刻**：将 DRAM scheduling 形式化为 MDP 是一个非常自然且有说服力的抽象。论文清晰地识别了传统 scheduler 缺乏 long-term planning 和 learning 能力这一根本局限，并用 RL 框架系统性地解决。这一思路对后续 memory system 和 microarchitecture 中的 ML 应用研究产生了深远影响。

2. **硬件实现完整且现实**：不仅提出算法，还给出了完整的 5-stage pipeline 硬件设计、32KB SRAM 的合理开销估算、以及 timing 可行性分析（12 commands/DRAM cycle 的吞吐量足够覆盖 64-entry queue）。正确性保障（anti-livelock、anti-starvation、refresh handling）的讨论也很充分。

3. **实验分析系统且有洞察力**：从多个角度拆解性能来源——额外 state information 的贡献 vs RL 的贡献（Section 5.1.2）、online vs offline 的对比（Section 5.1.3）、参数敏感性（Section 5.1.4）、多控制器扩展性（Section 5.2）、带宽效率分析（Section 5.3）。Queue occupancy 与 miss penalty 的良性循环分析尤其有说服力。

4. **Feature selection 方法论严谨**：从 226 个候选属性中通过 forward stepwise selection 自动筛选出 6 个属性，避免了人工挑选的主观性，也为 RL 在 hardware design 中的应用提供了方法论参考。

### 不足

1. **Workload 的代表性和多样性不足**：
   - 仅使用 9 个 parallel application，且均为科学计算/数据挖掘类。缺少 multiprogrammed workload（多个独立单线程程序混合执行），而这恰恰是 CMP 上更常见、scheduling 更困难的场景。
   - 缺少 memory-intensive 和 compute-intensive 混合的场景分析。
   - 所有程序都 run to completion，未测试 context switch 场景下 RL 的适应能力（尽管论文声称 ε-greedy 能处理）。

2. **Optimistic scheduler 作为 upper bound 的定义有争议**：
   - 该 upper bound 移除了除 CAS-to-CAS 以外的所有 timing constraints，这在物理上完全不可实现。RL 仅达到了这个 upper bound 性能提升的 27%，这意味着巨大的 gap 未被解决。论文将此归因于 timing constraints 和 adaptation overhead，但缺乏更细粒度的 breakdown 分析。

3. **SARSA vs Q-Learning 的选择缺乏论证**：
   - SARSA 是 on-policy 算法，Q-Learning 是 off-policy 且理论上收敛更快。论文未解释为何选择 SARSA，也未对比两者。考虑到 exploration 时 on-policy 和 off-policy 的行为差异可能显著，这个选择值得更多讨论。

4. **公平性和 QoS 完全未涉及**：
   - 论文仅优化总体吞吐量，未考虑 thread-level fairness。在 CMP 环境下，RL 可能学会 starve 某些 thread 以最大化总 bus utilization。虽然有 10,000-cycle timeout 机制，但这是非常粗粒度的保护。Section 5.4 与 Fair Queueing 的对比（FQ 仅 2% 提升）并不能说明 RL 本身是公平的。

5. **Reward 设计的单一性**：
   - Reward 仅基于 data bus utilization（read/write 发出时 reward=1），这隐含假设 bus utilization 与系统性能高度相关。但在某些场景下（如 latency-sensitive workload），优化 utilization 不等于优化性能。论文虽在评估中验证了两者的相关性，但未探索更精细的 reward 设计（如基于 miss penalty、IPC 等）。

6. **DDR2 时代的局限性**：
   - 实验基于 DDR2-800 单 rank 配置，仅 4 banks/chip，state space 相对较小。现代 DDR5/HBM 系统具有更多 bank groups、更多 ranks、更复杂的 timing constraints，CMAC 的 32KB 存储和 6 个属性是否仍然足够存疑（尽管这不应苛责 2008 年的工作）。

### 疑问或值得追问的点

- Feature selection 在 4 个 benchmark 上训练，然后在全部 9 个上评估——虽然论文声称"no qualitative differences"，但这个 train/test 划分非常弱，且训练集选择了仿真最快的 4 个，可能有 selection bias。
- CMAC 的 hash collision 率是多少？论文称"not a major problem"但未给出量化数据。对于 long-running workload，collision 是否会导致 Q-value 退化？
- ε = 0.05 意味着每 20 个 DRAM cycle 就有一次随机决策。在 high-contention 场景下，这个探索代价是否可接受？是否考虑过 decaying ε 策略？
- 论文未讨论 RL scheduler 的 warm-up 时间。从 optimistic initialization 到收敛需要多少 cycle？这对 short-running workload 或频繁 context switch 的场景可能是关键问题。
- 多控制器场景下声称"无需协调"，但实验中每个 channel 服务不同物理地址，thread 间的内存访问可能自然分散。如果存在 channel-level contention（如 NUMA-unaware allocation），是否还能维持这个结论？
