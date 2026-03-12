---
title: "Accelerating Dependent Cache Misses with an Enhanced Memory Controller"
authors: "Milad Hashemi, Khubaib, Eiman Ebrahimi, Onur Mutlu, Yale N. Patt"
venue: "ISCA 2016"
year: 2016
---

# Accelerating Dependent Cache Misses with an Enhanced Memory Controller

## 基本信息

- **发表**：ISCA (International Symposium on Computer Architecture), 2016
- **作者单位**：UT Austin, Apple, NVIDIA, ETH Zürich & Carnegie Mellon University
- **关键词**：Dependent Cache Misses, Memory Controller, Near-Memory Computing, Pointer Chasing, Compute Migration

## 一句话总结

> 将 dependent cache miss 的依赖链迁移到 memory controller 端执行，绕过片上争用延迟，降低 20% 访存延迟。

## 问题与动机

### 核心问题：多核系统中 dependent cache miss 的延迟放大

论文瞄准的是多核处理器中一个被长期忽视的性能瓶颈：**dependent cache misses**——即一次 LLC miss 的返回数据是另一次 LLC miss 的地址来源（典型如 pointer chasing）。这类 miss 在 mcf 等应用中占比极高（mcf 约 40% 的 LLC miss 为 dependent miss），且由于两次 miss 的延迟被串行化，对性能影响巨大（mcf 的 IPC 仅为 0.3）。

### 为什么现有方法不够

1. **On-chip contention 严重放大了有效访存延迟**：论文通过 Figure 1 的 breakdown 展示，对于高 memory intensity 的应用（MPKI > 10），实际 DRAM access 时间不到总 miss latency 的一半，大部分延迟来自片上 interconnect、shared cache、DRAM bus 等共享资源的争用。这意味着 dependent miss 不仅要承受两次 DRAM 延迟，还要承受两次片上争用延迟。

2. **Prefetching 对 dependent miss 效果差**：论文评估了 GHB、Stream、Markov 三种 prefetcher，平均只能覆盖不到 20% 的 dependent cache misses（Figure 3）。根本原因在于 dependent miss 的地址是 data-dependent 的，模式难以被 prefetcher 捕获。更严重的是，不准确的 prefetch 会显著增加带宽消耗（Markov prefetcher 增加 42% 的带宽消耗），在带宽受限的多核系统中代价高昂。

3. **Pre-execution 技术（Runahead/CFP）不针对 dependent miss**：Runahead Execution 和 Continual Flow Pipelines 的核心思路是丢弃或延迟 dependent slice 来发掘新的 independent miss 的 MLP。它们本质上是在"绕过" dependent miss 而非"加速"它。

### 关键观察

论文的核心 observation 是：**source miss 和 dependent miss 之间的指令数通常很少**（Figure 6 显示平均在 2-10 条 uop 左右），且这些中间操作多为简单的整数算术（指针运算、地址计算）。这意味着如果能在 DRAM 数据到达芯片时立即执行这些短链操作，就可以避免数据回传 core 再重新发请求的片上延迟。

## 核心方法

### 关键思路

在 memory controller 端增加轻量级计算能力（Enhanced Memory Controller, EMC），当 core 检测到 dependent cache miss 可能发生时，自动从 ROB 中提取依赖链（dependence chain），迁移到 EMC 执行。EMC 在 DRAM 数据就绪后立即执行依赖链并直接发出后续 DRAM 请求，从而绕过片上 interconnect 和 cache hierarchy 的往返延迟。

### 技术细节

#### 1. EMC 计算微架构（Section 4.1）

EMC 的设计哲学是 **minimum functionality**——只实现执行 pointer-arithmetic 所需的最小功能集：

- **Front-end**：没有 fetch、decode、rename 硬件。仅包含 2 个 uop buffer（4-core 配置），每个可容纳最多 16 条 uop 的依赖链。多个 buffer 允许 EMC 被多核共享（8-core 配置为 4 个 context）。
- **Back-end**：2-wide issue，8-entry reservation station，支持 out-of-order issue 和 wakeup（通过 CDB 实现 tag broadcast）。关键设计选择是支持 **non-blocking memory access** 以利用依赖链内部的 MLP。包含小型 LSQ（8 entries）。
- **支持的指令集**：仅整数操作——add/subtract/move/load/store 和逻辑操作（and/or/xor/not/shift/sign-extend）。不支持浮点和向量操作。这既简化了微架构，又允许在生成链时过滤掉无关操作。
- **寄存器文件**：每个 context 配备 16-entry PRF 和 16-entry live-in vector。由于 core 的 PRF 远大于 EMC（256 vs. 16），需要在 core 端进行寄存器重映射（Register Remapping）。
- **Data Cache**：4KB, 4-way，2-cycle access，用于利用 temporal locality（如刚从 DRAM 返回的 cache line）。通过在 inclusive LLC 的 directory entry 中增加 1 bit 来维护 coherence。
- **TLB**：每个 core 对应 32-entry TLB（circular buffer），缓存最近被 EMC 访问的 PTE。不处理 page fault——遇到 TLB miss 则停止执行，通知 core 重新执行整条链。

**面积开销**：EMC 总面积约 2.2mm²（含 5.9KB 额外存储），仅为全功能 OoO core 面积的 10.4%，约占四核芯片总面积的 2%。

#### 2. 依赖链生成（Section 4.2）

这是论文最精巧的机制设计，**完全复用 core 已有的 OoO 执行基础设施**：

**触发条件**：当 ROB head 因 LLC miss 而 full-window stall 时，使用 3-bit saturating counter 判断 dependent cache miss 是否可能（有 dependent miss 则 increment，无则 decrement；top 2 bits 任一为 1 则触发）。

**链生成过程**（以 Figure 9 为例，共 5 cycle）：

- **Cycle 0**：处理 source miss。将其目的寄存器（C1）通过 Register Remapping Table (RRT) 映射到 EMC 物理寄存器（E0）。然后在 home core 的 CDB 上 **pseudo broadcast** C1 的 tag——这不是真正的执行或提交，只是模拟 wakeup 过程以发现依赖链。
- **后续 Cycle**：被 pseudo wakeup 的 uop 检查其所有 source operand 是否 ready 或已在 RRT 中。如果 ready，从 core PRF 读数据放入 live-in vector；如果在 RRT 中，则使用 EMC 寄存器编号。目的寄存器分配新的 EPR 并写入 RRT，然后继续 broadcast 以发现更深层依赖。
- **终止条件**：所有依赖操作已被识别，或达到最大链长度（16 uops）。
- **过滤**：只有生成 dependent cache miss 地址所需的操作才被包含在链中，无关操作被过滤掉。

**关键设计决策**：链生成只在 full-window stall 期间进行，此时 core 的执行资源基本空闲，因此 pseudo wakeup 过程不会干扰正常执行。同时，利用 core 已有的 CDB、RAT 等硬件，新增硬件极少（主要是 RRT 和 live-in vector）。

#### 3. EMC 执行（Section 4.3）

- **输入**：live-in source vector + 可执行的 uop chain。
- **Load 处理**：先查 EMC data cache；miss 则查 LLC 或直接发 DRAM 请求（通过 miss predictor——基于 PC 索引的 3-bit counter array 预测是否会 LLC miss，如果预测会 miss 则直接绕过 LLC 发 DRAM 请求）。
- **Store 处理**：仅包含 register spill 类型的 store（通过检查 home core LSQ 中是否存在对应的 fill load 来判定）。
- **Branch 处理**：EMC 接收的是 core 已经做过 branch prediction 的指令流。Branch direction 随链一起发送。EMC 可以检测 misprediction 但不能从正确路径重启——发现 mispredict 后停止执行并通知 core。
- **Memory ordering**：EMC 执行的 memory operation 通过 address ring 发消息回 core，core snoop 此请求并填充 LSQ entry。这既支持 memory disambiguation（发现冲突可取消链执行），又保证 store 按 program order 在 home core 的 store queue 中 drain 后才 globally visible。
- **异常处理**：任何 exceptional event（branch misprediction、TLB miss、exception）都导致 core 重新执行整条链，不使用 EMC。这是一种**投机执行+回退**的模型。

#### 4. 多 Memory Controller 支持（Section 4.4）

对于 8-core 配置，可以有多个分布式 EMC。跨 channel 的依赖（一个 EMC 产生的请求映射到另一个 EMC 管理的 channel）通过 EMC 间直接通信处理，不回退到 core，从而节省延迟。

### 与现有工作的区别

| 对比对象 | 关键差异 |
|---------|---------|
| **GHB/Markov Prefetcher** | Prefetcher 依赖历史模式预测未来地址，对 data-dependent 的地址无能为力且带宽低效。EMC 发出的是 demand request（不是 speculative prefetch），且通过实际执行计算得到精确地址。 |
| **Runahead Execution / CFP** | Runahead/CFP 丢弃 dependent slice 以发掘 independent miss 的 MLP。EMC 恰恰相反——它加速的是被 Runahead 丢弃的那些 dependent slice。两者正交互补。 |
| **PIM (Processing-in-Memory)** | 先前的 PIM 工作（如 PIM-enabled instructions, TOP-PIM）需要特殊的 3D-stacked memory 或修改 DRAM 工艺。EMC 不修改 DRAM，仅在 memory controller 端（仍在 CPU die 上）增加计算能力，且是自动透明的，不需要 ISA 修改或编程模型变化。 |

## 实验评估

### 实验设置

- **仿真平台**：In-house cycle-accurate x86 simulator，详细建模 core 微架构、cache hierarchy、non-uniform access latency DDR3 memory system。能耗建模使用 McPAT + CACTI。
- **硬件配置**：
  - Core：4-wide issue, 256-entry ROB, 92-entry RS, 3.2 GHz
  - L1: 32KB I-Cache + 32KB D-Cache, 3-cycle latency
  - L2/LLC: Distributed shared, 1MB/core, 18-cycle latency
  - Interconnect: 2 bi-directional rings (control 8B / data 64B)
  - DRAM: DDR3, 1 Rank/Channel, 8 Banks, 8KB Row, CAS 13.75ns, 800MHz bus; 4-core: 2 channels; 8-core: 4 channels
  - Memory Scheduler: Batch Scheduling (PAR-BS)
- **Workload**：SPEC CPU2006，按 MPKI 分为 high intensity (≥10) 和 low intensity (<10)。High intensity 包括 omnetpp, milc, soplex, sphinx3, bwaves, libquantum, lbm, mcf。随机生成 10 组四核异构 workload (H1-H10) + 8 组同构 workload。每个应用从 representative SimPoint 开始执行至少 50M 指令。
- **对比 Baseline**：
  - No prefetching
  - GHB G/DC prefetcher (1k-entry, 12KB)
  - Stream prefetcher (32 streams, distance 32)
  - Markov + Stream prefetcher (1MB correlation table)
  - 所有 prefetcher 配合 FDP 进行 throttling

### 关键结果

1. **性能提升**：在 H1-H10 四核 workload 上，EMC 平均性能提升 15%（vs. no-PF baseline）、**13%（vs. GHB prefetcher）**、10%（vs. stream PF）、11%（vs. Markov+stream PF）。mcf 同构 workload 提升最高达 30%。

2. **延迟降低**：EMC 发出的 cache miss 平均观察到的延迟比 core 发出的低 **20%**（Figure 18）。延迟节省来自三个方面：绕过 interconnect 回传 core 的路径、绕过 LLC lookup、降低 DRAM contention（其中 DRAM contention 降低贡献最大，但另外两个因素在部分 workload 中也很显著甚至占主导）。

3. **能耗降低**：EMC 在异构 workload 上平均降低 **11%** 能耗，同构 workload 降低 **9%**。而 prefetcher 反而增加能耗（Markov+Stream 增加带宽 52%）。EMC 仅增加 8% 的 memory traffic（同构仅 3%），远低于任何 prefetcher。

4. **可扩展性**：8-core 配置下 EMC 收益略高于 4-core（17% vs. 15% over no-PF），因为更多核意味着更严重的片上争用，EMC 的优势更明显。双 MC 配置由于 EMC 间通信开销，收益略低于单 MC 但差距很小（-0.8%）。

### 结果分析

**性能差异的解释**（Section 6.3，对比 H1 和 H4）：

论文识别出三个与性能提升正相关的统计量：
- EMC 生成的 miss 占总 miss 的比例（H1: ~10%, H4: ~22%）
- Row-buffer conflict rate 的下降幅度（H1: <1%, H4: 19%）
- EMC data cache hit rate（H1 远低于 H4）

**Row-buffer conflict 降低机制**：EMC 因为更快发出依赖请求，可以在竞争请求关闭 row 之前命中已打开的 row。两种场景：(1) dependent request 命中 source miss 同一 row（15% 的情况）；(2) 多个 dependent request 到同一 row 被 batch 在一起（85% 的情况）。

**Sensitivity analysis**：
- DRAM bandwidth sensitivity（Figure 20）：从 1C1R 到 4C4R，在低带宽（高争用）场景下 EMC 优势更大；即使在 4C4R 高带宽配置下仍有 11% 提升。
- EMC vs. Prefetching 交互（Figure 21）：GHB/Stream/Markov+Stream 分别仅覆盖 30%/21%/48% 的 EMC 所发请求，说明 EMC 在很大程度上补充了 prefetcher 无法预测的地址。

**效果不好的场景**：lbm 几乎没有 dependent cache misses，EMC 无用武之地；且 lbm 的规则访问模式几乎用满了带宽，prefetcher 表现更好。

## 审稿人视角

### 优点

1. **问题定义精准，动机充分**：论文非常清晰地量化了 on-chip delay 在总 miss latency 中的占比（Figure 1），以及 dependent cache misses 对性能的影响（Figure 2 中 mcf 若 dependent miss 全命中则性能提升 95%），再加上 prefetcher 对 dependent miss 的低覆盖率（Figure 3），三组数据构成了极有说服力的动机链条。

2. **设计哲学优雅**：EMC 的 "minimum functionality" 设计理念贯穿始终。不做通用 PIM，而是精准针对 pointer-arithmetic 这一特定计算模式；不新增 fetch/decode/rename 硬件，而是复用 core 的 OoO 基础设施生成依赖链；面积仅为 full core 的 10.4%。这种 "just enough" 的设计在工程上非常务实。

3. **链生成机制设计精巧**：利用 core 在 full-window stall 期间的空闲执行资源进行 pseudo wakeup，零额外执行硬件开销。RRT 进行寄存器重映射的设计避免了在 EMC 端维护大 PRF 的成本。整个过程对已有微架构的侵入性可控。

4. **实验全面且有深度**：不仅报告了性能数字，还通过 latency breakdown（Figure 18, 19）、row-buffer conflict 分析（Figure 16）、EMC cache hit rate（Figure 17）等多维度分析深入解释了性能提升的来源。DRAM bandwidth sensitivity study（Figure 20）证明了 EMC 的收益不能简单通过增加带宽/bank 来替代。

5. **与 prefetching 的正交性论证充分**：Figure 21 量化了 EMC 与各 prefetcher 的重叠覆盖率，证明两者互补而非替代。

### 不足

1. **Workload 代表性有限**：仅使用 SPEC CPU2006 单线程应用在多核上的拷贝组合。缺少真正的多线程共享内存应用（如 PARSEC、SPLASH-2），而这些应用中 dependent cache miss 的模式可能截然不同（如 concurrent data structure 的 pointer chasing）。此外，SPEC CPU2006 在 2016 年已略显陈旧，SPEC CPU2017 中的新应用可能提供更丰富的评估。

2. **对 chain 生成失败场景的分析不足**：论文没有详细分析以下场景的频率和影响：(a) 依赖链中包含不被 EMC 支持的操作（如浮点/向量）导致链截断；(b) EMC TLB miss 导致回退到 core 重新执行的比例和性能损失；(c) branch misprediction 导致 EMC 执行被取消的频率。这些 "失败路径" 的代价分析对于评估机制的鲁棒性至关重要。

3. **Memory consistency 和 ordering 讨论不够深入**：论文提到 EMC 执行的 store 必须在 home core 的 store queue 中按 program order drain 后才 globally visible，并类比了 TSX 的保证。但对于更复杂的 memory ordering 场景（如 EMC load 和其他 core 的 store 之间的 ordering）、以及在 relaxed memory model 下的正确性论证不够严谨。对于 ISCA 论文而言，这方面需要更形式化的正确性论证。

4. **单一 memory controller 位置假设**：4-core 配置下 EMC 位于 ring 上单一位置（类似 Haswell），但现代处理器（尤其是 server 级）的 MC 布局更复杂（多 MC、mesh interconnect）。虽然 Section 4.4 讨论了 dual MC，但评估中的 8-core dual MC 仅显示了微小差异，缺少对更大规模系统（16+ cores）的可扩展性分析——尤其是 EMC context 数量如何随核数增长。

5. **缺少与 Runahead Execution 的直接组合评估**：论文明确指出 EMC 与 Runahead/CFP 正交互补——EMC 加速 dependent miss，Runahead 加速 independent miss。但实验中没有评估两者组合的效果，这是一个明显的遗漏，读者自然会期望看到这个实验。

6. **EMC 的公平性和 QoS 问题未讨论**：EMC 是多核共享的资源（4-core 配置仅 2 个 context）。当多个 core 同时需要使用 EMC 时，如何仲裁？是否会导致某些 core 的请求被饿死？在 heterogeneous workload 混合中，这一问题可能显著影响公平性，但论文完全未涉及。

### 疑问或值得追问的点

- **EMC 在现代 HBM 系统中的价值如何？** HBM 的高带宽可能缓解 DRAM contention，但其 base die 上的逻辑层天然适合部署 EMC 类计算。EMC 的收益来源（绕过片上延迟 vs. 降低 DRAM contention）的相对重要性在 HBM 架构中可能发生显著变化。
- **链长度上限 16 uops 是否过于保守？** Figure 6 显示部分应用的 dependent chain 平均长度接近甚至超过 10。更长的链可能覆盖更多 dependent miss，但也意味着更大的 EMC 硬件开销和更高的回退代价。这个 trade-off 的 sensitivity 在论文中给出了参数选择但未展示详细的 sensitivity curve。
- **与现代 prefetcher（如 IPCP、Berti、SPP）的对比？** 论文发表时（2016）的 prefetcher baseline 已不代表当前 state-of-the-art。更先进的 prefetcher 对 irregular access pattern 的覆盖率更高，EMC 的增量收益可能需要重新评估。
- **对 LLM inference 中 KV cache 的 pointer chasing 模式是否适用？** 现代 LLM serving 中 KV cache 的访问模式包含大量 indirect/dependent access，EMC 的思路在 HBM-based accelerator 的 memory controller 中可能有新的应用场景。
