---
title: "Impulse: Building a Smarter Memory Controller"
authors: "John Carter, Wilson Hsieh, Leigh Stoller, Mark Swanson, Lixin Zhang, et al."
venue: "HPCA 1999"
year: 1999
---

# Impulse: Building a Smarter Memory Controller

## 基本信息
- **发表**：HPCA 1999（第五届 IEEE High-Performance Computer Architecture 国际研讨会）
- **单位**：University of Utah / Intel Corporation

## 一句话总结

> 在 memory controller 中增加可配置的物理地址 remapping 和 prefetching，无需修改 CPU/cache/DRAM 即可改善 memory-bound 应用性能。

## 问题与动机

**核心问题**：1999 年前后，处理器性能以每年约 60% 的速度增长，而 DRAM latency 每年仅改善约 7%，bandwidth 每年改善约 15-20%。这一 processor-memory performance gap 导致许多缺乏足够 locality 的重要应用（如 sparse matrix、database、signal processing、multimedia、CAD）无法有效利用处理器算力。

**现有方案的不足**：当时已有多种硬件方案被提出来缓解 memory wall 问题，包括 configurable cache [25, 26]、processor-in-memory (PIM) [14, 18, 24]、stream buffer [13, 16] 等。然而这些方案几乎都需要对处理器、cache 或 DRAM 进行重大修改，实际落地极为困难。具体而言：

- PIM 方案（如 IRAM [14]、RADram [18]）需要在 DRAM die 上集成处理逻辑，而 DRAM 工艺为容量优化而非速度优化，处理能力受限。
- Morph [26] 将可编程逻辑嵌入几乎所有 datapath，灵活但设计复杂度极高。
- Stream buffer [13, 16] 仅支持 regular stride 访问模式，无法应对 irregular 应用。
- Yamada [25] 在处理器端做 relocation，无法节省 bus bandwidth，也不能改善 L2 cache 利用率。

**Impulse 的定位**：将"智能"集中放在 memory controller 这一单一修改点上，不需要改动 CPU、cache 或 DRAM 设计，因此可被现有通用系统直接采用。

## 核心方法

### 关键思路

Impulse 的核心洞察是：系统中存在大量未被 DRAM 支撑的"空闲"物理地址空间（即 shadow address space），可以利用这些地址作为"别名"，在 memory controller 中进行一次额外的地址翻译，将 shadow address 映射到真实的物理数据上。通过这种 remapping，应用程序可以控制数据在 cache line 中的排布方式，从而改善 bus utilization 和 cache hit rate，而整个过程对处理器完全透明。

### 技术细节

#### 1. 整体架构

Impulse 在传统 memory controller 基础上增加了两大功能模块：

**(A) Shadow Address Remapping**

处理器发出的地址到达 memory bus 后，Impulse controller 首先判断该地址是普通物理地址还是 shadow address：
- 若为普通物理地址，直接传递给 DRAM scheduler，不增加额外延迟（这是关键设计目标）。
- 若为 shadow address，经过以下流水线处理：
  1. **Shadow Descriptor (SDesc) 匹配**（步骤 b）：从 8 个 shadow space descriptor 中选择匹配的描述符。虽然模型中配置了 8 个 SDesc，实际实验中最多只用到 3 个。
  2. **Address Calculation (AddrCalc)**（步骤 c）：使用简单 ALU 将 shadow address 根据 SDesc 中存储的 remapping 信息，翻译为 pseudo-virtual address。
  3. **Page Table Lookup (PgTbl)**（步骤 d）：将 pseudo-virtual address 通过 on-chip TLB（backed by main memory）翻译为真实物理地址。引入 pseudo-virtual space 是为了支持跨越多个 page 的数据结构 remapping，同时节省地址位数。
  4. **DRAM Scheduling**（步骤 e/f）：物理地址被传递给 DRAM scheduler 进行实际 DRAM 访问。
  5. **Data Assembly**（步骤 g/h）：SDesc 将从多次 DRAM 读取中收集到的数据组装成一个完整的 cache line，发送回 bus。

**(B) Controller-Based Prefetching**

- 对于 non-remapped data：controller 内含一个 **2KB SRAM prefetch buffer**，使用简单的 one-block lookahead prefetcher。
- 对于 remapped (shadow) data：每个 shadow descriptor 配有一个 **256-byte buffer** 用于 prefetch shadow memory，以隐藏 scatter/gather 操作的多次 DRAM 访问延迟。

#### 2. 支持的 Remapping 类型

Impulse 当前支持三种 shadow-to-physical remapping 函数：

| Remapping 类型 | 映射关系 | 典型用途 |
|---|---|---|
| **Direct mapping** | shadow page → physical page（1:1） | 物理页重着色（page recoloring）、构建 superpage |
| **Strided mapping** | `soffset → pvaddr + stride × soffset` | Tile remapping、矩阵对角线提取 |
| **Scatter/gather (indirection vector)** | `soffset → pvaddr + stride × vector[soffset]` | 稀疏矩阵间接寻址优化 |

为保持硬件简单高效，Impulse 对 remapping 施加了限制：strided mapping 中对象大小必须是 2 的幂次（避免在 controller 中实现除法器）。

#### 3. 软件接口与使用流程

以矩阵对角线 remapping 为例，完整的使用流程为：
1. 应用分配一段连续虚拟地址空间用于映射对角线元素（即新变量 `diagonal`）。
2. OS 从不对应真实 DRAM 的物理地址池中分配一段连续 shadow address。
3. OS 向 memory controller 下载 mapping function（base + stride）。
4. OS 向 memory controller 下载 pseudo-virtual space 到 physical space 的 page mapping。
5. OS 将虚拟别名 `diagonal` 映射到 shadow memory，flush 原地址在 cache 中的数据，返回。

当前阶段由手工修改应用内核完成系统调用；论文提到正在探索类似 vectorizing compiler 的编译器算法来自动化。

#### 4. 三种优化应用

**(A) Scatter/Gather for SMVP**

针对 sparse matrix-vector product (SMVP) 中通过 indirection vector `COLUMN[]` 间接访问向量 `x` 的问题：
- 传统方式：每次间接访问 `x[COLUMN[j]]` 都要加载一整个 cache line，其中只有一个元素有用。
- Impulse 方式：建立 shadow alias `x'`，使得 `x'[k] = x[COLUMN[k]]`，memory controller 负责 gather 操作，将多个有用元素打包进一个 cache line。

性能提升来源：(1) L1 cache spatial locality 大幅提升（hit rate 从 64.6% 升至 88.0%）；(2) 处理器不再需要发出 indirection load（减少总 load 指令数）。

代价：L2 cache temporal locality 下降，因为 remapped 后的 `x'` 元素地址各不相同，无法在 L2 中复用。

**(B) Page Recoloring for SMVP**

利用 direct mapping 对物理页进行重着色，消除 L2 cache 中的 conflict miss：
- 将 `x` 向量着色到 L2 cache 的前一半。
- 将 `DATA` 和 `COLUMN` 分别着色到 L2 后半部分的两个象限。
- 效果上将 L2 cache 的一小部分用作 `DATA`、`ROWS`、`COLUMNS` 的 stream buffer。

传统系统中物理页重着色需要数据拷贝，Impulse 通过 shadow mapping 实现零拷贝重着色。

**(C) Tile Remapping for Dense Matrix-Matrix Product**

针对 tiled 矩阵乘法中 tile 在虚拟地址空间不连续导致的 cache conflict 问题：
- 使用 base-stride remapping 将非连续的 tile 映射到连续的 shadow space。
- 将 L1 cache 划分为三个段，分别放置 C tile（输出）、A tile 和 B tile。
- 切换 tile 时通过 Impulse remapping 完成，仅需 purge/flush cache，无需数据拷贝。

约束条件：tile size 必须是 cache line 的整数倍（128 bytes）；数组需要 padding 使 tile 对齐到 128 bytes。

### 与现有工作的区别

| 对比对象 | 关键差异 |
|---|---|
| **Morph [26]** | Morph 在几乎所有 datapath 中嵌入可编程逻辑，灵活但极为复杂。Impulse 将修改局限于 memory controller，设计远为简单，可被当时的现有架构直接采用。 |
| **Stream Buffer (McKee et al. [16], Jouppi [13])** | Stream buffer 仅支持 regular stride 模式；Impulse 额外支持通过 indirection vector 的 scatter/gather，可处理 irregular 应用。同时 Impulse 在 controller 端做 remapping 可节省 bus bandwidth，stream buffer 做不到。 |
| **Yamada [25]** | Yamada 在处理器端做 relocation + prefetch 到 L1 cache，不节省 bus bandwidth，也无法改善 L2 cache 利用率。Impulse 在 memory controller 端做 remapping，bus 上传输的是已经打包好的 dense cache line。 |

## 实验评估

### 实验设置

- **仿真平台**：Paint simulator [20]，模拟一个 HP PA-RISC 1.1 处理器及 BSD 微内核。
- **硬件规格配置**：
  - 处理器：120 MHz，single-issue，HP PA-RISC 1.1
  - Bus：120 MHz HP Runway bus
  - L1 data cache：32KB，direct-mapped，virtually-indexed / physically-tagged，write-back / write-around，non-blocking，32-byte line，1 cycle hit latency
  - L2 data cache：256KB，2-way set-associative，physically-indexed / physically-tagged，write-allocate / write-back，128-byte line，7 cycle hit latency
  - Memory access latency：40 cycles
  - TLB：unified I/D，fully associative，NRU replacement
  - Impulse controller：2KB SRAM prefetch buffer（non-remapped data），每个 SDesc 256-byte buffer（remapped data），8 个 shadow descriptor
  - DRAM scheduler：in-order（scheduler 设计尚未完成，论文中使用简单顺序调度）
  - Instruction cache：assumed perfect
- **Workload**：
  - NAS Class A Conjugate Gradient (CG-A)：14000×14000 稀疏矩阵，约 200 万非零元素
  - Dense matrix-matrix product：512×512 矩阵，32×32 tile
- **对比 Baseline**：
  - Conventional memory system（无任何优化）
  - Conventional + Impulse controller prefetching only
  - Conventional + L1 cache next-line prefetching
  - Conventional + both prefetching
  - Impulse scatter/gather / page recoloring / tile remapping × 各种 prefetching 组合

### 关键结果

1. **Scatter/gather + prefetching 在 CG-A 上达到 1.67× speedup（67% 性能提升）**：L1 hit rate 从 64.6% 提升至 88.0%（无 prefetch），配合 controller prefetching 后 avg load time 从 5.24 cycles 降至 3.53 cycles。最佳配置（scatter/gather + 两种 prefetch 均开启）speedup 达 1.95×。

2. **单独的 controller-based prefetching 在 CG-A 上仅提升 4%**，而 L1 cache next-line prefetching 提升 12%。但 controller prefetching 无需修改处理器核心，对不支持硬件 prefetch 的处理器有价值。两者组合时效果最好（1.13×）。

3. **Page recoloring 在 CG-A 上提升较温和**：无 prefetch 时 speedup 1.04×，加 controller prefetch 后 1.09×，加两种 prefetch 后 1.19×。主要通过消除 L2 cache conflict miss，将 memory hit ratio 从 5.5% 降至 4.4%。

4. **Tile remapping 在矩阵乘法上达到 1.98×–2.01× speedup**，L1 hit rate 从 49.0% 飙升至 99.4%，avg load time 降至约 1 cycle。与 software tile copying（speedup 1.95×）相当或略优，但无需实际数据拷贝。

### 结果分析

**Scatter/gather 的收益来源拆解**：论文指出约 1/3 的 cycle 节省来自减少 load 指令数（indirection load 被移到 controller 端），其余来自 L1 hit rate 大幅提升。L2 hit rate 实际是下降的（从 29.9% 降至 4.4%），因为 remapped 元素地址各异不可复用，但总体性能仍显著提升。

**Prefetching 在不同场景下的效果差异**：
- 对于 scatter/gather 场景，prefetching 效果显著，因为 gather 需要多次 DRAM 访问来填充一个 cache line，prefetch 可以有效隐藏这些延迟。
- 对于 tile remapping 场景，由于 copying/remapping 本身已将 L1 hit rate 推至 ~99%，prefetching 几乎无差异。
- 对于 conventional 矩阵乘法，L1 cache prefetching 反而轻微降低性能（speedup 0.996×），因为极低的 L1 hit rate 导致 prefetch 在 L2 cache 产生过多 contention。

**论文未做 sensitivity analysis**：没有对 prefetch buffer 大小、SDesc 数量、cache 大小、DRAM latency 等关键参数进行 sensitivity study。

## 审稿人视角

### 优点

1. **修改点集中且实用性强**：将所有硬件修改限制在 memory controller 这一处，不需要动 CPU、cache 或 DRAM 设计，这在当时（乃至现在）都是非常有吸引力的工程约束。这种"最小侵入"的设计哲学使得方案在工业界采纳的门槛很低。

2. **Shadow address 的概念设计优雅**：利用已有物理地址空间中的空闲部分作为 remapping 的中介，无需引入新的硬件地址空间或修改 ISA，是一个巧妙的 insight。将 OS、应用程序和 memory controller 三者协同工作的软硬件接口设计得比较清晰。

3. **覆盖面较广的优化能力**：scatter/gather、page recoloring、tile remapping 三种不同优化覆盖了 irregular 和 regular 两类 memory access pattern，说明了架构的通用性。且论文还提到了 superpage 构建 [21] 和 IPC 优化等潜在应用场景。

4. **对非 remapped 访问零开销的设计目标**：明确提出且在架构中体现了 non-shadow 地址不增加延迟的设计原则，避免了对其他程序的负面影响。

### 不足

1. **Workload 覆盖严重不足**：仅评估了 CG-A 和 512×512 矩阵乘法两个 kernel，而非完整应用。论文声称可以优化 database、multimedia、CAD 等应用，但未提供任何此类 workload 的评估数据。作为一个声称通用性的架构，两个 scientific kernel 的评估远不够有说服力。

2. **DRAM scheduler 设计未完成**：论文明确承认 DRAM scheduler 设计尚未完成，实验使用的是简单的 in-order scheduler。这意味着论文所报告的性能数据未能反映 scatter/gather 产生的 irregular DRAM access pattern 对 DRAM bank/row buffer locality 的实际影响。对于一篇以 memory controller 设计为核心贡献的论文，DRAM scheduler 是极为关键的组件，其缺失是一个重大弱点。

3. **仿真平台过于简化**：single-issue、120 MHz 处理器、256KB L2 cache 即便在 1999 年也算偏低端。论文自己也提到 superscalar 上 speedup 应该更大，但这只是推测。non-blocking cache 的建模细节（MSHR 数量等）不够清晰，instruction cache 假设为 perfect 也不现实。

4. **缺乏面积/功耗/时序分析**：论文未讨论 Impulse controller 增加的硬件开销，包括 SDesc 存储、AddrCalc ALU、PgTbl TLB、prefetch SRAM buffer 的面积成本，以及这些额外逻辑是否会影响 memory controller 的时钟频率。

5. **软件使用复杂度被低估**：当前需要手工修改应用代码来插入系统调用，编译器自动化"正在探索"但无结果展示。Cache flush/purge 的开销在 tile remapping 场景中被声称"很小"，但实际在多核/多线程场景下（论文未涉及）可能成为严重瓶颈。data consistency 完全依赖应用/编译器通过 cache flush 保证，这在实际使用中非常容易出错。

6. **未考虑多核/多处理器场景**：在 SMP/NUMA 系统中，shadow address 的 coherence 问题、多个处理器同时使用 Impulse remapping 时的资源竞争问题（仅 8 个 SDesc）完全未被讨论。

### 疑问或值得追问的点

- Scatter/gather 导致 L2 hit rate 从 29.9% 降至 4.4%，论文将此归因于 remapped 元素地址不可复用。但是否存在通过更智能的 shadow address 分配策略来部分保留 L2 reuse 的可能？即是否可以将 scatter/gather 与 page recoloring 结合使用？
- DRAM scheduler 使用 in-order 策略时，scatter/gather 产生的多次 random DRAM 访问的实际 latency 可能被严重低估。论文报告的 40 cycles memory latency 是否反映了 row buffer miss 的代价？如果 scatter/gather 的多次 DRAM 访问频繁触发 row activation，实际 latency 可能远大于报告值。
- 8 个 SDesc 的设计选择依据是什么？对于更复杂的应用（如多个 sparse matrix 的 fusion kernel），SDesc 数量是否会成为瓶颈？
- 论文未讨论 TLB 在 pseudo-virtual address 翻译过程中的 miss 率和代价。对于 scatter/gather 场景，一次 cache line fill 可能需要访问分布在多个 page 上的元素，TLB miss 的概率和影响如何？
