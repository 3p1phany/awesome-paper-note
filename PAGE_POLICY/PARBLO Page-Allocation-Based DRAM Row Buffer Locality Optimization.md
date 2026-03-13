---
title: "PARBLO: Page-Allocation-Based DRAM Row Buffer Locality Optimization"
authors: "Wei Mi, Xiao-Bing Feng, Yao-Cang Jia, Li Chen, Jing-Ling Xue"
venue: "Journal of Computer Science and Technology (JCST) 2009"
year: 2009
---

# PARBLO: Page-Allocation-Based DRAM Row Buffer Locality Optimization

## 基本信息

- **发表**：Journal of Computer Science and Technology, Vol.24, No.6, pp.1086–1097, November 2009
- **单位**：中国科学院计算技术研究所（ICT, CAS）& University of New South Wales

## 一句话总结

> 通过在程序起始处插入初始化循环来控制 OS page allocation 顺序，使数据在 DRAM bank 中有序分布，从而降低 row buffer conflict。

## 问题与动机

**问题**：DRAM row buffer conflict 会显著增加 memory access latency（row buffer hit 比 miss 节省超过一半的延迟），现有的硬件和软件优化手段各有局限，无法充分消除 conflict。

**为什么重要**：对于 memory-intensive 的科学计算程序，row buffer miss rate 可高达 50% 以上（FCFS 下 geomean 为 53%），这直接制约系统性能。

**现有工作的不足**：

1. **FR-FCFS scheduler**：只能看到 memory transaction queue 中少量请求（论文配置为 32 entries），而两次具有 row buffer locality 的 L2 cache miss 之间往往间隔数十到数百条指令，scheduler 的视野太小，无法充分重排。
2. **XOR-based address mapping**：本质上是对物理页面做随机化重映射，打破了 L2 cache 与 memory 之间的地址映射对称性，能减少 cache conflict miss 和 writeback 造成的 row buffer conflict，但无法利用程序的数据访问模式——页面分布依然是随机的。
3. **软件方法**（unroll-and-jam, loop unrolling + scheduling）：对程序 kernel 部分有侵入性修改，可能干扰编译器的其他优化，且面临与 FR-FCFS 类似的"视野"限制。Array fusion 理论上可行，但当多个数组形状和访问模式不同且考虑虚拟内存时，融合极其困难。

## 核心方法

### 关键思路

PARBLO 的核心洞察是：OS 的 page allocation 策略（尤其是 bin-hopping）会将连续的 page fault 请求分配到连续的物理页面；而 address mapping 将连续物理页面映射到不同 bank。因此，如果能**控制程序中 page fault 的触发顺序**，使其与目标循环的访问模式一致，就能让具有 row buffer locality 的数据落在同一 bank 的同一 DRAM page 中，从而大幅减少 conflict。

关键 idea 是：**在程序最开头插入一个轻量级的初始化循环**，按照目标循环的数组访问模式依次触发 page fault，利用 bin-hopping 获得连续物理页面，间接控制数据在 DRAM 中的布局。

### 技术细节

#### 整体框架（三步流程）

**Step 1: Select Target Loops（选择目标循环）**

- 通过 profiling 获取每个循环的 L2 cache miss 数量 α 和平均 reuse distance β（定义为两次访问同一 DRAM page 之间访问了多少个不同 DRAM page）。
- 按 α/β 排序选择目标循环：α 大意味着 miss 多（优化空间大），β 小意味着 locality 好（优化效果好）。

**Step 2: Select Candidate Reference Set（选择候选引用集）**

这一步分为三个子阶段：

1. **分组（Partition into Groups）**：对目标循环中的 affine array reference 做依赖分析。只考虑高维度（去掉 stride < 虚拟页大小的低维度），建立 array dependence graph，将其转为无向图后取最大连通子图作为一个 group。每个 group 访问 page-contiguous 且不重叠的区域，因此 page allocation 互相独立。

2. **选择 leading reference**：在每个 group 内，将引用按 Uniform Group Set (UGS) 分组（UGS 中的引用仅常数项不同），每个 UGS 选一个 leading reference（最先访问新数据、最可能引发 cache miss 的引用）。所有 leading reference 按 stride 分区并排名——所属分区越大，rank 越高。最终选取 rank 最高的 M 个 leading reference（M = bank 数量）进入候选集。

3. **多循环合并**：如果多个目标循环访问重叠区域、具有相同 loop header、且合并后候选引用数 ≤ M，则合并其候选集。按循环权重从高到低处理。

**Step 3: Generate Initialization Loops（生成初始化循环）**

- 初始化循环的循环结构与目标循环相同，但只包含候选集中的引用。
- **正确性保证**：使用 `x = x` 形式的 trivial assignment（而非 `x = 0`），确保不修改程序语义。需要修改编译器使其不将此 trivial assignment 优化掉。必须是 write 操作（而非 read），因为 Copy-on-Write 机制下 read 不会触发真正的 page allocation。
- **开销控制**：初始化循环只需每页 touch 一次，stride 可以设为 32、64 甚至更大，开销可忽略（实测 < 1%）。
- 循环边界值通过 profiling 获取。

#### 辅助优化

**Case 1：UGS 中存在非 leading reference 的 cache miss**

- 这些 miss 对应的物理页面可能不与相邻 miss 的页面连续，导致额外 conflict。
- 解决方案：**loop tiling**，缩短 leading reference 与其他 cache miss reference 的 reuse distance。这与编译器的 cache 优化目标一致，无需额外修改 kernel。

**Case 2：leading reference 数量 > bank 数量**

- 无法避免两个 reference 落在同一 bank。
- 解决方案：**loop splitting**（对最内层循环做分裂），减少每个子循环的 leading reference 数量。可能影响 instruction scheduling 和 register/L1 cache reuse，需配合 strip mining 缓解。

#### PARBLO 与 XOR Address Mapping 的协同

这是论文的一个重要分析。在 write-back L2 cache 系统中：

- PARBLO 可以保证 read miss（R1, R2, R3, R4）访问连续物理页面、分布在不同 bank，互不 conflict。
- 但 dirty cache line 的 writeback（WB1, WB2）时机不可控，且由于 cache 与 memory 的地址映射对称性，writeback **必然**与触发它的 read 落在同一 bank，产生 conflict。
- XOR address mapping 打破了这种对称性：经过 XOR 重映射后，read miss 之间仍保持无 conflict（论文给出了形式化证明——因为连续物理页面的高位 mask bits 相同，XOR 不改变它们之间的 bank 差异），而 writeback 的 conflict 从"必然发生"变为"随机概率"，且概率随 bank 数量增加而降低。

### 与现有工作的区别

| 方法 | 作用层面 | 核心机制 | 局限 |
|------|---------|---------|------|
| FR-FCFS [Rixner, ISCA'00] | MC scheduler | 优先调度 row buffer hit 的请求 | 受限于 transaction queue 大小，视野小 |
| XOR Address Mapping [Zhang et al., MICRO'00] | MC address mapping | XOR bank index 与高位 bits 打破对称性 | 本质是随机化，无法利用程序访问模式 |
| **PARBLO** | **Software（程序初始化）** | **控制 page allocation 顺序使数据有序分布** | **仅适用于 affine loop，依赖 bin-hopping 策略** |

关键差异在于 PARBLO 从软件层面（程序初始化）出发，具有全局视野，能根据程序的数据访问模式主动安排数据在 DRAM 中的布局，而非像 scheduler 一样被动重排或像 XOR 一样盲目随机化。三者是互补而非替代关系，PARBLO-1（FR-FCFS + XOR + PARBLO）效果最优。

## 实验评估

### 实验设置

- **仿真平台**：Simics-based FeS2 full-system simulator + DRAMsim v1.2（嵌入 FeS2 中），运行未修改的 Linux 2.6.26
- **处理器配置**：3 GHz, 4-wide out-of-order; L1 cache 16 KB 8-way write-through (1 cycle); L2 cache 2 MB 8-way write-back (4 cycle); ROB 80 entries; 物理页 4 KB, 内存 2 GB
- **DRAM 配置**：667 MHz DDR2; 1 channel, 2 ranks, 8 banks/rank (共 16 banks); row buffer 4 KB; memory transaction queue 32 entries; 详细 timing 参数完整给出（tRAS=30, tRCD=8, tCAS=8, tRP=8 等 DRAM cycles）
- **Workload**：7 个 benchmark——SPEC2000 (swim), SPEC2006 (leslie3d, GemsFDTD, lbm, bwaves), NPB2.3 serial-C (sp, bt)。均为科学计算程序。编译器为 open64-4.1, -O3, 开启 software prefetching。仿真 100M 指令（无 loop splitting 时）或完整目标片段。
- **对比 baseline**：FC (FCFS), FR (FR-FCFS), PARBLO-0 (FR-FCFS + PARBLO), FR-XOR (FR-FCFS + XOR), PARBLO-1 (FR-FCFS + XOR + PARBLO)

### 关键结果

1. **Row buffer miss rate 大幅降低**：PARBLO-1 在 FR-XOR 基础上进一步降低 row buffer miss rate，geomean 降幅 37.4%，最高 76%（swim）。FCFS 的 geomean miss rate 为 53%，FR-FCFS 降至 ~35.7%（降 32.6%），FR-XOR 降至 ~25.9%（降 51.1%），PARBLO-1 降至 ~16.5%。

2. **性能提升**：PARBLO-1 相比 FR-XOR 实现最高 15% 的 speedup（swim），geomean 为 5%。speedup 与 miss rate 降幅大致成正比。

3. **PARBLO 与 XOR 互补性**：无 XOR 时 PARBLO-0 的 miss rate 仍高达 32.8%，加上 XOR 后（PARBLO-1）降至 16.5%，验证了 writeback conflict 需要 XOR 配合解决。

4. **对 cache miss rate 影响极小**：PARBLO-1 vs FR-XOR 的 L1/L2 miss ratio 差异在 bt/sp 上 < 3%，其他 benchmark < 1%。

### 结果分析

**Sensitivity Analysis 覆盖全面**：

- **Bank 数量（8~128）**：PARBLO 在中等 bank 数（16, 32, 64）效果最显著。Bank 太少（8）时无法为所有 leading reference 分配独立 bank，效果受限；bank 很多（128）时 XOR 本身就能将 conflict 稀释到很低，PARBLO 的边际收益减小。
- **Row buffer 大小（2 KB~8 KB）**：PARBLO 相对于 XOR 的 miss rate 降幅在不同 row buffer size 下变化不大，说明 PARBLO 不要求 row buffer size = physical page size。
- **Instruction window size（80~1280）**：扩大 instruction window 对 FR-XOR 和 PARBLO-1 都有轻微改善（因为开了 software prefetching，instruction window blocking 已大幅减少），但 PARBLO-1 始终优于 FR-XOR。
- **Memory transaction queue size（变化范围未明确给出具体数值，但图中可见）**：32 entries 已足够 FR-FCFS 发挥效果，继续增大无显著变化。PARBLO-1 在各 queue size 下均优于 FR-XOR。

**特殊 benchmark 分析**：

- **bwaves**：FR-XOR 下 miss rate 仅 14.9%，PARBLO-1 无进一步改善——因为 bwaves 的原始初始化模式恰好与 PARBLO 所需一致。反向验证：人为打乱 bwaves 初始化后 miss rate 升至 44.6%。
- **GemsFDTD**：需要 loop tiling 配合（open64 已自动完成），否则 PARBLO 仅改善 5%。
- **sp**：需要 loop splitting + strip mining，副作用极小（instruction count +0.14%, L1 miss +0.2%, L2 miss +1.02%）。

## 审稿人视角

### 优点

1. **思路新颖且实用**：从 page allocation 角度切入 row buffer locality 优化是一个非常巧妙的视角。利用 OS 的 bin-hopping 行为而不修改 OS 本身，实现简单优雅。这个 insight——"控制 page fault 顺序就能控制数据在 DRAM 中的布局"——具有启发性。

2. **与现有优化正交互补**：论文清晰地分析了 PARBLO 与 FR-FCFS、XOR address mapping 的关系，尤其是 Section 5 对 PARBLO + XOR 在 write-back cache 场景下的协同分析，包含形式化证明（Proof 1），技术上严谨。

3. **对程序主体无侵入性**：只修改程序初始化部分（插入初始化循环），不干扰编译器对 kernel 的优化，副作用极小（cache miss ratio 变化 < 3%）。这是相比传统软件方法（unroll-and-jam 等）的重要优势。

4. **Sensitivity analysis 较为全面**：覆盖了 bank 数量、row buffer 大小、instruction window size、memory transaction queue size 四个维度，结论合理。

### 不足

1. **适用性窄**：方法仅适用于 affine loop + affine array reference 的科学计算程序。7 个 benchmark 全部是规则的科学计算 workload（SPEC FP + NPB），对 irregular access pattern（如 graph processing、sparse matrix、pointer-chasing）完全无能为力。论文声称"可扩展到更广泛的程序"但未给出任何具体方案。

2. **依赖 bin-hopping page allocation 策略**：论文假设 OS 采用 bin-hopping 或类似策略（如 Linux buddy system 在内存压力不大时的行为）。但在内存碎片化严重、多进程竞争、或 OS 采用不同策略时，连续 page fault 不一定得到连续物理页面，PARBLO 的前提可能不成立。论文对这一关键假设的鲁棒性缺乏讨论。

3. **手工应用，缺乏自动化工具**：论文明确说 transformation 是 "applied by hand"，虽然描述了算法框架（Fig.3），但没有实现自动化的编译器 pass。profiling 获取 loop bound、α、β 等参数也需要人工介入。这限制了方法的实际可部署性。

4. **单核单线程场景**：所有实验均在单核单线程下进行。论文在 future work 中提到扩展到多核，但多核场景下多个程序竞争物理页面，bin-hopping 行为将更加不可控，且不同程序的初始化循环可能互相干扰。这是一个根本性的可扩展性问题。

5. **初始化循环的 Copy-on-Write 假设**：方法要求初始化循环使用 write 操作（`x = x`）来触发真正的 page allocation，并需修改编译器防止 trivial assignment 被优化掉。这虽然简单，但引入了对编译器行为的额外依赖，且在不同编译器/优化级别下的可移植性存疑。

6. **仿真规模偏小**：仅仿真 100M 指令，对于 SPEC 和 NPB 这种大规模 workload 可能无法捕获完整的程序行为。且未讨论 warm-up 策略。

### 疑问或值得追问的点

- 在真实 Linux 系统上（而非 full-system simulation），内存碎片化时 buddy system 的 bin-hopping-like 行为是否可靠？是否有实际硬件验证数据？
- 初始化循环本身会占用物理内存（因为触发了 page fault 和实际的 page allocation），对于内存受限的场景，这额外的内存开销是否被考虑？
- 方法对 NUMA 系统的适用性如何？不同 NUMA node 的 page allocation 策略可能完全不同。
- 如果目标循环的 loop bound 在运行时才确定且变化较大（如输入依赖），profiling 获得的固定 loop bound 是否会导致初始化循环不匹配？
- 2009 年的 DDR2 时代 bank 数为 8~16；现代 DDR5 系统有 32+ bank group，且引入了 bank group 概念，PARBLO 的 bank-level 分析是否仍然适用？
