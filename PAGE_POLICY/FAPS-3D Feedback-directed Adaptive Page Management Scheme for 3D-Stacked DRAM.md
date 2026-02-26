---
title: "FAPS-3D: Feedback-directed Adaptive Page Management Scheme for 3D-Stacked DRAM"
authors: "Muhammad M. Rafique, Zhichun Zhu"
venue: "MEMSYS 2019"
year: 2019
---

# FAPS-3D: Feedback-directed Adaptive Page Management Scheme for 3D-Stacked DRAM

## 基本信息

- **发表**：MEMSYS (International Symposium on Memory Systems), 2019
- **作者**：Muhammad M. Rafique, Zhichun Zhu（University of Illinois at Chicago）

## 一句话总结

> 利用 3D-stacked DRAM logic base 中的 policy engine，基于 per-bank row-buffer hit-rate 反馈式动态切换 open/close page policy。

## 问题与动机

### 核心问题

在多核系统中，多个应用并发执行产生交织的、随机的 memory access pattern，静态的 page policy（无论是 open-page 还是 close-page）无法在所有执行阶段都取得最优性能。论文要解决的是**如何在运行时动态调整 page policy 以适应变化的 memory access behavior**。

### 为什么重要

1. **Inter-application diversity**：不同应用的 cache miss rate 和 row-buffer hit-rate 差异显著。
2. **Intra-application diversity**：同一应用在不同执行阶段的 row-buffer hit-rate 也有剧烈波动。论文通过实验展示了 SPEC CPU2006 中 gcc 的 hit-rate 在 16%-72% 之间波动，namd 在 40%-81% 之间波动，gemsFDTD 在 17%-79% 之间波动。这意味着静态 page mode 在整个执行期间不可能是最优的。

### 现有工作的不足

- **Access-based policies (ABP)**（如 PBRM [Awasthi et al., PACT 2011]）：在 DRAM page 粒度维护 history table 追踪每个 row 的 hit 次数来预测 page closure 时机。问题在于 history table 容量有限，在多程序环境下随机访问流中 table hit rate 较低，预测精度下降；且 storage overhead 较大。
- **Time-based policies (TBP)**（如 Intel Adaptive HAPPY [Ghasempour et al., MEMSYS 2016]）：通过 timeout counter 控制 page 保持 open 的时间长度。问题在于其全局性质，不考虑短时间尺度内应用行为的动态变化。
- 上述方案均针对传统 DDRx 内存系统设计，**未充分利用 3D-stacked DRAM 的结构优势**（独立 vault 控制、logic base 可集成自定义逻辑、大量 bank 提供的并行度）。

## 核心方法

### 关键思路

核心 observation 是：在 3D-stacked DRAM 中，每个 vault 的 vault controller 可以独立管理其下属 bank，且 logic base 提供了集成自定义逻辑的面积。因此可以在 vault controller 中嵌入一个轻量级的 **policy engine**，以 **per-bank 粒度**监控 row-buffer hit-rate，通过 **2-bit saturating counter FSM** 反馈式地在 open-page 和 close-page 之间动态切换，而无需修改处理器端的 memory controller。

关键洞察：由于 open-page 下 conflict 的延迟是 hit 延迟的约 3 倍（tRP + tRCD + tCL vs. tCL），当 hit-rate 低于 50% 时，conflict 带来的额外延迟已经抵消了 hit 节省的时间，此时 close-page 更优。

### 技术细节

#### 整体架构

Policy engine 位于每个 vault 的 logic base 中，与 DRAM command queue 协同工作。其操作对处理器端的 memory controller 完全透明，无需修改主控端。

#### Threshold Selection（阈值选择）

基于延迟分析推导判断准则：

- Open-page hit 延迟：tCL
- Open-page conflict 延迟：tRP + tRCD + tCL
- Close-page miss 延迟：tRCD + tCL

由于 tRCD ≈ tRP ≈ tCL（均为 11 cycles），conflict 延迟约为 hit 延迟的 3 倍。因此当 hit-rate < 50% 时，open-page 的平均延迟高于 close-page，应切换为 close-page。

#### Dynamic Bank Profiling（动态 bank 分类）

- 每个 bank 维护一个 **2-bit saturating counter** 作为 FSM
- 状态 (10)₂ 和 (11)₂ → 推荐 open-page mode
- 状态 (01)₂ 和 (00)₂ → 推荐 close-page mode
- 每 1000 次 bank access 为一个 epoch，epoch 结束时更新策略
- Bank 按 hit-rate 分为三组：
  - **HHR (High-Hit-Rate)**：hit-rate > 75%
  - **MHR (Medium-Hit-Rate)**：25% ≤ hit-rate ≤ 75%
  - **LHR (Low-Hit-Rate)**：hit-rate < 25%

#### Algorithm I：当前为 open-page 的 bank

- 计算实际 bank hit-rate = bank hits / bank accesses
- 若 hit-rate < 25%（LHR）：FSM 直接跳转到 (00)₂，下一 epoch 切换为 close-page
- 若 25% ≤ hit-rate < 50%：FSM saturating decrement
- 若 hit-rate ≥ 50%：FSM saturating increment
- Epoch 结束后根据 FSM 状态决定下一 epoch 的 page mode

#### Algorithm II：当前为 close-page 的 bank

- 由于 close-page 下不存在真正的 row-buffer hit，使用 **SRAM-based hit-register** 记录"potential hit"（如果下一次访问与上一次访问同一 page，则计为 potential hit）
- 计算 Potential Bank Hit-Rate (PBHR) = potential hits / total accesses
- 若 PBHR ≥ 75%（HHR）：FSM 直接跳转到 (11)₂，下一 epoch 切换为 open-page
- 若 50% ≤ PBHR < 75%：FSM saturating increment
- 若 PBHR < 50%：FSM saturating decrement
- Epoch 结束后根据 FSM 状态决定下一 epoch 的 page mode

#### Scheduling Policy 联动

- Open-page bank 使用 **FR-FCFS** 调度（优先服务 row hit）
- Close-page bank 使用 **FCFS** 调度
- Address mapping 也相应调整：open-page 时低位分配给 column 以最大化 row hit，close-page 时低位分配给 vault 以最大化并行度

#### Overhead 分析

- 每个 bank 的 hit-register entry：52 bits（bank ID + access count + last/current page ID + hit count）
- 每个 vault 16 banks → 16 × 52 = 832 bits = 104 bytes
- 32 个 vault 合计：**3.25 KB**（比 PBRM 的 320 KB 少 **98x**）
- 面积开销（32nm CACTI 6.0 估算）：0.08 mm²
- 能耗开销：0.023 nJ/access
- 每 epoch 一次除法运算，开销可忽略

### 与现有工作的区别

| 特征   | PBRM [13]            | Intel Adaptive HAPPY [35] | FAPS-3D                           |
| ---- | -------------------- | ------------------------- | --------------------------------- |
| 策略类型 | Access-based (动态)    | Time-based (动态)           | Access-based (动态)                 |
| 追踪粒度 | Per DRAM page        | 全局                        | **Per bank**                      |
| 存储开销 | ~320 KB              | 较低                        | **~3.25 KB**                      |
| 目标平台 | DDRx                 | DDRx                      | **3D-stacked DRAM**               |
| 自适应性 | 受限于 history table 容量 | 不考虑短期行为变化                 | 反馈式 FSM，响应快                       |
| 实现位置 | Memory controller    | Memory controller         | **Vault controller (logic base)** |

关键差异：FAPS-3D 在更粗的 per-bank 粒度上工作（而非 per-page），但通过利用 3D-stacked DRAM 的 vault 独立性和大量 bank 的并行度，在 storage overhead 减少 98x 的同时仍获得了更好的性能。

## 实验评估

### 实验设置

- **仿真平台**：gem5 cycle-accurate simulator (x86-64)，扩展支持 5 种 page policy
- **处理器配置**：8 个 out-of-order cores @ 3.0 GHz，8-issue superscalar，192-entry ROB
- **Cache 层次**：
  - L1 I/D: 32KB private, 4-way, 64B line, 4 cycles
  - L2: 256KB private, 8-way, 64B line, 10 cycles
  - L3: 16MB shared, 16-way, 64B line, 20 cycles
  - MOESI coherence protocol
- **3D-stacked DRAM**：8GB，32 vaults，8 DRAM layers，16 banks/vault，64 TSVs/vault，DDR3-1600 timing (tRCD=tRP=tCL=11 cycles)，1KB row-buffer
- **Workload**：SPEC CPU2006，12 组 8 程序混合负载：
  - HM1-HM4：高内存密集型（MPKI > 20）
  - LM1-LM4：低内存密集型（1 ≤ MPKI ≤ 20）
  - MX1-MX4：混合型（4 HM + 4 LM）
  - 快进初始化阶段，warm-up 100M instructions，详细模拟 800M instructions
- **对比 baseline**：
  - CLOSE-P（静态 close-page + FCFS）
  - OPEN-P（静态 open-page + FR-FCFS）
  - PBRM（Prediction-based Row-Buffer Management [Awasthi et al.]）
  - INTAH（Intel Adaptive HAPPY [Ghasempour et al.]）

### 关键结果

1. **整体性能提升**：FAPS-3D 相比 CLOSE-P 平均提升 **15.2%**，相比 OPEN-P 提升 **6.9%**，相比 PBRM 提升 **4.0%**，相比 INTAH 提升 **3.3%**（geometric mean of IPC across 12 workload sets）。

2. **主存访问延迟降低**：FAPS-3D 相比 CLOSE-P 降低 MMAL **41.0%**，相比 OPEN-P 降低约 24.4%，相比 PBRM 降低约 9.3%，相比 INTAH 降低约 4.8%。

3. **Bank conflict 减少**：相比 OPEN-P，FAPS-3D 平均减少 bank conflict **17.5%**，PBRM 减少 14.7%，INTAH 减少 15.1%。

4. **能耗降低**：FAPS-3D 相比 CLOSE-P 降低能耗 **12.7%**，相比 OPEN-P 降低 5.7%，相比 PBRM 降低 3.4%，相比 INTAH 降低 2.2%。

### 结果分析

- **高内存密集型负载获益最大**：HMx 平均提升 22.2%（over CLOSE-P），因为这类 workload 频繁访问主存，page policy 优化效果显著。
- **低内存密集型负载获益较小**：LMx 平均提升 7.3%（over CLOSE-P），PBRM 在此类负载上与 FAPS-3D 性能接近，但 FAPS-3D 的 storage overhead 优势（98x 更小）仍有意义。
- **Queue occupancy**：FAPS-3D 将 read queue 平均占用从 13.7%（CLOSE-P）降至 8.2%，write queue 从 40.6% 降至 31.0%，归因于 bank conflict 减少带来的更高效请求处理。
- 论文**未做 sensitivity analysis**（如 epoch length、threshold 值、bank 数量等参数的敏感性分析），仅提到阈值是"empirically selected"。

## 审稿人视角

### 优点

1. **问题定位清晰，动机充分**：通过 SPEC CPU2006 benchmark 的 row-buffer hit-rate 时序图清楚展示了 intra-application diversity，有效论证了动态 page policy 的必要性。

2. **设计简洁实用，overhead 极低**：per-bank 粒度的 2-bit FSM + 52-bit hit-register 的设计非常轻量，总共仅 3.25KB，比 PBRM 少 98x，且不需要修改处理器端 memory controller，工程可行性好。

3. **充分利用了 3D-stacked DRAM 的架构特性**：将 policy engine 放在 vault controller 的 logic base 中，利用 vault 的独立性实现 per-bank 自治管理，这是一个自然且合理的设计选择。

4. **实验评估较为完整**：不仅报告了 IPC 性能，还分析了 MMAL、bank conflict、queue occupancy、energy consumption 等多个维度。

### 不足

1. **Threshold 选择缺乏理论或系统性分析**：25%/50%/75% 的阈值被描述为"empirically selected"，但论文没有提供任何 sensitivity study 来证明这些值的最优性或鲁棒性。对于不同的 tRCD/tRP/tCL 比例关系或不同的 workload 特征，这些阈值是否仍然有效？这是一个重要的缺失。

2. **Epoch length (1000 accesses) 固定且未做分析**：epoch 过长会导致对行为变化的响应迟滞，过短则会引入噪声和频繁切换的开销。论文未讨论 epoch length 的选择依据，也未做 sensitivity analysis。

3. **实验方法论存在一些问题**：
   
   - 使用的是 **SPEC CPU2006**（2019 年的论文），workload 偏老旧，且缺乏真正的多线程负载（如 PARSEC、SPLASH-2）或新兴的 data-intensive workload。多程序混合并不等同于真正的共享内存多线程应用。
   - 3D-DRAM 模型是"simplified and accurate"的自建模型，而非使用成熟的 3D DRAM 模拟器（如 CasHMC），模型准确性缺乏独立验证。
   - Baseline 的 scheduling policy 选择有些不一致：CLOSE-P 用 FCFS，OPEN-P 用 FR-FCFS。虽然这有其合理性（close-page 不存在 row hit 可供优先调度），但使得性能比较中混杂了 scheduling policy 差异的影响。

4. **Per-bank 粒度 vs. per-page 粒度的 trade-off 讨论不够深入**：论文声称 per-bank 粒度能达到与 per-page 粒度"comparable"的性能，但缺乏详细分析在什么条件下 per-bank 粒度会显著劣于 per-page 粒度（例如当同一 bank 内不同 row 的访问 pattern 差异很大时）。

5. **缺乏可扩展性分析**：论文固定在 32 vaults、16 banks/vault 的配置下评估。当 vault 数量或 bank 数量变化时，scheme 的效果如何？在更大规模的 HBM2/HBM3 配置下是否仍然有效？

6. **与 scheduling policy 的交互未充分探讨**：page policy 和 scheduling policy 紧密耦合，论文仅使用了 FCFS 和 FR-FCFS 两种基本策略，没有探索与更先进 scheduling policy（如 ATLAS、BLISS、TCM 等）的组合效果。

### 疑问或值得追问的点

- 当 close-page bank 使用 hit-register 追踪 "potential hit" 时，如何处理多个请求在 queue 中排队的情况？hit-register 只记录"上一次访问的 page"，是否会因为 queue reordering 导致 potential hit 的估计不准确？
- Algorithm I 和 II 使用了不同的阈值（25%/50% vs. 50%/75%），这种不对称设计引入了 hysteresis，有助于避免频繁切换。但论文未明确讨论这一设计意图，也未量化切换频率和由此带来的 transition overhead。
- 初始状态全部设为 open-page 是否合理？对于 memory-intensive workload 的初始阶段，这可能导致大量 conflict。是否考虑过更保守的初始策略（如 close-page）或基于 first-epoch profiling 的初始化？
- Epoch 边界处的策略切换是否会引入 pipeline bubble 或 timing violation？从 open-page 切换到 close-page 时，正在进行的 bank operation 如何处理？
