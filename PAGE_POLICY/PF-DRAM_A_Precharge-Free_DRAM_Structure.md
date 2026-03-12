---
title: "PF-DRAM: A Precharge-Free DRAM Structure"
authors: "Nezam Rohbani, Sina Darabi, Hamid Sarbazi-Azad"
venue: "ISCA 2021"
year: 2021
---

# PF-DRAM: A Precharge-Free DRAM Structure

## 基本信息

- **发表**：ISCA (The 48th Annual International Symposium on Computer Architecture), 2021
- **作者单位**：IPM (Institute for Research in Fundamental Sciences) & Sharif University of Technology, Tehran, Iran

## 一句话总结

> 通过消除 DRAM Precharge 阶段，利用 bitline 上残留电压作为下次 Activation 的起点，实现功耗降低 35.3%、性能提升 8.6%。

## 问题与动机

**核心问题**：DRAM 的 latency 和 energy per access 在过去多代技术演进中几乎没有改善，而 Precharge + Activation 是 DRAM 功耗的主要来源。

**为什么重要**：

- Memory subsystem 占系统功耗的 25%-57%。
- 在传统 DRAM 中，每次 Activation 前都必须将 subarray 中所有 bitline precharge 到 VDD/2，这不仅消耗大量能量，还引入 tRP 延迟（DDR4 中约 14ns，占 tRC 的约 1/3）。
- 随着多核系统的普及，row-hit rate 急剧下降（论文数据显示从单核 73.8% 降到 8 核 12.4%），precharge 操作更加频繁，功耗问题愈发严重。

**现有工作的不足**：

- 所有已有 DRAM 设计（从最初原型到最新产品）都需要 Precharge 阶段来为 sense amplifier 建立参考电压。
- Row-Buffer Decoupling (RBD) [ISCA'14] 等方案虽可以通过 overlap tRP 来隐藏延迟，但并未减少 precharge 本身的能耗，甚至在 write 操作中引入额外的 precharge。
- 数据压缩/编码方案虽然利用数据相似性减少功耗，但引入解压延迟和额外的地址映射复杂度。
- 传统 DRAM 中，**无论 bitline 上前后两次访问的值是否相同，charge/discharge 的能耗都完全一样**——这是根本性的浪费。

## 核心方法

### 关键思路

**Key Observation**：在传统 DRAM 中，Precharge 将 bitline 拉到 VDD/2，然后 Activation 再将其驱动到 VDD 或 0，即使连续两次访问同一 bitline 上的值完全相同，这组 charge-discharge 循环仍然完整执行。实测数据显示，连续两次 Activation 中 bitline 发生翻转的概率平均仅为 9.1%（单核），即绝大多数 precharge + activation 的能耗是冗余的。

**Core Idea**：取消 Precharge 阶段，让 bitline 保持上一次 Activation 结束后的电压状态（VDD 或 0），仅当新访问 cell 的值与 bitline 残留电压不同时才发生翻转。这需要一个能在 8 种不同起始状态下正确工作的特殊 sense amplifier 设计。

### 技术细节

#### 1. Bitline 配置的根本改变

传统 DRAM 中，一对 bitline（bitline 和 $\overline{\text{bitline}}$）始终携带互补电压。PF-DRAM 中将其重新命名为 **Bitline Left (BLL)** 和 **Bitline Right (BLR)**，因为它们不再携带互补值——两者始终保持相同电压（同为 VDD 或同为 0）。

#### 2. SA Imbalancer（核心创新）

PF-DRAM 需要处理 8 种可能的起始状态组合（见论文 Table I）。SA Imbalancer 是两个 tri-state NOT gate，由 BOOSTL 和 BOOSTR 信号控制：

- 当被访问 cell 连接在 BLL 侧时，激活 BOOSTL；连接在 BLR 侧时，激活 BOOSTR。
- **Scenario I（cell 值 = bitline 值，无扰动）**：SAL 和 SAR 电压相同，此时 boosted 一侧的两个并联 PMOS/NMOS 晶体管在 race condition 中获胜，驱动 SA 到正确状态。
- **Scenario II（cell 值 ≠ bitline 值，有扰动 VSPF）**：电压扰动 VSPF = VDD · Ccell/(CBL + Ccell)，幅度是传统 DRAM 中 VSconv 的 **2 倍**（因为 cell 与 bitline 之间电压差从 VDD/2 增加到 VDD）。此时单个 NOT gate 因输入端电压更接近 rail（0 或 VDD）而驱动电流更大，在 race 中胜出。

#### 3. Bitline Decoupling & Voltage Updater

- **M4, M5（Decoupler）**：将 bitline 与 SA 内部节点（SAL, SAR）解耦。SA 激活时断开 bitline，使 SA 仅驱动寄生电容极小的内部节点，大幅加速 stabilization 并降低功耗。
- **M2, M3（Voltage Updater）**：Restoration 阶段将 BLL 和 BLR 同时连接到 SA 的同一侧输出，使两条 bitline 更新为相同电压（即被访问 cell 的值）。通过 odd/even wordline 选择决定激活 VUL+DCR 还是 VUR+DCL。

#### 4. 操作时序（5 个 Time Period）

| 阶段                        | 操作                                   | 信号状态                               |
| ------------------------- | ------------------------------------ | ---------------------------------- |
| TP1: Charge Sharing       | WL 激活，cell 与 BLL/BLR 共享电荷            | BLEQ=1→0, DCL/DCR=1, SAEN=0        |
| TP2: SA Stabilization     | SA 激活并快速稳定                           | SAEN=1, BOOSTL/BOOSTR=1, DCL/DCR=0 |
| TP3: BL Update            | SA 驱动 bitline 更新 + Column Access 可并行 | VUR+DCL 或 VUL+DCR 激活               |
| TP4: Row Open             | 行保持打开，等待后续列访问                        | 维持 TP3 状态                          |
| TP5: Closing & Equalizing | 仅均衡 SA 内部节点（非 bitline）               | SAEN=0, BOOST=0, DCL/DCR=1, BLEQ=1 |

#### 5. 关键设计权衡（Trade-offs）

**功耗 vs. 最坏情况峰值功率**：

- 当连续访问导致 bitline 每次都翻转时（极端对抗性访问模式），PF-DRAM 的峰值能耗比传统 DRAM 高 29%。这是因为 bitline 电压摆幅从 VDD/2 增加到 VDD。需要在电源轨设计上考虑此 worst-case。

**面积 vs. 功能**：

- SA Imbalancer 电路带来约 8.8% 的面积开销，但其中 M1-M3 在传统 DRAM 中已存在（用于 precharge/equalize），M4-M5 decoupling transistors 的开销仅约 0.8%。Imbalancer 晶体管 W/L=2/1 远小于 SA 晶体管的 10/1，实际面积开销可能更小。

**与数据压缩的交互**：

- PF-DRAM 在数据完全随机时仍有效（P0→1=25%，此时 Eacc(PF)=0.64×Eacc(conv)）。但数据压缩会降低 DRAM 中数据的相似性，可能削弱 PF-DRAM 的效果。两者存在一定程度的 tension。

**VDD/2 电源的消除**：

- PF-DRAM 不再需要 VDD/2 internal regulator，简化了电源管理设计。

#### 6. 能耗模型

传统 DRAM 每次访问能耗（与数据无关）：

$E_{acc}^{conv} = E_{pre} + E_{act}^{conv} = \frac{1}{2}(1+\beta) \cdot C_{BL} \cdot V_{DD}^2$

PF-DRAM 每次访问能耗（依赖于 bitline flip 概率）：

$E_{acc}^{PF} = P_{0 \to 1} \cdot 2 \cdot C_{BL} \cdot V_{DD}^2$

其中 β≈0.54 为 precharge 阶段 bitline 间电荷共享因子。当 P0→1 足够低时（实测平均 ~9.1% 单核），PF-DRAM 的能耗优势非常显著。

### 与现有工作的区别

| 对比方案                                      | 核心差异                                                                                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Row-Buffer Decoupling (RBD)** [ISCA'14] | RBD 通过 early precharging 隐藏 tRP 延迟，但仍执行 precharge 操作（不省能耗），且 write 时引入额外 precharge。PF-DRAM 则从根本上消除 precharge。                         |
| **Closed yet Open DRAM** [DAC'18]         | 同样利用 bitline isolation 同时执行 Read 和 Precharge，减少了 SA 的 precharge latency，但仍需 precharge bitline 到 VDD/2。PF-DRAM 完全无需 bitline precharge。 |
| **CROW** [ISCA'19]                        | CROW 通过 copy row 减少 activation latency 并提升能效，但其优化仍在传统 precharge 框架内。PF-DRAM 是从 bitline 工作模式层面的根本性变革。                                  |

## 实验评估

### 实验设置

- **仿真平台**：
  
  - Architecture-level：**gem5** full-system simulator
  - Circuit-level：**HSPICE**（16nm Multi-Gate Predictive Technology Model）
  - 使用 Lumped-element model（512 段）建模 bitline
  - Monte Carlo 仿真 2000 次迭代评估 Process Variation 鲁棒性

- **硬件规格配置**：
  
  - CPU：单核/2/4/8 核 OoO X86-64, 4-issue, 3GHz
  - L1：每核私有 32KB 4-way, 2 cycle; L2：共享 2MB 8-way, 8 cycle
  - 主存：8GB DDR4-2400, 2 Channel, 2 Rank/Channel, 16 Bank/Rank, 8KB row-buffer, Open-row policy, 64ms refresh interval
  - MAT 512×512, CBL=144fF, Ccell=24fF

- **Workload**：
  
  - SPEC CPU2017（8 INT + 8 FP）：deepsjeng, leela, mcf, omnetpp, perlbench, x264, xalancbmk, xz, bwaves, cam4, imagick, lbm, nab, namd, povray, wrf
  - PARSEC 2.1（5 个多线程）：bodytrack, canneal, ferret, fluidanimate, stream
  - 各 workload fast-forward 1B 指令，详细模拟 500M 指令

- **Timing 参数**：
  
  - 传统 DRAM：tRCD=14.2ns, tCL=14.2ns, tRP=14.2ns
  - PF-DRAM：tRCD=10.8ns, tRP=0.8ns（仅均衡 SA 内部节点）

- **对比 baseline**：传统 DDR4 DRAM（带 complete bitline decoupling scheme）

### 关键结果

1. **功耗降低**：平均降低 **35.3%**（最高 54.2%，bwaves 工作负载）。Precharge+Activation 功耗具体降低：单核 6.4×, 2 核 5.2×, 4 核 4.0×, 8 核 3.2×。Refresh 功耗平均降低约 6.9×。

2. **性能提升**：平均 memory access latency 降低 **19.2%**。IPC 平均提升 **8.6%**，最高达 24.3%。8 核配置下 SPEC CPU2017 最高 IPC 提升 15.1%。

3. **鲁棒性**：在极端 PV 条件下（σ=20%），PF-DRAM 的平均故障率 0.34%，优于传统 DRAM 的 0.58%。Noise margin 方面，PF-DRAM（85-90mV）略优于传统 DRAM（85mV）。

4. **面积开销**：约 **8.8%**，cell array 无修改，开销集中在 SA Imbalancer 和 decoupling transistors。

### 结果分析

**Bitline-flip 概率是关键因素**：

- 单核系统下 SPEC CPU2017 平均 bitline-flip 率仅 9.1%，最低 3.9%（x264），最高 19.5%（namd）。
- 多核增加 flip 率（1 核→8 核：9.1%→13.6%），因并发访问降低数据依赖性。但由于 narrow-width values、大量 '0' bits、机器码低 hamming distance 等因素，增长幅度有限。
- PARSEC 2.1 多线程 workload 的 flip 率增长比多程序 SPEC CPU2017 更慢（数据相似性更高）。

**多核场景更受益**：

- 多核下 row-hit rate 急剧下降（73.8%→12.4%），传统 DRAM precharge 更频繁，PF-DRAM 的 latency 优势更明显（单核减少 2.1ns → 8 核减少 28.8ns）。
- PDP (Power-Delay Product) 平均降低：SPEC CPU2017 约 43.0%，PARSEC 2.1 约 31.3%。

**效果较差的场景**：

- fluidanimate（PARSEC 2.1）功耗降低仅 12.6%-16.7%，因其 row-hit rate 较高且 bit-flip 概率较大。
- cam4 功耗降低最低（21.5%），但其 memory access rate 超过 30%，因此 IPC 提升反而较为明显。

## 审稿人视角

### 优点

1. **根本性创新**：论文从 DRAM bitline 工作原理层面提出了全新范式——这是自 DRAM 发明以来首次从根本上消除 Precharge 阶段的设计。idea 清晰且 elegant，改动集中在 subarray 级别的 SA 电路，cell array 完全不变。

2. **全面的评估方法论**：同时使用 HSPICE circuit-level 和 gem5 architecture-level 仿真，覆盖了功能正确性、PV 鲁棒性、noise immunity、性能、功耗等多个维度。Monte Carlo 2000 次迭代的 PV 分析、noise margin 扫描等都是扎实的电路级验证。

3. **正交性好**：与几乎所有已有 DRAM 功耗优化技术正交（power gating、fine-grained activation、refresh optimization 等），可以叠加使用。这极大地提升了该方案的实用价值。

4. **面积开销可接受**：8.8% 的面积开销对于 35%+ 的功耗收益和 8.6% 的性能提升是合理的 trade-off，且 cell array 完全无需修改。

5. **兼容性**：与 JEDEC DDR/HBM 标准兼容，仅需 memory controller 端做最小修改（主要是取消 PRE 命令的发送逻辑和调整 timing 参数），降低了产业化的门槛。

### 不足

1. **Peak power 问题未充分讨论**：论文承认 worst-case 下 PF-DRAM 峰值能耗比传统 DRAM 高 29%，但仅一段带过。在实际芯片设计中，peak power 直接决定电源网络设计和封装规格。恶意 workload（如 RowHammer 攻击变体）可能故意触发此场景，这方面需要更深入的分析和缓解策略。

2. **Bitline-to-bitline/wordline coupling noise 分析不够定量**：论文定性分析了 PF-DRAM 中 bitline 电压摆幅更大（VDD vs VDD/2）会增加 coupling noise，但建议"用更弱的 updater transistors 降低 dV/dt"来缓解，缺少 HSPICE 的定量仿真数据支撑。在高密度 MAT 中，这可能是制约工艺 scaling 的关键问题。

3. **数据编码/压缩的交互效应缺乏实验数据**：论文提到数据压缩可能降低 PF-DRAM 效率，但未给出任何量化实验。考虑到 HBM3 等现代存储接口已广泛使用数据编码/compression，这一交互效应对实际部署至关重要。

4. **Write 操作的详细分析不足**：论文主要聚焦 Read/Refresh 的能耗模型，但对 Write-back（尤其是 dirty line write-back 导致的值更新）如何影响 bitline-flip 概率的分析较少。在 write-intensive workload 下效果可能有所不同。

5. **实验方法论的局限**：
   
   - 仅使用 Open-row policy 评估，未对比 Close-row policy 下的效果（尽管 PF-DRAM 在 close-row 下应该更有优势，因为传统 close-row 每次访问都要 precharge）。
   - gem5 的功耗模型基于 datasheet 的 IDD 参数 derating，而非更精确的 DRAMPower 或 Ramulator + DRAMSim 等专用功耗模型。
   - 500M 指令的模拟窗口对于某些 memory-intensive workload 可能不够充分。

6. **tRCD 减少的论证可以更严谨**：论文声称 tRCD 从 14.2ns 降到 10.8ns，理由是 SA 不需要直接驱动 bitline 的大电容。但实际上 tRCD 还包含 wordline activation 和 charge sharing 延迟，SA stabilization 只是其中一部分。需要更详细的时序分解来支撑这个 3.4ns 的减少量。

### 疑问或值得追问的点

- **Temperature sensitivity**：高温下 DRAM cell retention time 降低，leakage 增加。PF-DRAM 中 bitline 长期维持在 VDD 或 0（而非 VDD/2），对 non-selected cell 的 leakage 路径和 retention 特性有何影响？论文声称 refresh rate 不变，但这需要更系统的温度敏感性分析。

- **与 RowHammer 的交互**：PF-DRAM 中相邻 bitline 可能长期处于 VDD，aggressor row 对 victim row 的干扰模式是否与传统 DRAM 不同？是否会加剧或缓解 RowHammer 效应？

- **3D-stacked DRAM（HBM）适用性**：论文提到 HBM 中 I/O 能耗更低使得 Precharge/Activation 占比更高，理论上 PF-DRAM 在 HBM 中收益更大。但 HBM 的 TSV 结构和更密集的 bank 布局是否对 PF-DRAM 的 SA Imbalancer 面积开销提出额外约束？

- **LPDDR 场景**：在移动设备（LPDDR）中，功耗优先级更高，PF-DRAM 的收益可能更大。但 LPDDR 通常使用 VDD=1.1V 甚至更低电压，VSPF 的 sensing margin 是否仍然足够？

- **关于 P0→1 的提取方法**：论文通过 gem5 trace 统计 bitline-flip 概率，但这依赖于特定的 address mapping 和 page interleaving 策略。不同的地址映射方案会如何影响 P0→1 的分布？
