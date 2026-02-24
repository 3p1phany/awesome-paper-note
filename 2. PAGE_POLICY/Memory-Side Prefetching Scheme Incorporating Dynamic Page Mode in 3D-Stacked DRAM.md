---
title: "Memory-Side Prefetching Scheme Incorporating Dynamic Page Mode in 3D-Stacked DRAM"
authors: "Muhammad M. Rafique, Zhichun Zhu"
venue: "IEEE Transactions on Parallel and Distributed Systems (TPDS), 2021"
year: 2021
---

# Memory-Side Prefetching Scheme Incorporating Dynamic Page Mode in 3D-Stacked DRAM

## 基本信息

- **发表**：IEEE TPDS, Vol. 32, No. 11, November 2021
- **作者单位**：University of Illinois at Chicago

## 一句话总结

> 在 3D-stacked DRAM 的 logic base 上实现 memory-side prefetching 与 dynamic page mode 的协同优化，通过 conflict-aware 策略和 utilization+recency 的 replacement policy 降低访存延迟。

## 问题与动机

论文要解决的核心问题是 3D-stacked DRAM 系统中 memory access latency 的优化。具体动机包含以下几个层面：

**Memory wall 问题在 many-core 时代加剧**：传统 DDRx 内存系统的有限 bandwidth 和较高的 off-chip access latency 无法满足现代多核处理器的需求。

**应用行为的多样性导致静态策略失效**：论文从两个维度论证了这一点：

- **Inter-application diversity**：多个应用并发执行时，cache miss rate 和 row buffer hit rate 差异巨大。
- **Intra-application diversity**：单个应用在不同执行阶段的 row buffer hit rate 也有显著波动。论文以 SPEC CPU2006 为例，gcc 的 hit rate 在 16%-72% 之间波动，namd 在 40%-81% 之间波动。这意味着静态 page mode 在整个执行周期内不是最优的。

**现有工作的不足**：

- 传统 memory-side prefetching（如 Meeting Midway [8]）使用 static open page mode，无法适应动态变化的访存行为。
- 既有 prefetch buffer replacement 采用简单的 LRU 策略，未考虑 prefetched data 的实际利用率。
- Core-side prefetching 缺乏对 memory state 的感知，aggressive prefetching 会浪费 bandwidth 并导致 interconnect congestion。

**3D-stacked DRAM 带来的机遇**：TSV 提供的巨大 internal bandwidth（不占用 off-chip bandwidth）、logic base 上可集成自定义逻辑、大量 bank 提供的 memory-level parallelism，使得 memory-side prefetching 和 dynamic page mode 可以以全新方式实现。

## 核心方法

### 关键思路

MPDM 的核心洞察是：**page mode 的选择与 prefetching 策略应当协同工作而非独立运行**。在 high locality 阶段使用 open page mode 并基于 conflict 信息做 prefetching，在 low locality 阶段切换为 close page mode 并基于 row utilization history 做 prefetching。同时，利用 3D-stacked DRAM 的 TSV internal bandwidth 以 row 粒度做 prefetching，并通过 utilization+recency 的复合 replacement policy 提升 prefetch buffer 的有效利用率。

### 技术细节

#### 1. 整体架构（MPDM Engine）

MPDM Engine 位于 3D-stacked DRAM 的 logic base，包含两个核心组件：

- **Page Policy Controller**：实现 dynamic page mode，内含 HR（Hit Register）用于追踪每个 bank 的 row hit 情况。
- **Prefetch Controller**：实现 conflict-aware prefetching，内含三个关键表：
  - **RUT（Row Utilization Table）**：追踪 row buffer 中 row 的 cache line 级别利用率（16 entries/channel，每 entry 20 bits）。
  - **CT（Conflict Table）**：追踪被写回 bank 但未被 prefetch 的 recently accessed rows，fully associative 结构，entries 由所有 bank 共享（32 entries/channel，每 entry 20 bits）。
  - **ET（Eviction Table）**：记录从 prefetch buffer 中被替换出去的 row 及其利用率信息，用于指导未来 prefetch 决策。
- **Prefetch Buffer**：SRAM-based，16KB/channel，fully associative，存储粒度为整个 row buffer size（1KB）。

操作流程：请求到达 MPDM Engine → 先查 prefetch buffer → miss 则访问 DRAM bank → 数据返回处理器的同时，根据当前 prefetching policy 决定是否将 currently open row prefetch 到 prefetch buffer。

#### 2. Adaptive Page Management Scheme

**Threshold 选择的理论依据**：由于 DRAM 的 activate（tRCD）、precharge（tRP）和 column access（tCL）延迟基本相当，conflict 的延迟约为 hit 的 3 倍（tRP + tRCD + tCL vs. tCL）。因此当 row buffer hit rate 低于 50% 时，conflict 带来的额外延迟已经抵消了 hit 节省的时间，此时 close page mode 更优。

**Bank 分类**：基于 row buffer hit rate，将 bank 分为三类：

- **HHR（High Hit Rate）**：hit rate > 75%
- **MHR（Medium Hit Rate）**：25% ≤ hit rate ≤ 75%
- **LHR（Low Hit Rate）**：hit rate < 25%

**2-bit 饱和计数器 FSM**：每个 bank 维护一个 2-bit 饱和计数器。状态 (10)₂ 和 (11)₂ 推荐 open page mode，状态 (01)₂ 和 (00)₂ 推荐 close page mode。

**Open Page Bank 的策略（Algorithm 1）**：

- 每 1000 次 bank access（一个 epoch）计算实际 hit rate。
- Hit rate < 25%：直接跳转到 (00)₂，下个 epoch 切换为 close page。
- 25% ≤ hit rate < 50%：计数器减 1。
- Hit rate ≥ 50%：计数器加 1。

**Close Page Bank 的策略（Algorithm 2）**：

- 由于 close page mode 下没有实际的 row buffer hit，使用 HR（Hit Register）追踪 **potential bank hit rate (PBHR)**，即如果保持 open page 时可能获得的 hit 率。
- PBHR ≥ 75%：直接跳转到 (11)₂，下个 epoch 切换为 open page。
- 50% ≤ PBHR < 75%：计数器加 1。
- PBHR < 50%：计数器减 1。

**设计权衡——Bank-level vs. Page-level 粒度**：论文选择 bank-level 管理而非 page-level，理由是：(1) bank-level overhead 远低于 page-level；(2) 通过实验验证（Fig. 4），同一 bank 内的大多数 page 在特定 phase 内表现相似；(3) 3D-stacked DRAM 有数百个 bank，请求在 bank 间充分分散，bank-level 粒度足够有效。

#### 3. Conflict-Aware Prefetching

**Open Page Prefetching Policy（Fig. 5）**：

- Prefetch buffer miss 且 bank 有 row open 时：
  - Row buffer hit → 直接从 row buffer serve，保持 page open。
  - Row buffer miss → 检查 CT 中是否有当前 open row 的 entry：
    - 有 → prefetch 当前 open row 到 prefetch buffer，释放 CT entry，precharge bank。
    - 无 → 将当前 open row 信息加入 CT（CT 满则替换 LRU entry），precharge bank。
  - 最后 activate 新 row 并 serve 请求。

**核心思想**：如果一个 row 第二次被 evict（即出现在 CT 中），说明它很可能会再次被访问，因此 prefetch 它以减少未来的 conflict。

**Close Page Prefetching Policy（Fig. 6）**：

- Row buffer hit 时：递增 row 的 utilization counter (UC)。
  - UC 超过阈值（实验中设为 4）→ prefetch 当前 row，precharge bank。
  - UC 未超阈值 → 检查是否有 pending request 到当前 row，有则保持 open，否则 precharge。
- Row buffer miss 时：检查 ET 中当前 open row 的历史记录。
  - 找到且 utilization > 25% → prefetch 当前 row，precharge bank。
  - 未找到或 utilization ≤ 25% → 直接 precharge，不 prefetch。

**设计意图**：close page 阶段更保守地 prefetch，只 prefetch 历史上被充分利用或高频访问的 row，避免浪费 prefetch buffer 空间和能耗。

#### 4. Prefetch Buffer Replacement Policy（Fig. 7）

**传统方案**：纯 LRU 替换。

**MPDM 方案——Utilization + Recency 复合策略**：

- **4-bit Utilization Counter (UC)**：每访问一个新的 distinct cache line，UC 加 1。
- **4-bit Recency Counter (RC)**：采用类 LRU 的 stack distance 管理方式——被访问的 row 的 RC 设为最大值，其他 RC 大于该 row 原 RC 值的 row 的 RC 减 1。

**替换决策**：

1. 优先替换所有 cache line 都已被访问的 row（已完全利用）。
2. 若无这样的 row，计算 UC + RC 之和，替换 sum 最小的 row。
3. 若存在 sum 相同的多个 row，替换 RC 最低的（最不 recently used）。
4. 被替换 row 的信息记入 ET，用于指导未来 prefetch 决策。

**设计权衡**：这种策略的优势在于让高利用率且频繁访问的 row 在 prefetch buffer 中存活更久，相比纯 LRU 更能区分 "有用但暂时未被访问" 和 "确实不再需要" 的 row。

#### 5. Hardware Overhead

- Dynamic page mode overhead：3.25KB SRAM（across 32 channels）+ 1024 个 2-bit 计数器，面积 0.08mm²，能耗 0.023 nJ/access。
- Prefetching overhead：RUT 1.25KB + CT 2.5KB + ET 1.125KB = 总共 4.875KB，相比 prefetch buffer 本身（16KB × 32 = 512KB）微不足道。
- Prefetching 使用 TSV internal bandwidth，不占用 off-chip bandwidth。

### 与现有工作的区别

| 特性               | BASE              | MMD (Meeting Midway) [8]          | CAMPS [41]     | MPDM                              |
| ---------------- | ----------------- | --------------------------------- | -------------- | --------------------------------- |
| Page mode        | Static open       | Static open                       | Static open    | Dynamic (adaptive)                |
| Prefetch 触发      | 连续 2 次 hit 同一 row | 基于 prefetch usefulness 动态调 degree | Conflict-aware | Conflict-aware + page mode 协同     |
| Replacement      | —                 | LRU                               | LRU            | Utilization + Recency             |
| Prefetch history | 无                 | 有（usefulness feedback）            | 有（ET）          | 有（ET + CT 协同）                     |
| 能否适应 locality 变化 | 否                 | 部分（调 degree）                      | 否              | 是（page mode + prefetch policy 联动） |

与 **MMD** 的关键差异：MMD 仅动态调整 prefetch degree 但使用 static open page，无法在 low locality 阶段减少 conflict。MPDM 通过 page mode 切换从根源上减少 conflict，再用 conflict-aware prefetching 进一步缓解残余 conflict。

与 **CAMPS**（同一作者的前序工作）的关键差异：CAMPS 只有 conflict-aware prefetching 但使用 static open page mode，MPDM 在此基础上加入了 dynamic page mode 并使其与 prefetching 策略联动，同时改进了 replacement policy。

## 实验评估

### 实验设置

- **仿真平台**：gem5（cycle-accurate x86-64 simulator），扩展以支持不同方案。能耗估算基于 [16][51] 的模型，SRAM overhead 使用 CACTI 6.0（32nm）估算。
- **处理器配置**：8 个 out-of-order cores @ 3.6 GHz，8-issue superscalar，192-entry ROB。
- **Cache 层次**：L1I/D 32KB (4-way), L2 256KB (8-way), L3 16MB (16-way, shared), MOESI protocol。
- **3D-stacked Memory**：8GB, 32 channels, 8 DRAM layers, 16 banks/channel, 64 TSVs/channel, 1.25 Gb/s TSV signaling rate。
- **DRAM Specs**：DDR3-1600, 1KB row buffer, 16384 rows/bank, tRCD = tRP = tCL = 11 cycles。
- **Memory Controller**：FR-FCFS scheduling, Row:Rank:Bank:Channel:Column address mapping, open page mode (baseline)。
- **Prefetch Buffer**：16KB/channel, fully associative, t_buf = 4 cycles。
- **Workload**：12 组 SPEC CPU2006 + CPU2017 的 8 核 multiprogramming workload mix，按 MPKI 分为 HM（>20）、LM（1-20）、MX（4HM+4LM）三类，每类 4 组。使用 reference data set，fast-forward initialization phase，warmup 100M instructions，详细模拟 1B instructions。
- **对比方案**：BASE、INTAH（Intel 自适应 page closure 的 HAPPY 实现）、FAPS-3D（作者前序工作，仅 dynamic page mode）、MMD（Meeting Midway，dynamic prefetch degree + static open page + LRU）、CAMPS（作者前序工作，conflict-aware prefetch + static open page）。

### 关键结果

1. **性能提升**：MPDM 平均提升 21.8% over BASE（HM: 31.2%, LM: 13.6%, MX: 21.3%），比 MMD 平均高 13.2%，比 CAMPS 平均高 6.6%。

2. **Main Memory Access Latency 降低**：MPDM 平均降低 42.0% over BASE（MMD 降低 16.2%，CAMPS 降低 29.2%）。

3. **Bank conflict 减少**：MPDM 平均减少 22.9% over BASE（MMD 10.8%，CAMPS 19.0%）。

4. **Prefetching accuracy**：MPDM 达到 73.0%（BASE 49.2%，MMD 61.6%，CAMPS 67.1%）。

5. **能耗降低**：MPDM 平均降低 10.9% over BASE。

6. **Queue occupancy**：MPDM 的 read queue 平均占用率为 6.4%（BASE 11.0%），write queue 为 20.6%（BASE 29.6%）。

### 结果分析

**Workload 类型敏感性**：HM workloads 获益最大（31.2%），因为它们频繁访问 main memory，更能受益于 dynamic page mode + prefetching 的联合优化。LM workloads 获益较低（13.6%），因为其 memory 访问频率本身就较低。

**Epoch length 敏感性**（Section 7.7.1）：epoch = 100 比 epoch = 1000 仅提升 1.4%，但计算和能耗 overhead 显著增加；epoch = 10000 则性能下降 3.3%，因为无法及时适应 phase 变化。最终选择 epoch = 1000 作为性能和开销的平衡点。论文还给出了平均 inter-access interval：HM 567 cycles，LM 2556 cycles，MX 1845 cycles，这为 epoch = 1000 的选择提供了数据支撑。

**Prefetch buffer size 敏感性**（Section 7.7.2）：从 16KB 增至 32KB 和 64KB，MPDM 性能分别提升 2.1% 和 2.9%，收益递减明显。因此选择 16KB 以获得最佳 cost-performance ratio。

## 审稿人视角

### 优点

1. **问题建模清晰，动机充分**：论文通过 Fig. 2 的 intra-application diversity 数据有效论证了 static page mode 的不足，动机自然流畅。从 dynamic page mode 到 conflict-aware prefetching 再到 replacement policy 的技术路线逻辑连贯。

2. **Dynamic page mode 与 prefetching 的协同设计有工程意义**：将 page mode 决策与 prefetching 策略联动是一个合理且有效的设计选择——open page 阶段侧重 conflict-driven prefetching，close page 阶段侧重 utilization-driven prefetching，两者互补。

3. **Hardware overhead 极低**：总额外存储开销仅约 4.875KB（不含 prefetch buffer 本身），面积和能耗开销微乎其微，实用性强。

4. **实验较为全面**：包含了性能、延迟、conflict、prefetch accuracy、queue occupancy、energy 六个维度的评估，以及 epoch length 和 prefetch buffer size 的 sensitivity study。

### 不足

1. **实验平台和 DRAM 参数偏陈旧**：使用 DDR3-1600 参数（tRCD = tRP = tCL = 11 cycles），这与 2020-2021 年发表时已广泛使用的 DDR4/HBM2 参数有较大差距。HBM2 的 timing 特性（更低的 tRCD/tRP ratio、不同的 bank group 组织）可能会影响 threshold 选择和整体收益。论文声称方案适用于 HMC 和 HBM，但实验中未使用任何 HBM 的实际 timing 参数。

2. **Workload 评估的局限性**：仅使用 SPEC CPU2006/2017 的 multiprogramming mix，缺乏对 multi-threaded workload（如 PARSEC、SPLASH-2）和真实 data-intensive application（如 graph processing、deep learning inference）的评估。3D-stacked DRAM 的典型应用场景（GPU 加速、HPC）未被涉及。

3. **Threshold 选择的 sensitivity study 不足**：25%/50%/75% 的阈值被描述为 "empirically selected"，但论文未提供不同阈值设置下的 sensitivity analysis。考虑到这些阈值对整个方案的行为有决定性影响，这是一个重要的评估缺失。

4. **Conflict Table 的可扩展性问题**：CT 是 fully associative 结构且由 bank 共享（32 entries/channel），在 bank 数量更多或访存更密集的场景下，32 entries 是否足够？CT 的 contention 和替换行为对性能的影响未被分析。

5. **缺乏与 core-side prefetcher 协同的评估**：论文在 3.3.2 节提到 "our scheme can also work with other CPU-side prefetching schemes"，但实验中没有评估 MPDM 与任何 core-side prefetcher（如 stride prefetcher、SPP 等）共存时的表现。在实际系统中，memory-side prefetcher 几乎总是与 core-side prefetcher 共存的。

6. **Row 粒度 prefetching 的合理性未充分讨论**：将整个 row（1KB）prefetch 到 SRAM-based buffer，在 spatial locality 不佳的场景下会造成严重的 buffer 空间浪费。尽管论文通过 utilization-based replacement 缓解了这个问题，但缺乏对 sub-row prefetching 粒度的探索和比较。

7. **Prefetch buffer 到 processor 的数据传输策略过于保守**：论文明确表示 "does not aggressively push the prefetched data to the processor"，但对于何时以及如何将 prefetch buffer 中的数据推送到 processor-side cache 的策略描述不够详细。这个决策对整体 latency 有重要影响。

### 疑问或值得追问的点

- Close page mode 下的 PBHR（potential bank hit rate）通过 HR 追踪 "如果保持 open 会获得的 hit"，但这个计数方式忽略了 close page mode 本身对 request scheduling 和 bank availability 的影响。实际切换到 open page 后的 hit rate 是否真的与 PBHR 一致？
- UC + RC 的等权相加（Section 5.2）是否最优？是否考虑过加权方案或其他组合函数？
- 在 open page prefetching policy 中，CT 使用 LRU 替换，但论文为 prefetch buffer 设计了更精细的 replacement policy——为什么不在 CT 上也使用类似的 conflict frequency-based 策略？
- 论文的 address mapping 使用 Row:Rank:Bank:Channel:Column，这种 mapping 倾向于最大化 row buffer hit。是否评估过其他 address mapping（如 page interleaving）对 MPDM 效果的影响？
