---
title: "Tackling Memory Access Latency Through DRAM Row Management"
authors: "Sriseshan Srikanth, Lavanya Subramanian, Sreenivas Subramoney, Thomas M. Conte, Hong Wang"
venue: "MEMSYS 2018"
year: 2018
---

# Tackling Memory Access Latency Through DRAM Row Management

## 基本信息

- **发表**：MEMSYS (The International Symposium on Memory Systems), 2018
- **作者单位**：Georgia Institute of Technology, Intel Labs
- **DOI**：10.1145/3240302.3240314

## 一句话总结

> 通过 global scoreboard 动态选择最优 timeout window 并结合 local row exclusion 机制，以极低硬件开销实现有效的 DRAM row buffer 管理。

## 问题与动机

### 要解决的问题

DRAM row buffer 管理策略（即 row 在最后一次 CAS 后应保持 open 多长时间）对访存延迟有重大影响。Open row policy 最大化 row hit 但引入 row conflict 开销；closed row policy 避免 conflict 但将本可命中的访问变为 row miss。Static timeout 是工业界的折中方案，但无法自适应不同 workload 的行为特征。

### 现有工作的不足

论文将 prior work 归纳为两个极端：

1. **简单但无效**：如 Park & Park [26] 的 2-bit 饱和计数器（per-bank 粒度的 smith predictor）、Jagtap et al. [13] 基于 pending queue 的 open/close 决策、Rokicki [28] 的概率性关闭策略。这些方法复杂度低，但无法捕捉应用的内存访问特征，性能提升有限。

2. **有效但复杂**：如 Xu et al. [37] 的 two-level access predictor（类似 branch prediction）、Khurshid et al. [17] 的 global history buffer、Awasthi et al. [1] 的 per-row access count tracking（ABP，需要 8192 entry 的追踪结构，~56KB 存储）、Stankovic et al. [34] 的 LDP（per-row 2-bit counter 做 liveness detection + per-bank dead time predictor，仅 liveness predictor 就需要 >256KB 存储）。这些方案在 per-row 粒度追踪行为，硬件开销高昂。

论文的核心动机是：**在性能和复杂度之间取得最佳平衡**，找到一种 simple yet effective 的 row management 策略。

### Limit Study

论文做了一个 perfect row close 的 limit study：假设可以在恰好正确的时机关闭 row（即既保证所有后续 row hit 被捕获，又在不同 row 的请求到来前恰好完成 precharge），相对 static timeout_50 baseline，平均性能提升达 8.2%。这表明 row management 问题确实有较大的优化空间。

## 核心方法

### 关键思路

两个核心观察：

1. **Global Observation**：如果能知道不同 timeout window 下的 row hit 和 row conflict 计数，就可以选择 "hit 增量减去 conflict 增量" 最大的 timeout window 作为全局最优。由于 ACT 和 PRE latency 量级相近（12-18ns），一个 miss→hit 的转换带来的收益和一个 miss→conflict 的转换带来的损失近似抵消，因此 `argmax(ΔHit - ΔConflict)` 是一个合理的优化目标。

2. **Local Observation**：少量 row 具有非常高的连续访问次数（如 Figure 4 所示的 long-tail 分布），全局 timeout 无法为它们提供足够长的 open 时间。对这些 row 应单独处理，保持 open 直到真正有不同 row 的请求到达。

### 技术细节

#### 1. Global Scoreboarding Mechanism

**核心结构**：per-bank 维护一个 scoreboard，包含 N 个 entry（默认 N=7），每个 entry 对应一个候选 timeout 值（50/100/150/200/300/400/800 cycles），记录该 timeout 下的 projected row hit count 和 row conflict count。

**Scoreboard 更新（Figure 6 流程图详解）**：

对每个到达 MC 的请求，针对 scoreboard 中的每个候选 timeout 值，投影该请求会是 hit/miss/conflict：

- **ACT 命令到达，且目标 row ≠ 先前 open row**（即 conflict 场景）：
  - 对于每个候选 timeout `t`，检查 `(CurrCycle - tRP - LastCASCycle) < t`。
  - 如果是（即在 timeout `t` 下 row 还未关闭），则该 timeout 下这次访问仍是 conflict → `conflicts[t]++`。
  - 如果否（row 在该 timeout 下已经被关闭），则这次访问变成 miss（避免了 conflict）。

- **ACT 命令到达，且目标 row == 先前 open row**（即 row 被关闭后又重新打开同一 row）：
  - 对于更大的 timeout 值，row 可能还没被关闭，此次访问可以变成 hit → `hits[t]++`（miss→hit 转换）。

- **CAS 命令到达**：
  - 如果是之前 ACT 标记的 hit 类型，且 timeout ≥ 当前 timeout，保留 hit 状态。
  - 如果 `(CurrCycle - LastCASCycle) < t`，在该 timeout 下仍可 hit → `hits[t]++`。
  - 否则标记为 miss。

**Timeout 选择（每 N=30000 个请求更新一次）**：

```
nextT = argmax_t (hitsIncr[t] - conflictsIncr[t])
```

其中增量是相对当前 timeout 计算的。引入 variation threshold（3%）防止频繁切换：只有当最优值比最差值有超过 3% 的相对差异时才更新 timeout。

**关键设计选择**：
- Timeout 间隔设计：考虑到 ACT/PRE latency 约 25-30 cycles，timeout 间隔至少 50 cycles 是合理的。后续加入更大的跳跃值（300/400/800）以覆盖更多样化的行为。
- 更新窗口 30000 requests：足够长以采集统计显著的数据，又不至于响应太迟。

#### 2. Local Row Exclusion Mechanism

**动机**：Figure 4 的 CASes-per-ACT 直方图表明少量 row activation 伴随极高的连续 CAS 数（>100），全局 timeout 即使很大也可能不足。

**检测机制**：
- 追踪每个 bank 的 "previous open row" 和 "是否因 timeout 过期被关闭"。
- 当一次 ACT 的目标 row == 刚刚因 timeout 过期被关闭的 row 时，将该 row 加入 row exclusion store（64 entries）。

**豁免机制**：
- 被标记的 row 在 timeout 过期时不关闭，而是保持 open 直到有不同 row 的请求到达同一 bank 为止（等价于对这些特定 row 使用 open row policy）。

**替换策略**：
- 当 exclusion store 满时，优先替换 "因 row exclusion 导致了最近一次 row conflict" 的 entry。即：如果某个 row 被豁免后反而造成了 conflict，说明 exclusion 对它无效，优先淘汰。

**Row Aliasing 探索**：
- 方案 i：追踪完整地址（channel + rank + bank + row address）→ 20 bits。
- 方案 ii：仅追踪 row address（16 bits），利用 address interleaving 下同一物理页可能分布在多个 bank 的相同 row 这一特性。
- 实验结果：无 aliasing（完整地址）效果更好（Table 3）。

### 硬件开销分析

**Scoreboard**：每个 entry 5 bytes（8-bit timeout + 16-bit hit count + 16-bit conflict count），7 entries × 16 banks = 560 bytes；加上辅助状态（previous open row address 16-bit, last CAS cycle 32-bit, next CAS type 7-bit per bank）= 110 bytes。**总计 670 bytes**。

**Row Exclusion Store**：64 entries × (20-bit address + 6-bit replacement counter + 1-bit closure type) per channel × 2 channels ≈ **432 bytes**。

**两者合计约 1.1 KB**，不到 LDP 方案 256KB 的 1/250。

### 与现有工作的区别

| 方案 | 方法特点 | 存储开销 | 平均加速 |
|------|---------|---------|---------|
| **Smith** [26] | Per-bank 2-bit 饱和计数器，预测 hit/conflict 做二元决策 | 极低（~2 bits/bank） | 低 |
| **ABP** [1] | Per-row 追踪上次 open 时的 hit 数，下次重复同样次数 | ~56 KB（8K entries/channel） | 中等，但个别 workload 严重退化 |
| **LDP** [34] | Per-row 2-bit liveness counter + per-bank dead time predictor | >256 KB | 最高（prior work 中） |
| **本文** | Global scoreboard（多 timeout 投影）+ local row exclusion（64 entries） | ~1.1 KB | ≥ LDP，且无退化 |

**关键差异**：
- Smith 只做 per-bank 的二元预测，无法捕捉 local 行为差异，也无法在多个 timeout 候选中做全局最优选择。
- ABP 在 per-row 粒度追踪行为，开销大且依赖上次访问模式的重复性假设，对 phase change 敏感。
- LDP 的 per-row liveness counter 提供了精细的 local 信息，但存储代价不可接受。本文的 row exclusion 仅对需要特殊处理的少量 row 追踪，用 64 entries 达到类似效果。

## 实验评估

### 实验设置

- **仿真平台**：Ramulator [20]（扩展实现 scoreboard 和 row exclusion），搭配简单 OoO core frontend，由 Pin [23] 驱动的 trace。
- **硬件配置**：
  - 4-wide fetch/retire, 128-entry ROB
  - Cache：L1 32KB/8-way, L2 256KB/8-way, L3 2MB/8-way（non-inclusive, LRU）
  - Load-to-use latency：L1 4c, L2 16c, L3 47c
  - 16 MSHRs per cache
  - Core:Memory 频率比 = 8:3
  - DRAM：LPDDR4, 2 channels, 1 rank/8 banks each
  - Address interleaving：RoBaRaCoCh（channel interleaving）
  - Scheduling：FR-FCFS with hit prioritization
  - Prefetch：关闭
- **Workload**：SPEC CPU 2006 single-core traces；multi-programmed 2-core mixes（按 memory intensity 分类为 MM, MN, NN 组合）
- **对比 Baseline**：
  - Static timeout_50（主 baseline）
  - Closed row policy
  - timeout_100, timeout_200
  - Open row policy
  - Smith [26]
  - ABP [1]
  - LDP [34]

### 关键结果

1. **Scoreboard + Row Exclusion vs. Static Timeout_50**：memory intensive workloads 平均加速 6.79%，全部 benchmark 平均 3.95%（Figure 7）。

2. **Scoreboard 单独 vs. Static Timeout_50**：memory intensive workloads 平均加速 6.34%，全部 benchmark 平均 3.35%。Row exclusion 贡献了额外的 ~0.45%/0.6% 增量（Table 3）。

3. **与 Prior Work 对比**（Figure 8, 9）：
   - 本文方案性能与 LDP 持平或略优，但硬件开销仅为 LDP 的 1/250。
   - ABP 在某些 workload 上出现高达 9.39% 的性能退化（如 xalancbmk），而本文方案的最差 workload 退化仅 3.92%（MIN%）。
   - Smith 性能提升最低，验证了单纯 per-bank 全局预测的局限性。

4. **多核评估**（Figure 10）：2-core multi-programmed workloads 上，weighted speedup 最高达 ~25%（MM 组合），验证了方案在竞争场景下仍然有效。

### 结果分析

**Sensitivity Analysis**：

- **Memory 配置影响**（Figure 11）：增加 channel 数（2→4）不影响机制有效性，speedup 比例类似。但使用 row interleaving（ChRaBaRoCo）时由于 MLP 大幅下降，整个系统变慢，idle row closure 机制的影响也相应减弱。
- **Row Exclusion 细节**（Table 3）：无 aliasing > 有 aliasing > 无 row exclusion，差距不大但一致。
- **替换策略**：淘汰 "最近因 exclusion 造成 conflict" 的 entry 效果最佳。
- **Scoreboard 参数**：variation threshold、update window、entry 数量的变化仍优于其他方案，但论文未给出详细敏感性数据（称 "do not provide further insight"）。

**效果好的场景**：memory intensive workloads（如 mcf 达 47.8% speedup under open policy 说明 row locality 极强），尤其是 row buffer locality 在不同 phase 间变化较大的 workload。

**效果有限的场景**：memory non-intensive workloads（右侧 benchmark 在 Figure 7 中几乎无变化），因为访存不是瓶颈。重要的是，本文方案在这些场景下也不会造成退化（最差 -3.92%，与 open/closed 极端策略的退化相比已很温和）。

## 审稿人视角

### 优点

1. **问题定位精准，工程价值高**。Row buffer management 是内存控制器设计中的实际工程问题，工业界普遍使用 static timeout 正说明此前的学术方案复杂度过高无法落地。本文在 ~1KB 开销下达到甚至超过 256KB 方案的性能，这个 cost-performance tradeoff 的改进是实质性的。

2. **两层机制设计互补且自洽**。Global scoreboard 捕捉整体趋势，row exclusion 处理 long-tail 特例，这种 "全局+局部" 的分层思路清晰合理。Row exclusion 的检测条件（timeout 过期关闭后立即重新 ACT 同一 row）简洁有效，不需要复杂的 pattern matching。

3. **Scoreboard 的 "投影" 机制设计巧妙**。不需要实际运行在不同 timeout 下才能获得 hit/conflict 统计，而是通过时间戳比较为每个请求同时投影多个 timeout 下的结果，这使得在线评估多个候选 timeout 成为可能且开销极低。

4. **实验对比全面，覆盖了 prior work 的多个代表性方案**，并且给出了 Pareto 图（Figure 9）直观展示 performance-area tradeoff。

### 不足

1. **仿真平台和 workload 的代表性存疑**。
   - Ramulator 搭配的是简单 OoO frontend（非 cycle-accurate full-system simulator），prefetch 关闭，这在现代处理器中极不现实。Prefetcher 会显著改变 row buffer 访问模式（增加 row hit rate、改变时间分布），关闭 prefetch 可能夸大了 row management 优化的效果。
   - 仅使用 SPEC CPU 2006 single-core traces，缺乏多线程 workload（如 PARSEC、SPLASH-2）、server workloads、以及 streaming/irregular memory access pattern 的评估。2-core multi-programmed 实验虽然有，但仍是 single-thread trace 的组合，不能代表真正的 shared-memory 多线程行为。

2. **Timeout 候选集是手工选择的**。50/100/150/200/300/400/800 这组值的选择缺乏系统性论证。虽然论文提到 "间隔至少 50 cycles"，但为何选择这 7 个具体值、是否有更优的候选集、候选集大小 N 的影响等均未充分探讨。论文声称参数敏感性实验 "不提供额外 insight" 而省略了详细数据，这对读者来说不太令人满意。

3. **Row exclusion 的检测机制可能存在 cold-start 和 hysteresis 问题**。
   - 一个 row 必须先经历一次 "timeout 关闭 → 重新 ACT" 才能被加入 exclusion store，这意味着第一次 burst 访问的起始阶段不可避免地会损失 hit。
   - 论文没有讨论 row 从 exclusion store 中退出的时机（除了被替换）。如果一个 row 的访问模式发生变化，它可能长期占据 exclusion store 而实际不再需要 exclusion 保护。
   - 64 entries 的 exclusion store 在多 bank（16 banks）共享时，平均每 bank 仅 4 entries，对于某些 workload 可能不足。

4. **LPDDR4 配置的通用性**。论文仅评估了 LPDDR4（2ch/1ra/8ba），未涵盖 DDR4/DDR5 的典型服务器配置（多 rank、bank group 约束、不同的 timing parameter 比例关系）。Bank group 对 row management 的影响可能不同——DDR4/5 的 bank group 内不同 bank 有额外的 tCCD_L 约束，可能影响 timeout 选择的最优值。

5. **评估指标单一**。论文仅报告了 runtime/speedup，未讨论：
   - 对 DRAM 功耗的影响（row 保持 open 更久意味着更高的 leakage，频繁 ACT/PRE 也消耗能量）。
   - 对 fairness 的影响（在多核场景下，某些应用的 row 被长时间 open 可能 starve 其他应用的请求）。
   - 对 refresh 干扰的影响（row 长时间 open 可能推迟 REF，影响 tREFI 遵从性）。

6. **缺少与 FRFCFS scheduling 的交互分析**。Baseline scheduler 是 FR-FCFS with hit prioritization，这本身就倾向于将同 row 的请求聚集调度。Row management 和 scheduling policy 之间存在耦合——如果 scheduler 已经充分 reorder 了请求来最大化 row hit，那么 timeout 管理的边际收益可能更小。论文未隔离和分析这一交互效应。

### 疑问或值得追问的点

- **Scoreboard 的投影准确性在 out-of-order scheduling 下如何？** 由于 MC scheduler 可以重排请求，一个请求在 scoreboard 投影时计算的 "距上次 CAS 的时间差" 可能因 scheduling 重排而与实际不同。论文似乎假设请求到达顺序即服务顺序，但 FR-FCFS 会优先调度 row hit 请求，这可能导致投影偏差。
- **Row exclusion store 是 per-channel 还是 per-bank？** 论文说 "maintained at a channel granularity"，那么同一 channel 下不同 bank 的 hot row 共享 64 entries，是否会因 bank 间不均衡而导致某些 bank 的 hot row 被挤出？
- **Update window 30000 requests 的粒度是否合适？** 对于 memory intensive workload（如 mcf），30000 requests 可能仅覆盖很短的执行窗口；而对于 non-intensive workload，30000 requests 可能跨越多个 program phase。按请求数而非时间/instruction 数来划分窗口是否最优？
- **与 DRAM refresh 的交互**：在 REF 命令到来时，所有 bank 的 row 必须关闭。这对 scoreboard 的统计和 row exclusion 的有效性有何影响？论文完全未提及 refresh。
