---
title: "A Mess of Memory System Benchmarking, Simulation and Application Profiling"
authors: "Pouya Esmaili-Dokht, Francesco Sgherzi, Valeria Soldera Girelli, et al."
venue: "MICRO 2024 (Best Paper Runner-up)"
year: 2024
---

# A Mess of Memory System Benchmarking, Simulation and Application Profiling

## 基本信息

- **发表**：MICRO 2024（57th IEEE/ACM International Symposium on Microarchitecture），Best Paper Runner-up
- **机构**：Barcelona Supercomputing Center (BSC), Universitat Politècnica de Catalunya (UPC), Micron Technology
- **开源**：Mess benchmark（GitHub: bsc-mem/Mess-benchmark）, Mess simulator（GitHub: bsc-mem/Mess-simulator）

## 一句话总结

> 提出基于 bandwidth–latency curve 族的统一框架 Mess，同时覆盖内存系统 benchmarking、simulation 和 application profiling，并揭示了主流 DRAM simulator 存在严重仿真误差。

## 问题与动机

内存系统的 benchmarking、simulation 和 application profiling 三个方面本质上紧密关联，但长期以来被独立、割裂的工具链处理：

1. **Benchmarking 的碎片化**：现有 benchmark 要么只测 unloaded latency（LMbench、Google multichase），要么只测 max sustainable bandwidth（STREAM、HPCG）。很少有工具覆盖整个 bandwidth–latency 空间，更不考虑不同 read/write ratio 对性能的影响。Intel MLC 虽能测 loaded latency，但只提供稀疏的 traffic composition 分析。

2. **Simulator 的不可信**：被学术界广泛信赖的 cycle-accurate memory simulator（DRAMsim3、Ramulator、Ramulator 2）虽然通过了 JEDEC timing 合规性验证，但从未被系统地与真实硬件的 bandwidth–latency 行为做全面对比。论文发现这些 simulator 存在严重问题——unloaded latency 低至 4 ns（真实系统 89 ns），bandwidth 超出理论最大值 1.8×，IPC 仿真误差高达数十个百分点。

3. **新技术支持滞后**：从 DDR5 量产到 public simulator 支持间隔了 3 年。CXL memory expander 等新兴设备更缺乏可靠的性能模型。

4. **Application profiling 粒度不足**：现有 profiling 方法（Roofline、Top-down CPI stack）将 application 简单分类为 memory-bound 或 compute-bound，但不能精确描述 application 在内存性能空间中的具体位置和动态变化。

## 核心方法

### 关键思路

Mess 的核心洞察是：**内存系统性能可以用一族 bandwidth–latency 曲线完整刻画**，每条曲线对应特定的 read/write traffic ratio。这个统一的性能表征可以同时服务于 benchmarking（刻画真实/仿真硬件）、simulation（作为分析式内存模型的输入）和 application profiling（定位 application 在曲线上的运行时位置）。

关键 insight 在于：不需要模拟复杂的 DRAM 内部时序（command scheduling、row buffer management 等微架构细节），仅用从真实硬件测量得到的 bandwidth–latency 关系就能达到远超 cycle-accurate simulator 的仿真精度。

### 技术细节

#### 1. Mess Benchmark 设计

**测量方法论**：每条 bandwidth–latency 曲线由数十个测量点构成，覆盖从 unloaded 到 fully-saturated 的全范围。

- **Latency 测量**：在一个 CPU core（或一个 GPU SM）上运行 pointer-chase benchmark。Pointer-chase 通过一系列数据依赖的 back-to-back load 指令实现，每条 load 的地址取决于上一条 load 的结果，因此执行被串行化。平均内存访问延迟 = 总执行时间 / 指令数。数组大小超过 LLC 容量以保证 main memory access，每个元素占一整条 cache line（64B）以消除 spatial locality，遍历顺序随机化以对抗 prefetching。
- **Bandwidth 生成**：在其余 CPU core 上并发运行 traffic generator。Traffic generator 直接用汇编编写，通过两个独立数组分别执行 load 和 store 操作。Bandwidth 的调节通过在 load/store 序列中插入可配置长度的 nop dummy loop 实现——nop 越多，memory operation 发射率越低，bandwidth 越低。
- **Bandwidth 测量**：通过 uncore hardware counter（如 Linux perf）读取实际内存带宽。
- **Read/Write ratio 覆盖**：100%-load 产生 100%-read traffic；100%-store 在 write-allocate 策略下产生 50%-read/50%-write traffic（因为每次 store 需先 read-allocate cache line 再 write back）。通过混合不同比例的 load/store 指令覆盖全范围。
- **OS 开销排除**：使用 huge page 减少 TLB miss，运行时通过 hardware counter 监控 TLB miss 和 page walk 并从延迟测量中扣除。

**跨平台支持**：benchmark 核心用汇编编写，覆盖 x86、ARM、Power、RISC-V 和 NVIDIA PTX。

**验证**：与 LMbench/Google multichase（unloaded latency）、Intel Advisor（max bandwidth，误差 <1%）、Intel MLC（loaded latency，Mess 约低 <5%，因为排除了 OS 开销）交叉验证。

**新发现——bandwidth decline（"wave form"）**：在部分平台上（AMD Zen2、Intel Skylake/Cascade Lake、Graviton 3、Sapphire Rapids、H100），继续增加 memory access rate 反而导致 measured bandwidth 下降，同时 latency 持续上升。论文在 Cascade Lake 上通过 row-buffer hardware counter 确认：bandwidth decline 与 row-buffer miss rate 的剧增高度相关。进一步通过拔掉 4/6 个 DIMM 增大 memory pressure 来加剧这一现象以验证因果性。

#### 2. Mess Simulator 设计

**核心思路**：不模拟 DRAM 内部时序细节，而是用 feedback control loop 让仿真中的 application 在 bandwidth–latency 曲线上寻找自洽的工作点。

**控制回路工作流程**（每个 simulation window = 1000 memory operations）：

1. **初始估计**：从某个初始位置 (messBW_i, Latency_i) 开始，根据当前 read/write ratio 选择对应曲线。
2. **CPU 仿真**：CPU simulator 以 Latency_i 作为 memory latency 仿真执行，产生 memory traffic。
3. **监测 bandwidth**：在 window 结束时读取 CPU simulator 实际产生的 bandwidth cpuBW_i。
4. **一致性检查**：比较 cpuBW_i 与 messBW_i。
   - 若一致：说明 latency 和 bandwidth 自洽，下一个 window 继续使用相同 latency。
   - 若不一致：说明 application 行为变化（如 phase change），需要调整。
5. **渐进调整**：`messBW_{i+1} = messBW_i + convFactor × (cpuBW_i − messBW_i)`。不一步到位，而是按比例逐步逼近，基于经典控制论中的 proportional–integral controller 机制。
6. **Latency 更新**：从曲线上查 `Latency_{i+1}`，然后扣除 CPU 侧已模拟的延迟（cache hierarchy、NoC 等）：`Latency^{Memory}_{i+1} = Latency_{i+1} − Latency^{CPU}_{i+1}`。

**集成方式**：通过标准的 CPU–memory simulator 接口集成到 ZSim、gem5 和 OpenPiton Metro-MPI。唯一需要的参数就是 bandwidth–latency 曲线族。

**对新技术的快速支持**：曲线可来自真实硬件测量或厂商提供的硬件模型（如 Micron 的 SystemC model 提供的 CXL memory expander 曲线），从而绕过漫长的 cycle-accurate model 开发周期。论文展示了 Mess 作为**首个支持 CXL memory expander 仿真**的 memory simulator。

#### 3. Mess Application Profiling

集成到 BSC 的生产级 HPC 性能分析工具 Extrae + Paraver 中：

- **定位**：以 10 ms 粒度采样 application 的 memory bandwidth（通过 hardware counter），将采样点投射到 Mess bandwidth–latency 曲线上。
- **Memory stress score**：对曲线上每个点计算 0~1 的 stress score，由 latency 和曲线斜率的加权和组成。斜率反映 bandwidth 变化对 latency 的敏感性——在陡峭区域，微小的 bandwidth 变化会导致 latency 剧增。
- **Timeline 分析**：stress score 嵌入 Paraver trace，可与 MPI 调用序列、计算 phase 长度、source code 关联分析。
- **Overhead**：< 1%。

### 与现有工作的区别

| 维度 | 现有工作 | Mess 的差异 |
|------|---------|-----------|
| **Benchmarking** | LMbench/multichase 只测 unloaded latency；STREAM 只测 max bandwidth；Intel MLC 只测稀疏 traffic composition | 全面覆盖 bandwidth–latency 空间，数十条曲线 × 每条数十个测量点，区分 read/write ratio 影响 |
| **Simulation** | DRAMsim3/Ramulator 做 cycle-accurate DRAM timing 仿真，通过 JEDEC timing 验证 | 不模拟 DRAM 内部细节，而用真实 bandwidth–latency 曲线 + feedback control，精度远超 cycle-accurate simulator |
| **Application profiling** | Roofline model 只关注 compute vs. memory bound 的二分类；Top-down CPI stack 只区分 latency vs. bandwidth stall | 将 application 精确定位在 bandwidth–latency 空间中，支持 10 ms 粒度的 timeline 分析并可关联 source code |

## 实验评估

### 实验设置

- **真实硬件平台**（8 个）：
  - Intel Skylake（24 cores, 6×DDR4-2666, 128 GB/s）
  - Intel Cascade Lake（16 cores, 6×DDR4-2666, 128 GB/s）
  - AMD Zen2 EPYC 7742（64 cores, 8×DDR4-3200, 204 GB/s）
  - IBM Power 9（20 cores, 8×DDR4-2666, 170 GB/s）
  - Amazon Graviton 3（64 cores, 8×DDR5-4800, 307 GB/s）
  - Intel Sapphire Rapids（56 cores, 8×DDR5-4800, 307 GB/s）
  - Fujitsu A64FX（48 cores, 4×HBM2, 1024 GB/s）
  - NVIDIA H100（132 SMs, 4×HBM2E, 1631 GB/s）

- **CPU Simulator**：
  - ZSim（event-based，modeling Intel Skylake 24-core）
  - gem5（cycle-accurate，modeling Graviton 3 64-core Neoverse N1）
  - OpenPiton Metro-MPI（RTL-level，64-core Ariane RISC-V）

- **Memory Simulator Baselines**：
  - gem5 simple memory model、gem5 internal DDR model
  - ZSim fixed-latency、M/D/1 queue、internal DDR model
  - DRAMsim3（trace-driven 和 ZSim-driven）
  - Ramulator（trace-driven 和 ZSim-driven）
  - Ramulator 2（trace-driven 和 gem5-driven）

- **Workload**：STREAM（Copy/Scale/Add/Triad）、LMbench、Google multichase、HPCG、SPEC CPU2006（CXL 部分）

### 关键结果

1. **现有 simulator 仿真精度极差**：
   - gem5 simple memory model 的 unloaded latency 仅 4–49 ns（真实 Graviton 3 为 122 ns），比值达 3.7×。
   - ZSim+Ramulator 的 bandwidth 超出理论最大值 1.8×，全域 latency 仅 25 ns。
   - gem5+Ramulator 2 的 max bandwidth 仅 126 GB/s，不到真实系统 292 GB/s 的一半；且作为"最复杂最受信任"的模型，仿真误差反而最大。
   - DRAMsim3 的 unloaded latency 为 52–63 ns（真实 89–109 ns），不建模 saturated bandwidth 区域。

2. **Mess simulator 精度大幅领先**：
   - ZSim+Mess 在 STREAM/LMbench/multichase 上平均 IPC error 仅 **1.3%**，而 Fixed-latency > 80%，Ramulator > 80%，DRAMsim3 约 21%，Internal DDR 约 18%。
   - gem5+Mess 平均 IPC error 仅 **3%**，而 Simple memory 30%，Internal DDR 15%，Ramulator 2 高达 52%。

3. **Mess simulator 速度极快**：
   - ZSim+Mess 仅比 fixed-latency model 慢 26%。
   - ZSim+Mess 比 ZSim+Ramulator 快 **13×**，比 ZSim+DRAMsim3 快 **15×**。

4. **CXL 仿真的独特发现**：CXL 作为 full-duplex interconnect，balanced read/write traffic 性能最好，100%-read 或 100%-write 反而性能下降——与 DDRx 行为完全相反。基于 Micron SystemC 模型的 Mess CXL 仿真在 ZSim 和 gem5 上均高度匹配厂商模型。

5. **意外 bug 发现**：Mess benchmark 在 OpenPiton Metro-MPI 上检测到异常高的 write traffic，追溯到 OpenPiton 框架生成的 coherency protocol 中的 bug（不只 evict dirty lines，而是 evict 所有 cache lines），已被开发者确认。

### 结果分析

**误差根源分析**（Section IV-D）：

论文通过 trace-driven 仿真（排除 CPU simulator 和接口误差）隔离 memory simulator 本身的问题：

- **ZSim 接口问题**：trace-driven DRAMsim3 和 Ramulator 的 bandwidth–latency 趋势优于 ZSim-driven 版本，说明部分误差来自 event-based CPU simulator 与 cycle-accurate memory model 的接口整合，与此前文献 [49, 80, 81] 的发现一致。
- **Row-buffer utilization 建模不准**：DRAMsim3 在大部分实验中 row-buffer hit rate 高达 84–93%（真实系统随 bandwidth 增长从 84% 降至更低）。Ramulator 在 >40% write traffic 下 hit rate 远高于真实系统。更重要的发现是：**真实系统中，低中 bandwidth 区域 write traffic 导致的 row-buffer 利用率下降并不直接转化为更高的 latency**——真实系统能 mask 这种 contention，而 DRAMsim3 和 Ramulator 均不具备这种能力。
- **Ramulator 2 本身误差极大**：trace-driven 结果与 gem5-driven 趋势一致，max bandwidth 不到真实系统一半，证实主要误差源于 Ramulator 2 自身。

**跨平台特性分析**：

- 不同平台的 unloaded latency 差异显著（85–363 ns），且与 DRAM 技术无直接对应关系（如 AMD Zen2 使用与 Intel Cascade Lake 相同 timing 的 DDR4，但 unloaded latency 高出近 30 ns），因为 load-to-use latency 包含了 on-chip 部分（cache hierarchy、NoC）。
- AMD Zen2 是唯一的异常：saturated bandwidth range 显著偏低（57–71%），且 write traffic 影响不符合通常模式（max write rate 性能接近 100%-read，而 mixed traffic 如 60%-read/40%-write 反而最差）。

## 审稿人视角

### 优点

1. **统一框架的 vision 非常出色**。将 benchmarking、simulation、profiling 三个长期割裂的领域用 bandwidth–latency 曲线这一简洁抽象统一起来，理念清晰且实用价值极高。框架的每个组件都可独立使用，但组合后产生 1+1+1>3 的效果。

2. **对主流 simulator 的全面"打脸"极具社区价值**。DRAMsim3、Ramulator 系列被学术界广泛使用且默认可信，论文以大量实验和严谨的控制变量（trace-driven vs. execution-driven、cross-validation with hardware counters）揭示了它们的严重不准确性，并分析了 row-buffer modeling 和 CPU–memory 接口两大误差来源。这对整个 memory system simulation 社区是一次重要的 wake-up call。

3. **实验覆盖面极广**。8 个真实硬件平台横跨 Intel/AMD/IBM/Fujitsu/Amazon/NVIDIA，涵盖 DDR4/DDR5/HBM2/HBM2E 和 CXL 五种内存技术，3 个 CPU simulator（event-based、cycle-accurate、RTL），5+ 种内存模型。如此大规模的跨平台比较在 memory system 文献中罕见。

4. **工程完整度极高**。全部代码开源（benchmark + simulator 均已发布），覆盖 5 种 ISA，已集成到生产级 HPC 工具（Extrae/Paraver）。CXL 仿真基于 Micron 厂商 SystemC 模型，具备产业可信度。Artifact appendix 提供了完整的复现指南。

5. **意外发现增强了论文说服力**。Mess 顺带检测到 OpenPiton coherency protocol bug 和真实硬件的 bandwidth decline 现象，表明该 benchmark 框架确实具备超越设计初衷的诊断能力。

### 不足

1. **Mess simulator 的适用性边界不够清晰**。Feedback control loop 基于当前 read/write ratio 选择曲线并假设 application 在曲线上有稳定工作点。但对于 access pattern 快速变化（如不同 phase 的 row-buffer hit rate 差异巨大）的 workload，1000 operations 的 window 是否足够？论文未充分讨论 convergence factor 的敏感性，也未分析控制回路在 phase transition 期间的暂态误差。

2. **Benchmark 的 access pattern 代表性存疑**。Mess traffic generator 中每个 process 对 array 做顺序遍历，多 process 并发形成"复杂"的 interleaving，但这与真实 application 的 access pattern（如 irregular/random access、pointer-chasing data structures）差距较大。论文报告了 row-buffer hit/miss rate 的范围（35/43/22% 到 84/13/3%），但未分析这是否足以覆盖真实 workload 的多样性。如果某个 application 的 row-buffer 行为显著偏离 Mess benchmark 测量时的模式，Mess simulator 的精度是否仍然可靠？

3. **评估 workload 过于简单且同质化**。用来验证 simulator 精度的 benchmark 仅限于 STREAM（streaming access）、LMbench 和 Google multichase（pointer-chase），均为纯 memory-intensive 且 access pattern 单一的 micro-benchmark。缺乏对 SPEC、PARSEC、GAP 等真实混合 workload 的 IPC 验证。CXL 部分虽用了 SPEC CPU2006，但也只在 Appendix 中简要讨论。

4. **对 row-buffer 误差分析停留在现象层面**。论文检测到 DRAMsim3 和 Ramulator 的 row-buffer hit rate 与真实系统不匹配，也观察到真实系统能 "mask" row-buffer contention，但未深入分析这种 masking 的微架构机制（可能涉及 out-of-order execution 的 memory-level parallelism、MC 的 reordering/scheduling 策略等）。作为 Best Paper Runner-up，在根因分析上还可以更深入。

5. **Mess simulator 本质上是一种 interpolation/lookup 方法**。它假设 application 在曲线上的行为可以由 instantaneous bandwidth 和 read/write ratio 这两个变量充分描述。但真实内存系统性能还受 address distribution（bank/rank/channel-level parallelism）、burst pattern、request reordering 等因素影响。论文未讨论这些额外维度何时可能导致 Mess 的 bandwidth–latency 曲线不再是好的 predictor。

### 疑问或值得追问的点

- **Convergence factor 如何选择？** 论文提到这是 user-defined parameter，但未讨论选择策略。过大可能振荡，过小可能跟不上 application phase change。是否做过 sensitivity analysis？
- **Multi-socket / NUMA 场景**？论文所有实验均为单 socket。在 NUMA 系统中 remote memory access 的 bandwidth–latency 特性不同，Mess 如何处理？
- **Write-allocate evasion**（如 Intel Ice Lake SP 的 non-temporal store 优化）对 Mess benchmark 和 simulator 的影响如何？论文将此列为 future work 但未提供任何初步结果。
- **HBM 的 bandwidth–latency 曲线与 application access pattern 的关联性**：HBM 具有极高的 bank-level parallelism，其 bandwidth–latency 行为是否对 access pattern 更敏感？A64FX 的 STREAM 利用率仅 49–55%（远低于 Graviton 3 的 78–82%），这种差异在 Mess simulator 中如何体现？
- **面向 LLM inference workload 的适用性**：KV cache 访问呈现 irregular、bursty、read-dominated 特征，与 Mess benchmark 的 traffic pattern 差异较大。Mess 的 bandwidth–latency 曲线能否为 HBM/HBM4 上的 LLM serving 提供准确的性能预测？
