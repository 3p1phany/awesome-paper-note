---
title: "Active Pages: A Computation Model for Intelligent Memory"
authors: "Mark Oskin, Frederic T. Chong, Timothy Sherwood"
venue: "ISCA 1998"
year: 1998
---

# Active Pages: A Computation Model for Intelligent Memory

## 基本信息
- **作者**：Mark Oskin, Frederic T. Chong, Timothy Sherwood (UC Davis)
- **发表**：ISCA 1998（根据内容和引用风格判断为 ISCA）

## 一句话总结
> 提出 Active Pages 计算模型，将 reconfigurable logic 集成到 DRAM subarray 旁，实现 data-intensive 计算向 memory 侧的 offloading，获得最高约 1000× 加速。

## 问题与动机

论文瞄准的是经典的 **processor-memory performance gap** 问题。1990 年代末，processor 性能按照摩尔定律持续增长，但 DRAM 访问延迟和带宽的改善远远跟不上。传统的 processor-centric 优化手段——prefetching、speculation、out-of-order execution、multithreading——虽然能缓解问题，但本质上仍然受限于 memory bus bandwidth。论文引用了 Wulf & McKee 的 "memory wall" 论文 [WM95] 和 Burger et al. 关于 memory bandwidth limitation [BGK96] 的工作作为动机支撑。

现有工作的不足主要体现在两方面：

1. **IRAM（Processor-in-Memory）方案**（Patterson, Berkeley）：将完整 processor core 集成到 DRAM die 上。问题在于：(a) yield 低，制造成本接近 processor 而非 DRAM；(b) 单芯片内只有一个 processor，parallelism 受限；(c) 与 commodity processor 生态不兼容。
2. **Fixed-function logic-in-DRAM**（如 Read-Modify-Write 方案 [B+97b]）：只能支持预定义的少数操作，无法适配多样化的应用需求。

论文的核心动机是：能否设计一种既能利用 DRAM 内部高带宽和并行性，又足够灵活以适配多种应用，同时保持低制造成本和与 commodity processor 兼容的 intelligent memory 方案？

## 核心方法

### 关键思路

Active Pages 的核心 insight 是将**数据页面（page of data）与一组可在该数据上执行的函数（AP functions）绑定**，类似于面向对象编程中数据与方法的封装，但映射到 hardware 层面。通过在每个 DRAM subarray 旁集成 reconfigurable logic（FPGA），实现大规模数据并行处理——问题规模越大，可用的 Active Pages 越多，计算能力随数据量线性增长。

关键洞察在于：很多 data-intensive workload 中，processor 花费大量时间做简单的数据操纵（比较、搜索、移位、gather/scatter），这些操作虽然计算量小但 memory access 密集。将这些操作放到 DRAM 侧执行，可以利用 subarray 内部远高于外部 bus 的带宽，同时释放 processor 做更复杂的运算（如浮点乘法）。

### 技术细节

#### 1. Active Pages 编程模型

Active Pages 模型定义了一套类似 virtual memory 的接口：

- **`AP_alloc(group_id, vaddr)`**：在虚拟地址 `vaddr` 处分配一个属于 group `group_id` 的 Active Page。
- **`AP_bind(group_id, AP_functions)`**：将一组函数绑定到一个 page group。可通过重复调用 `AP_bind` 来替换函数（因为 FPGA 资源有限，可能需要 rebinding）。
- **标准 `read(vaddr)`/`write(vaddr, data)`**：用于数据读写和同步。
- **Synchronization variables**：通过普通 memory-mapped read/write 实现 processor 与 AP functions 之间的协调，类似 NIC 的 memory-mapped registers。

AP functions 使用 **virtual address**，可以引用进程地址空间内的任何虚拟地址（包括其他 page），但跨页引用应尽量少用。

#### 2. Partitioning 策略

论文提出了两种 partitioning 模式：

- **Processor-centric partitioning**：适用于包含复杂计算（如浮点运算）的应用。Active Pages 负责 data gathering/indexing，processor 负责核心计算。典型例子是 sparse matrix multiply：AP 做 index comparison 和 gather/scatter，processor 做浮点乘法。
- **Memory-centric partitioning**：适用于以数据操纵和整数运算为主的应用。大部分计算在 Active Pages 内完成，processor 主要做 activation 和结果收集。典型例子是 image median filtering。

#### 3. RADram 架构实现

RADram（Reconfigurable Architecture DRAM）的物理设计基于 gigabit DRAM 的 subarray 结构：

- 每个 **512 KB subarray** 旁配置 **256 个 Logic Elements (LEs)**，每个 LE 基于 4-LUT（与 FPGA 中的标准逻辑单元一致）。
- 面积预算：根据 SIA roadmap，1-gigabit DRAM chip 如果将一半面积用于 logic，可支持约 32M 晶体管，足够为每个 512K subarray 配 256 LEs（每个 LE 约 1K 晶体管）。
- Reconfigurable logic 时钟频率假设为 **100 MHz**（processor 为 1 GHz，即 10:1 的 logic divisor）。
- Memory bus：32-bit data transfer，每 10 ns 一次传输。
- DRAM 与 logic 之间的带宽保守设定为 **32 bits**（论文指出可以增加到 256-512 bits，但受功耗和面积约束限制）。

#### 4. Inter-page Communication

采用 **processor-mediated** 方式：当 AP function 需要访问非本地 page 的数据时，它 block 并触发 processor interrupt。Processor 通过 read/write 操作代为完成跨页数据传输。这种方式简化了 virtual memory 管理，但假设跨页通信不频繁。

#### 5. Computation Scaling 模型

论文提出了一个重要的分析框架，将 Active Pages 性能随 problem size 的增长分为三个区域：

- **Sub-page region**：问题规模小，Active Pages 利用率低，activation time 主导，加速比不显著。
- **Scalable region**：Active Pages 数量随数据量线性增长，计算能力也线性增长，speedup 线性提升。
- **Saturated region**：processor 成为瓶颈（无论是计算能力还是 memory bus bandwidth），speedup 趋于饱和甚至可能因 synchronization overhead 增大而略有下降。

论文进一步建立了解析模型（Figure 7），将总执行时间分解为三部分：activation time `TA(i)`、post-processing time `TP(i)` 和 non-overlap time `NO(i)`（processor 等待 AP 完成的 stall 时间）。

#### 6. 设计权衡（Trade-offs）

- **Reconfigurable vs. Fixed logic**：选择 reconfigurable logic 牺牲了速度（FPGA 比 ASIC 慢很多），但获得了灵活性。不同应用需要截然不同的 AP functions（如 median sort vs. index comparison vs. database search），fixed logic 无法通用化。
- **DRAM process vs. Logic process**：在 DRAM process 上做 logic 意味着逻辑速度和密度较差，但保持了 DRAM 级别的制造成本和 yield。论文明确选择保守假设（DRAM process），而非 merged DRAM-logic process。
- **Processor-mediated vs. Hardware inter-page communication**：选择前者牺牲了跨页通信性能，但大幅简化了 virtual memory 和 paging 的设计复杂度。
- **32-bit internal bandwidth**：保守选择以限制功耗，牺牲了潜在的更高并行度。

### 与现有工作的区别

| 维度 | Active Pages / RADram | IRAM [Pat95] | Fixed-logic PIM [B+97b] |
|------|----------------------|--------------|--------------------------|
| 灵活性 | 高（reconfigurable） | 高（通用 processor）| 低（固定操作集）|
| 并行度 | 极高（128+ 并行 AP） | 低（单 processor core）| 中等 |
| 制造成本 | 接近 DRAM | 接近 processor（yield 低）| 接近 DRAM |
| 与 commodity processor 兼容 | 兼容（标准 memory bus）| 不兼容（替代 processor）| 兼容 |
| 浮点能力 | 无（依赖 host processor）| 有 | 无 |

## 实验评估

### 实验设置

- **仿真平台**：修改版 SimpleScalar v2.0，将 conventional memory hierarchy 替换为 Active-Page memory system 模型。扩展了 Intel MMX 指令集支持。编译器为 GCC v2.7.2.1，`-O3` 优化。
- **硬件配置（参考配置）**：
  - CPU: 1 GHz
  - L1 I-Cache: 64 KB, L1 D-Cache: 64 KB (2-way associative)
  - L2 Cache: 1 MB (4-way associative)
  - Reconfigurable logic: 100 MHz
  - Cache miss penalty: 50 ns
  - Memory bus: 32-bit, 10 ns per transfer
- **参数扫描范围**：L1 D-Cache 32K-256K，L2 Cache 256K-4M，Logic 频率 10-500 MHz，Cache miss penalty 0-600 cycles
- **Workload（6 个应用，覆盖 memory-centric 和 processor-centric）**：
  - Memory-centric: STL Array (insert/delete/find), Database query (unindexed address book search), Median filter (image processing), LCS dynamic programming (DNA sequence matching)
  - Processor-centric: Sparse Matrix Multiply (Harwell-Boeing finite element + Simplex register allocation), MPEG-MMX (P/B frame correction)
- **对比 baseline**：相同应用在 conventional memory system（相同 SimpleScalar 配置）上的优化实现

### FPGA 综合结果

所有 AP functions 均用 VHDL 手写并综合到 Altera FLEX-10K10-3 FPGA：

| Application | LEs | Speed | Code Size |
|-------------|-----|-------|-----------|
| Array-delete | 109 | 29.0 ns | 2.7 KB |
| Array-insert | 115 | 26.2 ns | 2.9 KB |
| Array-find | 141 | 32.1 ns | 3.5 KB |
| Database | 142 | 35.4 ns | 3.5 KB |
| Dynamic Prog | 179 | 39.2 ns | 4.5 KB |
| Matrix | 205 | 45.3 ns | 5.6 KB |
| MPEG-MMX | 131 | 34.6 ns | 3.3 KB |

所有设计均在 256 LEs 的面积约束内。

### 关键结果

1. **显著加速**：在 scalable region 内，所有应用均获得显著 speedup。Memory-centric 应用（如 median filtering kernel）在大问题规模下达到约 **1000× speedup**；database 在 100+ pages 时达到约 **100× speedup**；matrix-simplex 和 matrix-boeing 达到约 **10-30× speedup**。

2. **Scaling 行为符合预期**：实验结果定性地验证了 Figure 1 的三区域 scaling 模型——sub-page region 加速比增长缓慢，scalable region 线性增长，saturated region 趋于平稳。Database、MMX、matrix 和 median filtering 均进入了 saturated region。

3. **Cache 影响有限**：L1 D-Cache 从 32K 增大到 256K，对 RADram 应用性能几乎无影响（除 median-total 有 stride effects）。L2 cache 大小变化也无显著影响。说明 Active Pages 应用的性能瓶颈不在 cache，而在 DRAM 内部计算能力和 memory bus bandwidth。

4. **技术敏感性**：
   - **Memory latency**（Figure 8）：RADram speedup 对 cache-to-DRAM latency 变化相对不敏感，因为核心计算发生在 DRAM 内部，不经过 cache。Speedup 随 latency 的变化方向取决于 conventional vs. partitioned 应用各自的 instruction cycles 与 memory stall cycles 比例。
   - **Logic speed**（Figure 9）：处于 scalable region 的应用对 logic 速度敏感（logic 变慢则 speedup 下降）；处于 saturated region 的应用对 logic 速度不敏感（瓶颈在 processor 侧）。

### 解析模型验证

论文用 `TA`、`TP`、`TC` 三个参数的简化模型预测 speedup，与仿真结果的 correlation 达到 0.830-0.999（Table 4）。Matrix-boeing 的 correlation 最低（0.830），因为其 per-page activation 和 computation time 随数据内容变化较大，违反了常数假设。

## 审稿人视角

### 优点

1. **抽象层次设计优雅**：Active Pages 作为计算模型，将 data 与 function 绑定的思想既简洁又通用，interface 设计（`AP_alloc`/`AP_bind`/标准 read/write）巧妙地复用了 virtual memory 语义，对 OS 和 commodity processor 的侵入性极小。这一点在 1998 年的 PIM 研究中是难得的工程导向思维。

2. **全面的实验方法论**：论文不仅给出了多种应用的 end-to-end speedup，还提供了 FPGA 综合验证（面积、时序）、解析模型及其 correlation 验证、cache sensitivity analysis、technology sensitivity analysis（logic speed 和 memory latency 的参数扫描）。对于一篇提出新计算模型的论文来说，evaluation 的完整性值得肯定。

3. **Scaling 分析框架有价值**：Sub-page / Scalable / Saturated 三区域模型是一个很好的直觉框架，对后续 PIM 研究理解 problem size 与加速比关系有启发意义。解析模型虽然简化，但捕捉了核心 trade-off（activation overhead vs. computation overlap vs. processor saturation）。

4. **与 IRAM 的对比论证有说服力**：从 yield/cost、parallelism、commodity compatibility 三个角度系统论证了 reconfigurable logic in DRAM 相对于 processor-in-DRAM 的优势，在当时的技术背景下是合理且有前瞻性的。

### 不足

1. **应用评估缺乏代表性和真实性**：所有 workload 都是 micro-kernel 级别的，缺乏完整应用的 end-to-end 评估。论文声称 "up to 1000× speedup"，但这是在 median filter kernel 这种极端 memory-centric、几乎完全可并行化的小 kernel 上取得的。对于真实应用（其中 Active Pages 计算只是整体执行的一部分），实际加速比会被 Amdahl's Law 大幅稀释。论文虽然提到了 Amdahl's Law（Figure 7 的 `Fraction_partitioned`），但并未在结果中给出真实应用场景下 partitioned fraction 的具体数值。

2. **Hand partitioning 是重大局限**：所有应用的 processor-memory partitioning 都是手工完成的，AP functions 是手写 VHDL。论文在 Future Work 中承认需要 compiler support for automatic partitioning，但这恰恰是最大的实用性障碍。没有自动化工具链，Active Pages 的 adoption 前景很不明朗。

3. **Inter-page communication 模型过于简化**：Processor-mediated 方式（interrupt + processor 代为读写）对于需要频繁跨页通信的应用（如 stencil computation 的 halo exchange、graph algorithm 的随机访问）会成为严重瓶颈。论文回避了这个问题，只选择了跨页通信需求较少的应用。对于 LCS dynamic programming 这类 wavefront 计算，跨页通信实际上是关键路径的一部分，但论文对此的讨论不够深入。

4. **功耗分析缺失**：论文承认功耗是 DRAM 芯片的重大关切（影响 refresh rate），但仅用一段定性讨论带过，没有任何量化的 power estimation。在 DRAM subarray 旁持续运行 reconfigurable logic 的功耗开销（尤其是 FPGA 的 static power）可能非常显著，这是一个不应被忽略的设计维度。

5. **仿真精度存疑**：SimpleScalar v2.0 本身是一个 processor 模拟器，论文将其扩展为 Active-Page memory system simulator，但对这个扩展的仿真精度缺乏充分描述。特别是 DRAM 时序模型的精度（是否建模了 row buffer、bank conflict、refresh 等）不明确。对于一篇核心贡献在 memory system 的论文，memory model 的精度描述不足是一个明显短板。

6. **Reconfiguration overhead 被低估**：论文提到当时 FPGA reconfiguration 需要 100s of milliseconds，并期望未来技术改善。但 AP function 的 rebinding（`AP_bind`）涉及 FPGA reconfiguration，这个开销在 time-sharing 或多应用场景下会非常昂贵。论文的 benchmark 评估中似乎没有包含 reconfiguration 时间。

### 疑问或值得追问的点

- 论文假设每个 subarray 256 LEs，但不同应用的 AP function 复杂度差异很大（109-205 LEs）。如果一个应用需要同时使用 insert、delete、find 三个函数，总 LEs 需求为 365，超过了 256 的限制。论文提到可以通过 `AP_bind` 重新绑定来切换函数，但这引入的 reconfiguration 开销是多少？是否在仿真中被考虑？
- 512 KB 的 page size 在当时属于 superpage，这意味着 TLB miss 和 page fault 的行为与传统 4 KB page 完全不同。OS 需要怎样的修改来支持 Active Pages 的 allocation 和 paging？如果 Active Page 被 swap 到 disk，需要同时保存 FPGA configuration，这个成本如何？
- 论文声称 RADram 可以作为 conventional memory system 使用且 "negligible performance degradation"，但未给出任何实验数据支撑这一说法。
- 对于 sparse matrix multiply 这类 processor-centric 应用，Active Pages 主要做 gather/scatter。现代 processor 的 gather/scatter 指令（如 AVX-512）是否已经在很大程度上解决了这个问题？（这是一个 retrospective 视角的问题。）
