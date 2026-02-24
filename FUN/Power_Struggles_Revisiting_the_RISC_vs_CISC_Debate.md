---
title: "Power Struggles: Revisiting the RISC vs. CISC Debate on Contemporary ARM and x86 Architectures"
authors: "Emily Blem, Jaikrishnan Menon, Karthikeyan Sankaralingam"
venue: "HPCA 2013"
year: 2013
---

# Power Struggles: Revisiting the RISC vs. CISC Debate on Contemporary ARM and x86 Architectures

## 基本信息

- **发表**：IEEE International Symposium on High Performance Computer Architecture (HPCA), 2013
- **机构**：University of Wisconsin - Madison

## 一句话总结

> 通过在真实硬件上的系统性测量，证明 ARM 与 x86 的性能和能效差异源于 microarchitecture 设计选择而非 RISC/CISC ISA 本身。

## 问题与动机

**核心问题**：在当代计算环境下，ISA 是 RISC 还是 CISC 是否对处理器的 performance、power 和 energy efficiency 有本质影响？

**问题重要性**：2013 年前后的计算格局发生了重大变化——ARM（RISC）开始进军高性能 server 市场，x86（CISC）开始进入 mobile 低功耗市场。此前关于 RISC vs. CISC 的研究主要在 1980-1990 年代完成，且几乎只关注 performance，未涉及 power 和 energy。当时的设计约束是 chip area 和设计复杂度，而今天的核心约束已转向 energy 和 power。

**现有工作的不足**：

- Bhandarkar & Clark（1991）比较 MIPS 与 VAX，结论是 RISC 有显著性能优势，但只看 performance。
- Bhandarkar（1995）比较 Pentium Pro 与 Alpha 21164，结论是 aggressive out-of-order 设计可以部分弥补 CISC 的 ISA 级限制，仍然只关注 performance。
- Isen et al. 比较 Power5+ 与 Intel Woodcrest，结论是通过 aggressive ILP 技术，CISC 和 RISC 可以达到相似 performance。
- 近年的非正式研究（如 AnandTech、Phoronix）声称 x86 的 "crufty" CISC ISA 带来了不可克服的 power overhead，将 ARM 的功耗优势归因于 ISA——但缺乏严格的方法学支撑。

总之，没有一项前期工作在 **相同方法学框架** 下同时考量 performance、power 和 energy，并在 **真实硬件 + 真实应用** 上进行系统测量。

## 核心方法

### 关键思路

论文的核心 insight 是：ARM 和 x86 处理器之间观察到的巨大 power 差异（1-2W vs. 5-36W）并非 ISA 本质导致，而是 **engineering design point** 的选择结果——即为不同 performance level 优化的 microarchitecture 和 technology node 差异。论文通过构建一个系统化的分析框架（从 format、operations、operands 三个维度），结合在 4 个真实硬件平台上的 26 个 workload 的超过 20,000 个数据点，逐层剥离 frequency、technology、microarchitecture 的影响，最终定位 ISA 的真实贡献。

### 技术细节

#### 1. 分析框架（Section 2）

论文从 ISA 教科书的三个关键特征出发构建分析框架：

| 维度             | RISC/ARM 特点       | CISC/x86 特点                                  | 历史对比                              | 现代趋同                                                               |
| -------------- | ----------------- | -------------------------------------------- | --------------------------------- | ------------------------------------------------------------------ |
| **Format**     | 固定长度指令（ARM: 4B）   | 变长指令（x86: 1B-16B）                            | CISC decode latency 阻碍 pipelining | µ-op cache 最小化 decode overhead；I-cache 隐藏 code density 差异          |
| **Operations** | 简单、单周期操作          | 复杂多周期指令（transcendentals, encryption, string） | 即使有 µcode，pipelining 仍然困难         | CISC 指令被 split 为 RISC-like micro-ops；现代 compiler 主要选择 RISC-like 指令 |
| **Operands**   | 操作数：寄存器、立即数；寻址模式少 | 操作数：内存、寄存器、立即数；寻址模式多                         | CISC decoder 复杂度更高                | x86 decode 对常见指令优化；CISC 指令 split 后 µ-op latency 与 ARM 类似           |

基于此框架，论文提出了一系列可验证的 empirical questions，如：x86 指令长度方差有多大？macro-op 数量是否相似？data access 数量是否相似？

#### 2. 实验平台选择（Section 3）

4 个平台的选择遵循严格的可比性原则：

| 特性           | Cortex-A8 | Atom N450 | Cortex-A9         | i7-2700    |
| ------------ | --------- | --------- | ----------------- | ---------- |
| ISA          | ARMv7     | x86       | ARMv7             | x86        |
| 频率           | 0.6 GHz   | 1.66 GHz  | 1 GHz             | 3.4 GHz    |
| Issue Width  | 2-way     | 2-way     | 2-way（实际可能 4-way） | 4-way      |
| Issue 类型     | In-Order  | In-Order  | OoO               | OoO        |
| L1 D/I Cache | 16KB/16KB | 24KB/32KB | 32KB/32KB         | 32KB/32KB  |
| L2 Cache     | 256KB     | 512KB     | 1MB/chip          | 256KB/core |
| Technology   | 65nm      | 45nm      | 45nm              | 32nm       |

**关键设计选择**：

- 配对比较策略：A8 vs. Atom（均为 dual-issue in-order），A9 vs. i7（均为 OoO），消除 microarchitecture 类型的混淆变量。
- 统一 OS：全部运行 Linux 2.6 LTS kernel。
- 统一 compiler：使用 gcc 4.4 cross-compiler，O3 优化但禁用 machine-specific tuning，以消除 compiler 影响。
- 禁用 THUMB 指令以保持更纯粹的 RISC 特性。

#### 3. Performance 分析流程（6 步法）

**Step 1: Execution Time** → 观察到巨大的跨平台性能差异（A9 vs. i7: 5×-102×）。

**Step 2: Cycle Count**（消除频率影响）→ 差距大幅缩小：in-order 核之间 ≤1.5×，OoO 核之间 ≤2.5×。

**Step 3: Instruction Count** → macro-op 数量跨 ISA 基本相同（说明 gcc 主要选择 RISC-like x86 指令）；x86 micro-op/macro-op ratio 通常 <1.3×。

**Step 4: Instruction Format & Mix** → 二进制大小在 SPEC INT、SPEC FP、Mobile 上 ARM 和 x86 相似；x86 动态指令平均比 ARM 短 25%（说明常用的 x86 指令编码很短）；load/store 比例跨 ISA 一致。

**Step 5: Microarchitecture 分析** → 性能差异的根因是 microarchitecture 资源的巨大差异（如 A9 BTB 512 entries vs. i7 BTB 8K-16K entries；A9 ROB ~56 entries vs. i7 ROB 160 entries）。

**Step 6: ISA 对 microarchitecture 的影响** → x86 ISA 仅强制要求 micro-op translation 和 µ-op cache，对 ROB/rename file/scheduler size 的影响很小（因为 x86 micro-op 数量与 ARM instruction 数量接近）。

#### 4. Power 和 Energy 分析流程（4 步法）

**Step 1: Raw Power** → x86 实现确实消耗显著更多功率（Atom/A8: 3×，i7/A9: 17-21×）。

**Step 2: Technology-Independent Power**（标准化到 45nm、1GHz）→ A8、A9、Atom 功率相差在 0.6× 以内，**Atom 甚至比 A8 和 A9 更低**。i7 功率高 6-7.6× 是因为它是 performance-optimized 设计。

**Step 3: Energy**（power × time）→ Atom 能耗低于 A8（因为执行更快），i7 能耗仅略高于 A9。

**Step 4: ISA 对能耗的影响** → x86 ISA 引入的额外能耗仅限于 micro-op translation 和 µ-op cache，影响很小。大功耗结构（大 L2、高关联 TLB、aggressive prefetcher、大 branch predictor）由 performance target 决定，非 ISA 所需。

#### 5. Trade-off 分析

- 使用 Power-Performance Pareto frontier 分析，所有 4 个处理器（无论 ISA）都落在同一条 cubic curve 上。
- Energy-Performance 分析中，A9 和 Atom 的能耗差距在 24% 以内，i7 通过 DVFS 降到 2GHz 可以达到与 A9 相同的 energy level 但提供 6× performance。
- 使用 Energy-Delay product（ED）和 ED^1.4 指标，性能与功耗平衡的 core（A15）ED 最优，而重视性能时 i7 在 ED^1.4 和 ED^2 上最优。

### 与现有工作的区别

| 对比工作                                               | 本文的关键差异                                                             |
| -------------------------------------------------- | ------------------------------------------------------------------- |
| **Bhandarkar & Clark (1991)**: MIPS vs. VAX        | 本文同时考量 performance、power、energy 三个维度，使用现代处理器和 workload              |
| **Bhandarkar (1995)**: Pentium Pro vs. Alpha 21164 | 本文使用真实硬件测量而非仅模拟，且覆盖 mobile/desktop/server 全场景                       |
| **Isen et al. (2009)**: Power5+ vs. Woodcrest      | 本文首次系统性地分离 ISA、microarchitecture、technology node 对 power/energy 的影响 |
| **非正式研究**（AnandTech 等）                             | 本文提供严格的方法学，而非基于少数 benchmark 的推测                                     |

## 实验评估

### 实验设置

- **硬件平台**：ARM Cortex-A8 (Beagleboard)、ARM Cortex-A9 (Pandaboard)、Intel Atom N450 (Dev Board)、Intel i7-2700 (Sandybridge Desktop)
- **测量工具**：
  - Performance: Linux `perf` 工具访问 hardware performance counters
  - Power: WattsUp 功率计，board power 通过 halt 状态测量后减去
  - Instruction Mix: DynamoRIO (x86)、gem5 functional emulator (ARM)
  - ILP 分析: Pin-based MICA tool
- **Technology Scaling**: 使用 2007 ITRS 表将所有核标准化到 45nm/1GHz
- **Workload**（共 26 个）：
  - Mobile: CoreMark, 2× WebKit regression tests
  - Desktop: SPEC CPU2006 (10 INT + 10 FP, test inputs)
  - Server: lighttpd (web-serving), CLucene (web-indexing), database kernels (data-streaming/analytics)
- **对比方式**：配对比较——A8 vs. Atom (in-order), A9 vs. i7 (OoO)

### 关键结果

1. **Performance 差异主要来自 microarchitecture 而非 ISA**：消除频率后，in-order 核之间 cycle count 差距 ≤1.5×，OoO 核之间 ≤2.5×；instruction count 和 mix 跨 ISA 基本一致。

2. **ISA 对 power 的影响微乎其微**：标准化到 45nm/1GHz 后，A8、A9、Atom 功率在 0.6× 以内，Atom（CISC）甚至低于 A8 和 A9（RISC）。

3. **Energy 由设计选择主导**：Atom 与 A8 的 raw energy ratio 为 0.8×（CISC 更优），i7 与 A9 的 energy ratio 为 1.7×（因 i7 是 performance-optimized 设计，对 "hard" benchmark 有 2-3× energy overhead）。

4. **所有处理器落在同一条 Power-Performance trade-off 曲线上**：遵循 cubic power/performance 关系，与 ISA 无关。i7 通过 DVFS 降到 2GHz 可以在与 A9 相同能耗下提供 6× 性能。

### 结果分析

**性能差异的根因剖析**（Table 8）：对 A9 vs. i7 差距 >3× 的 benchmark 进行逐一分析——omnetpp 的差距来自 branch MPKI（A9: 59 vs. i7: 2.0）和 I-Cache MPKI（A9: 33 vs. i7: 2.2）；bwaves 差距达 30×，根因是 A9 branch MPKI 比 i7 高 1000×。这些差异全部可归因于 microarchitecture 资源差异（如 BTB 512 vs. 16K entries），而非 ISA。

**Sensitivity 分析**：

- Compiler 影响：gcc vs. icc 对 SPEC INT 的 static code size 差异 <8%，平均指令长度差异 <4%。
- Machine-specific optimization 影响：超过半数 SPEC benchmark <5%，其余在 ±20% 范围内波动。
- Power 测量精度：WattsUp 方法与 i7 内建 energy counter 的差异为 4%-17%（平均 12%），倾向于低估 core power。

**效果较差的场景**：对于 cache miss rate 很高、core utilization 低的 "hard" benchmark（如许多 SPEC FP），i7 的 fixed energy cost（来自为 high-performance 提供的大型结构）导致其 energy 比 A9 差 2-3×。这说明 performance-optimized 设计在低利用率场景下有显著的 energy penalty。

## 审稿人视角

### 优点

1. **方法学严谨，分析层次清晰**：论文构建了系统化的 6-step performance 分析 + 4-step power/energy 分析框架，逐层剥离 frequency、technology node、microarchitecture 的影响，最终定位 ISA 的贡献。这种 "peeling the onion" 的方法论值得推广。将 ISA 特征分解为 format/operations/operands 三个维度并分别提出 empirical questions 的框架也很有启发性。

2. **基于真实硬件的全面测量**：在 4 个平台、26 个覆盖 mobile/desktop/server 的 workload 上收集超过 20,000 个数据点，比单纯依赖模拟器的研究更有说服力。数据公开发布（www.cs.wisc.edu/vertical/isa-power-struggles）也体现了可重复性原则。

3. **结论具有重大实际影响**：论文在 ARM 进军 server、x86 进军 mobile 的关键时间节点（2013 年）发表，其 "ISA is irrelevant" 的结论对工业界的架构决策具有直接指导意义，有效反驳了当时流行的 "ARM 天生更省电" 的迷思。

4. **坦诚的 limitation 讨论**：Table 4 系统地列出了所有 limitation 及其 implications，Table 9 详细描述了实验过程中遇到的 challenges 和 workarounds，这种透明度增强了可信度。

### 不足

1. **每种 ISA 仅有 2 个实现，样本量不足以做强统计推断**：论文自己也承认 "a core's location on the frontier does not imply optimality"。特别是 Pareto frontier 分析中使用 cubic/quadratic curve fitting 只有 4 个数据点，过拟合风险高。缺少来自其他厂商的实现（如 Qualcomm Krait、NVIDIA Tegra 等已调研但未纳入），削弱了结论的普适性。

2. **Power 测量粒度较粗**：使用 WattsUp 测量 board-level power 然后减去 board power 的方法引入了 4%-17% 的误差。更关键的是，无法分离 **decode 单元的 power 贡献**——这恰恰是 RISC vs. CISC 争论中最核心的 power 组件。论文承认 "No direct decoder power measure"，将其影响定性为 "2nd order"，但缺乏量化证据支持这一判断。

3. **Compiler 控制可能引入偏差**：为了消除 compiler 影响而使用 gcc 并禁用 machine-specific tuning，这在公平性上有合理性，但也意味着 x86 未能利用其 ISA 的全部能力（如 SSE/AVX intrinsics、复杂寻址模式优化）。Finding P4 中 "gcc picks RISC-like instructions from x86 ISA" 可能恰恰是因为 gcc 的 x86 后端在禁用 machine-specific tuning 后保守了；这使得 "x86 macro-op count ≈ ARM instruction count" 的结论可能在使用 vendor compiler（如 ICC）时不成立。虽然论文提供了 gcc vs. icc 的 code size 比较（差异 <8%），但未比较 instruction count 和 mix。

4. **Workload 代表性存疑**：
   
   - SPEC CPU2006 使用 **test inputs** 而非 reference inputs（因为 A8 只有 256MB 内存），test inputs 的 working set 远小于 reference inputs，无法体现 memory hierarchy 的压力差异。
   - Mobile workload 仅 3 个（CoreMark + 2 WebKit tests），难以代表真实 mobile 场景中的 GPU offload、中断处理、多任务等特性。
   - Server workload 使用了简化的 kernel 实现而非完整的 CloudSuite 应用。

5. **未考虑 64-bit ISA 的影响**：论文明确指出因为 64-bit ARM 尚未上市而只比较 32-bit，但 64-bit 模式下 x86 的寄存器数量翻倍（8→16 GPR），这可能显著改变 instruction count 和 register spilling 行为，进而影响 power 和 performance 的比较结论。

### 疑问或值得追问的点

- **Decoder power 的量化**：论文最核心的 gap 在于无法直接测量 x86 decode pipeline 的 power overhead。如果有 RTL-level 的 power breakdown（如后来的一些学术处理器所提供的），是否会改变 "ISA is irrelevant" 的结论？后续工作（如 Boehm et al. 的 RISC-V 研究）是否补充了这一数据？

- **Technology scaling 的准确性**：使用 ITRS 2007 的 scaling factor 将 65nm A8 和 32nm i7 标准化到 45nm，但不同 technology node 的 device type（HP vs. LOP vs. LSTP）差异未被考虑。这可能对 leakage power 的 scaling 引入系统性偏差。

- **结论的有效范围**：论文指出在非常低性能的嵌入式处理器上（如 ATmega324PA），x86 ISA 的 overhead 是 "clearly untenable"。那么在另一个极端——超高性能的 server 处理器上（如当时的 Xeon E5 或后来的 Zen 架构），结论是否仍然成立？特别是当 core 变得极度宽发射（6-8 wide）时，x86 decode 的 power 比例是否会上升？

- **ISA extensions 的影响**：论文禁用了 SIMD 和 machine-specific optimizations。在实际应用中，ISA extensions（如 AVX-512、ARM SVE）对 performance 和 energy efficiency 的影响可能远大于 base ISA 的 RISC/CISC 差异。这是否意味着未来 ISA 的竞争力更多取决于 extensions 的生态系统，而非 base ISA 的设计哲学？
