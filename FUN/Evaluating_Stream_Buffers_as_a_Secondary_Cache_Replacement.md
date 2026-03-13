---
title: "Evaluating Stream Buffers as a Secondary Cache Replacement"
authors: "Subbarao Palacharla, R. E. Kessler"
venue: "ISCA Workshop / IEEE 1994"
year: 1994
---

# Evaluating Stream Buffers as a Secondary Cache Replacement

## 基本信息
- **作者**：Subbarao Palacharla (University of Wisconsin-Madison), R. E. Kessler (Cray Research, Inc.)
- **发表**：IEEE, 1994
- **机构**：Wisconsin-Madison CS Department / Cray Research

## 一句话总结
> 提出用 stream buffer（FIFO prefetch buffer）替代昂贵的 L2 cache，并引入 filter 和 non-unit stride 检测两项增强技术，在科学计算负载上达到与 L2 cache 可比的 hit rate。

## 问题与动机

**核心问题**：1990年代的高性能处理器依赖大容量 off-chip L2 SRAM cache 来弥合处理器与主存之间的延迟鸿沟，但大容量 L2 cache 成本极高，尤其在大规模并行系统（1K+ 处理器）中，需要 GB 级 SRAM，造成巨大的经济负担。

**动机**：科学计算负载通常具有规则的、可预测的内存访问模式（streaming access、strided access），这意味着这些程序对 L2 cache 提供的 temporal locality 利用有限。相比之下，利用访问模式的规律性进行 prefetch 可能是更经济高效的方案。

**现有工作不足**：
- 基于 PC（program counter）的 stride prefetch 方案（如 Baer & Chen [1]）需要修改处理器微架构，无法直接用于 commodity processor。
- 软件 prefetch 方案需要额外指令执行开销，消耗处理器 pin bandwidth，且无法预测 conflict/capacity miss。
- Jouppi 原始的 stream buffer [10] 只处理 unit-stride 访问，对非单位步长访问无效；且没有机制控制无效 prefetch 带来的带宽浪费。

## 核心方法

### 关键思路

论文的核心 insight 是：对于科学计算负载，内存访问的**空间规律性（spatial regularity）**远比 **temporal locality** 更值得利用。Stream buffer 作为一种轻量级 FIFO prefetch 结构，可以以极小的硬件成本（每个 buffer 仅需一个 comparator、一个 adder 和少量 SRAM）捕获这种规律性，从而替代昂贵的 L2 cache。关键在于：(1) 通过 filter 机制减少投机 prefetch 的带宽浪费；(2) 通过地址空间分区（partition）实现 off-chip non-unit stride 检测。

### 技术细节

#### 1. 基本 Stream Buffer 结构

Stream buffer 是 FIFO 结构，每个 entry 包含：tag、valid bit、cache block data。此外配备一个 incrementer 用于生成下一个 prefetch 地址，一个 comparator 用于将 miss address 与 buffer head 的 tag 匹配。

**工作流程**：
- 当 on-chip cache miss 发生时，miss address 与所有 stream buffer 的 head 并行比较。
- **Hit**：数据从 stream buffer 传输到 primary cache，buffer 向后移动并继续 prefetch 后续 cache block。
- **Miss**：采用 LRU 策略淘汰最老的 stream，将其 flush 并重新分配给当前 miss address，从该地址开始 prefetch 连续的 cache block。
- Write-back 绕过 stream buffer 直接写回主存，并 invalidate stream 中可能存在的 stale copy。

**设计参数**：
- **Stream 数量**：论文实验表明 7-8 个 stream 即可使 hit rate 饱和，因为科学程序循环中通常访问有限数量的数组。
- **Stream depth（每个 stream 的 entry 数）**：固定为 2。Depth 主要取决于内存系统延迟——需要足够深以覆盖主存延迟并持续供给处理器数据。论文选择 depth=2 是为了尽量少做内存系统假设。

#### 2. Filter 机制（减少带宽浪费）

**问题**：原始 stream buffer 在每次 miss 时都会分配/重分配一个 stream，导致大量无效 prefetch。Extra Bandwidth (EB) 的计算公式为：

$$EB = \text{stream miss ratio} \times \text{depth}$$

例如 trfd 的 EB 高达 96%。

**Filter 设计**：维护一个包含 N 个 entry 的 history buffer（实验表明 8-10 个 entry 足够）。对于 miss address `a`，在 history buffer 中存储 `a+1`。只有当后续 miss 命中 history buffer 中的某个 entry 时（说明观察到了连续的 block `a` 和 `a+1` 的 miss），才分配 stream 从 `a+2` 开始 prefetch。否则，该 miss 被视为"孤立引用"（isolated reference），不触发 stream 分配。

**Filter 的双重效果**：
- 减少无效 prefetch 数量（直接减少 EB）。
- 防止活跃 stream 被孤立引用打断（间接提升 hit rate）。

加入 filter 后的 EB 公式变为：

$$EB = \text{stream miss ratio} \times \text{filter hit ratio} \times \text{depth}$$

**Trade-off**：Filter 需要至少观察两次连续 miss 才能确认 stream，因此对于 stream length 很短（<5）的程序会损失 hit rate。例如 appbt 中 63% 的 hit 来自 stream length < 5 的访问序列，filter 使 hit rate 从 65% 降至 45%。论文建议：若程序的带宽需求不高且内存系统能承受额外带宽，应关闭 filter。

#### 3. Non-unit Stride 检测方案

**问题**：fftpde、appsp、trfd 等程序包含大量非单位步长访问，原始 stream buffer 无法捕获。Off-chip 检测 stride 的难点在于无法获取 load/store 指令的 PC。

**方案：地址空间分区（Partition Scheme）**：
- 将物理地址分为两部分：高位的 **tag** 和低位的 **czone**（concentration zone，大小可由程序员/编译器在运行时通过 memory-mapped register 设置）。
- 维护一个 non-unit stride filter（history buffer），每个 entry 包含：partition tag、state bits、last address、stride。
- 使用一个 3-state FSM（INVALID → META1 → META2 → allocate stream）检测 stride：验证第三个地址与第二个地址的差等于第二个与第一个的差。
- Non-unit stride filter 放在 unit-stride filter **之后**，只处理 unit-stride filter miss 的引用，避免干扰常见的 unit-stride 检测。

**Czone 大小的 trade-off**：
- 太小：三次连续 strided reference 可能不在同一个 partition 内，导致检测失败。
- 太大：多个不同 stream 的引用可能落入同一个 partition，干扰 stride 检测。
- 最优大小约为 stride 的两倍多。对于 fftpde，czone 需 16-23 bits；对于 appsp 和 trfd，较大的 czone 即可。

### 与现有工作的区别

| 对比对象 | 关键差异 |
|---------|---------|
| **Jouppi 原始 stream buffer [10]** | 原始方案仅考虑从 L2 到 L1 的 prefetch，且只支持 unit stride，无 filter 机制。本文将 stream buffer 用于替代 L2（从主存直接 prefetch），并新增 filter 和 non-unit stride 检测。 |
| **Baer & Chen stride prefetch [1]** | 基于 PC 的 reference prediction table，需要修改处理器微架构。本文方案完全 off-chip，不需修改 commodity processor，但在 stride 检测能力上受限于只能观察物理地址。 |
| **Mowry et al. 编译器 prefetch [12]** | 软件 prefetch 需要编译器支持和额外的 prefetch 指令开销，消耗 pin bandwidth。本文是纯硬件方案，对软件透明。 |

## 实验评估

### 实验设置
- **仿真方法**：Trace-driven simulation。使用 Shade [17] 生成 primary cache miss 的地址 trace，输入到自定义的 stream buffer simulator。
- **Trace 采样**：Time sampling [11]，tracing on/off 周期为 10,000/90,000 references（采样率 10%）。
- **Cache 配置**：64KB I-cache + 64KB D-cache，4-way set associative，write-back + write-allocate，random replacement。
- **Workload**：15 个科学计算程序，来自 NAS suite（8个：embar, mgrid, cgm, fftpde, is, appsp, appbt, applu）和 PERFECT suite（7个：spec77, adm, bdna, dyfesm, mdg, qcd, trfd）。Fortran 代码通过 f2c 转为 C，使用 gcc 2.4.3 -O2 编译。数据集大小从 0.1 MB 到 14.7 MB 不等。
- **性能指标**：**Stream hit rate**（on-chip miss 中命中 stream buffer 的比例）作为主要指标，而非 execution time 或 CPI，理由是 hit rate 反映 stream 的最大潜在收益，且不绑定特定内存系统设计。
- **对比 baseline**：L2 cache（associativity 1-4，block size 64/128 bytes），使用 Set Sampling [11] 评估 hit rate。

### 关键结果

1. **基本 stream buffer 性能**：大多数 benchmark 的 hit rate 在 50%-80% 范围内，7-8 个 stream 即饱和。L2 cache 的 local hit rate 典型值为 70%-85%（通用应用），科学代码因缺乏 temporal locality 通常更低。Stream buffer 达到可比水平。

2. **Filter 效果**：对多数 benchmark，filter 将 Extra Bandwidth 减少 50% 以上，hit rate 损失很小或无损失。典型案例：trfd 的 EB 从 96% 降至 11%，hit rate 几乎不变；fftpde 的 EB 从 158% 降至 37%，hit rate 反而因防止活跃 stream 被干扰而提升。负面案例：appbt 的 hit rate 从 65% 降至 45%（因 63% 的 hit 来自短 stream）。

3. **Non-unit stride 检测**：对含大量非单位步长访问的程序效果显著——fftpde 的 hit rate 从 26% 提升到 71%，appsp 从 33% 到 65%，trfd 从 50% 到 65%。其他 benchmark 增益较小。

4. **与 L2 cache 的 scalability 对比**：随数据集增大，stream buffer 性能通常提升（如 applu 从 62% 到 73%），而达到相同 hit rate 所需的 L2 cache 容量成倍增长（applu 从 1MB 到 2MB）。这表明 stream buffer 对大数据集更具可扩展性。

### 结果分析

**性能好的场景**：访问模式规则、以 unit/constant stride 为主的科学计算（embar, cgm, mgrid, applu 等），stream length 较长（>20 的比例高）。

**性能差的场景**：
- **Array indirection（scatter/gather）**：adm、dyfesm 因大量间接数组访问导致 hit rate 低。
- **不规则稀疏矩阵**：cgm 在更大数据集上性能反而下降（从 85% 到 51%），因为更大的稀疏矩阵元素分布更不规则。
- **短 stream**：appbt 中大量 stream length < 5 的访问序列，filter 会伤害性能。

**Stream length 分布分析**（Table 3）是理解 filter 效果的关键：大部分 benchmark 的 hit 呈双峰分布——集中在 <5 和 >20 两个区间。Filter 对后者友好，对前者不利。

## 审稿人视角

### 优点

1. **问题定位清晰，实用价值高**：论文准确抓住了科学计算负载中 temporal locality 弱但 spatial regularity 强的特点，提出用轻量级硬件替代昂贵 L2 cache 的思路，对大规模并行系统有直接的成本效益意义。这一思路后来在 Cray T3D/T3E 等系统中得到验证。

2. **Filter 机制设计精巧**：history buffer 的设计（存储 `a+1` 而非 `a`）简单有效，以极低硬件开销（8-10 entry 的小表）实现了显著的带宽节省。论文对 filter 的 trade-off 分析（EB 公式推导、stream length 分布与 hit rate 损失的关联）逻辑严密。

3. **Benchmark 覆盖广**：15 个科学应用程序的评估在当时是相当全面的，且论文诚实地展示了 stream buffer 表现不佳的案例（adm、dyfesm、cgm 大数据集），没有选择性报告。

4. **Scalability 分析有前瞻性**：Table 4 中对数据集增大时 stream vs. L2 cache 的对比，清楚说明了 stream buffer 对科学计算的长期优势，这一结论在后续的大规模 HPC 系统设计中被反复验证。

### 不足

1. **性能指标选择过于保守**：仅使用 hit rate 作为评估指标，完全回避了对 execution time/IPC 的评估。论文虽然给出了理由（不想绑定特定内存系统设计），但 hit rate 无法反映 prefetch timeliness 的影响——stream buffer 的 hit 可能数据尚未从主存返回（论文在 Section 8 末尾承认了这一点），这与 L2 cache hit 的语义有本质区别。缺少对 prefetch latency coverage 的建模是一个重要缺陷。

2. **Stream depth 固定为 2 缺乏充分论证**：depth 是 stream buffer 最关键的设计参数之一，直接影响 latency hiding 能力和带宽消耗。论文以"尽量少做假设"为由固定 depth=2，但这使得 hit rate 结果的实际意义存疑——在真实系统中 depth 需要足够大以覆盖主存延迟，但更大的 depth 会放大 EB。论文缺少对 depth 的 sensitivity analysis。

3. **Non-unit stride 检测方案的实用性存疑**：czone 大小需要程序员或编译器设定，且对不同程序的最优值差异很大（fftpde 需 16-23 bits，appsp/trfd 则需较大值）。这增加了使用复杂度，且论文未讨论如何自动化这一过程。此外，czone 是全局设置，无法同时适配同一程序中不同 stride 的数组访问。

4. **实验方法论的局限**：
   - Trace-driven simulation 无法捕获 prefetch 对 memory-level parallelism (MLP) 和 memory controller scheduling 的影响。
   - 10% 的 time sampling 可能遗漏程序执行阶段变化（phase behavior）。
   - 数据集偏小（PERFECT suite 的 miss rate 极低，部分 <0.1%），论文自身也承认这一点。
   - 未考虑多核/多处理器场景下 stream buffer 与 coherence protocol 的交互。

5. **缺少对 irregular access pattern 的深入分析**：论文简单地指出 scatter/gather 模式下 stream buffer 效果差，但未探讨可能的改进方向（如结合 indirect prefetch 或 runahead 技术），也未分析 irregular access 在总 miss 中的占比与其对整体性能的影响权重。

### 疑问或值得追问的点

- Stream buffer depth=2 在实际 DRAM latency（~100ns 量级）下是否真的足够？如果 depth 增大到 4-8，EB 和 hit rate 的 trade-off 如何变化？
- Filter 的 history buffer 采用何种替换策略？论文未明确说明，但在高 miss rate 场景下替换策略可能显著影响 filter 效果。
- Non-unit stride detection 的 3-state FSM 只能检测 constant stride。对于 multi-level nested loop 产生的 non-constant stride 模式（如 tiling 后的访问），该方案完全失效。是否考虑过更灵活的 stride prediction 机制？
- 论文的目标系统是 Cray T3D 这样主存带宽远超处理器需求的系统。在主存带宽受限的 commodity 系统中，stream buffer 的 EB 开销是否会成为严重瓶颈？
- 论文未讨论 stream buffer 与 out-of-order execution 的交互——在乱序处理器中，多个 miss 可能同时 outstanding，stream buffer 的 FIFO 语义是否仍然合适？
