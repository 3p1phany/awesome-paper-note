---
title: "Challenges and Research Directions for Large Language Model Inference Hardware"
authors: "Xiaoyu Ma, David Patterson"
venue: "IEEE Computer 2025"
year: 2025
---

# Challenges and Research Directions for Large Language Model Inference Hardware

## 基本信息

- **发表**：IEEE Computer, 2025
- **作者单位**：Google DeepMind
- **论文性质**：Vision / Position Paper（非传统实验论文，属于方向性综述与展望）

## 一句话总结

> 针对 LLM inference 的 memory 和 latency 瓶颈，提出 HBF、PNM、3D stacking、低延迟互连四个硬件研究方向。

## 问题与动机

### 核心问题

LLM inference 正在成为 AI 产业最大的硬件挑战：模型服务成本高昂（OpenAI 2024年预计亏损约 50 亿美元），而新兴趋势（MoE、Reasoning、Multimodal、Long context、RAG、Diffusion）进一步加剧了推理的资源需求。当前 GPU/TPU 架构本质上是为 training 设计的，对 inference（尤其是 Decode 阶段）存在系统性的不匹配。

### 两大 Decode 挑战

**Challenge #1: Memory**
- Decode 是 autoregressive 的，每步只生成一个 token，天然是 memory bound。
- **Memory Wall 持续恶化**：NVIDIA GPU 的 64-bit FLOPS 从 2012-2022 增长了 80X，但 memory bandwidth 仅增长 17X，差距持续扩大。
- **HBM 成本趋势恶化**：HBM3e 的 $/GB 和 $/GBps 从 2023-2025 增长了约 1.35x（与 DDR4 成本持续下降形成鲜明对比，DDR4 同期 capacity cost 降至 0.54x，bandwidth cost 降至 0.45x）。
- **DRAM density scaling 减速**：从 8Gb 到 32Gb die 的 4x 增长将耗时超过 10 年，而此前每 3-6 年即可实现 4x 增长。
- **纯 SRAM 方案不可行**：Cerebras 和 Groq 尝试用 full-reticle SRAM chip 规避 DRAM 问题，但 LLM 的容量需求很快超出了 on-chip SRAM 的能力，两家公司后来都不得不加装外部 DRAM。

**Challenge #2: End-to-End Latency**
- User-facing inference 需要秒级响应，涉及 time-to-completion 和 time-to-first-token 两个维度。
- Reasoning model 在生成用户可见 token 之前先生成大量 "thought" tokens，进一步拉长 TTFT。
- **互连延迟超越带宽成为关键瓶颈**：LLM inference 需要 multi-chip 系统（因模型权重过大），但 Decode 阶段 batch size 小、消息频繁且体量小，latency 比 bandwidth 更重要。传统 supercomputer interconnect 为 training 优化了 bandwidth，对 inference 的小消息场景不友好。

### 动机总结

论文通过 Table 2 系统梳理了各 LLM 趋势对 memory capacity、memory bandwidth、interconnect latency、compute 的压力分布，指出**除 Diffusion 外的所有趋势都主要压在 memory 和 interconnect latency 上**，而非 compute。因此，论文聚焦于改善 memory 和 interconnect latency 的四个硬件方向，而非增加 compute FLOPS。

## 核心方法

### 关键思路

论文的核心洞察是：**当前 AI 硬件的设计哲学——full-reticle die + 高 FLOPS + 多 HBM stacks + bandwidth-optimized interconnect——与 LLM Decode inference 的需求是系统性错配的。** Decode 阶段真正需要的是更大的 memory capacity、更高的 memory bandwidth-per-watt、以及更低的 interconnect latency。基于此，提出四个互补的研究方向。

### 技术细节

#### ① High Bandwidth Flash (HBF) — 10X 容量

**核心思路**：仿照 HBM 的 stacking 架构，将 flash dies 堆叠起来，在保持接近 HBM 带宽的同时获得 10X 的容量。

**关键参数对比**（Table 3）：

| 指标 | 1 HBF Stack | 1 HBM4 Stack | 1 DDR5 Module | 1 Flash Card |
|------|-------------|--------------|---------------|-------------|
| Capacity | 512 GB | 48 GB | 64 GB | 4096 GB |
| Bandwidth | 1638 GB/s (read) | 1638 GB/s | 51 GB/s | 4 GB/s (read) |
| Power | <80 W | 40 W | 12 W | 50 W |
| GBps/Watt | >20.5 | 41 | 4 | 0.1 |
| Read Latency | 1000s ns | 10-100 ns | 10-100 ns | 10000s ns |
| Bytes per Read | 4096 | 32 | 64 | 4096 |
| Write Endurance | Low | High | High | Low |

**两大限制及应对**：
- **Write endurance 有限**：Flash 的写擦次数有限，因此 HBF 只能存放 inference 时不频繁更新的数据（如 frozen weights、slow-changing context），不适合存放每个 token 都在更新的 KV Cache。
- **Page-based read + 高延迟**：Flash 读操作以 page 为粒度（数十 KB），延迟远高于 DRAM（微秒级）。小粒度读取会降低 effective bandwidth。

**适用场景**：
- 10X weight memory：inference 时权重是 frozen 的，非常适合 HBF，可以支撑如 DeepSeek v3 的 256-expert MoE 等巨型模型。
- 10X context memory：适合 slow-changing context（web corpus、code database、paper corpus），不适合 KV Cache。
- 缩小 inference system 规模：更大的单节点容量意味着更少的芯片数，降低通信开销、提升可靠性。

#### ② Processing-Near-Memory (PNM) — 高带宽

**PIM vs PNM 的关键区分**：
论文强调了一个重要且常被混淆的概念区分——
- **PIM (Processing-in-Memory)**：processor 和 memory 在**同一个 die** 上。
- **PNM (Processing-Near-Memory)**：processor 和 memory 在**相邻但独立的 die** 上。

**PNM 优于 PIM 的核心论据**（针对 datacenter LLM inference，Table 4）：

| 维度 | PIM | PNM | 优势方 |
|------|-----|-----|--------|
| Data movement power | Very low (on-chip) | Low (off-chip but nearby) | PIM |
| Bandwidth per Watt | Very high (5X-10X) | High (2X-5X) | PIM |
| Memory-logic coupling | 同 die | 分离 die | **PNM** |
| Logic PPA | DRAM 工艺，慢且高功耗 | Logic 工艺，性能/功耗/面积更优 | **PNM** |
| Memory density | 与 logic 共享面积，降低 | 不受影响 | **PNM** |
| Commodity pricing | 低量产、少供应商、低密度 | 不受影响 | **PNM** |
| Power/Thermal budget | Logic 在 memory die 上受限严重 | Logic 约束更宽松 | **PNM** |
| Software sharding | 需切分到 32-64 MB bank 粒度 | 切分粒度可达 16-32 GB，大 1000x | **PNM** |

**关键洞察**：PIM 虽然在 bandwidth 和 data movement power 上更优，但对 datacenter LLM 而言，**software sharding 的难度是致命的**——LLM 的 memory structures 很难被切分到 32-64 MB 的 bank 粒度且保持低通信开销。PNM 的 shard 粒度大 1000x，对 LLM 分区友好得多。

**但对 mobile 设备**，PIM 可能反而可行：移动端模型更小、context 更短、batch size = 1，sharding 需求简化，PIM 的能效优势更有价值。

#### ③ 3D Memory-Logic Stacking — 高带宽

**两种实现方式**：
1. **Compute-on-HBM-base-die**：复用 HBM 设计，在 HBM 的 base die 中插入 compute logic。带宽与 HBM 相同，但功耗降低 2-3X（因数据路径缩短）。
2. **Custom 3D solutions**：通过更宽更密的 memory interface 和更先进的 packaging 技术，实现超越 HBM 的带宽和 bandwidth-per-watt。

**关键挑战**：
- **Thermal**：3D 设计的散热更困难（表面积更小）。一种解决思路是限制 compute logic 的 FLOPS，以低时钟频率和电压运行——这对于 arithmetic intensity 本就很低的 LLM Decode 是可接受的。
- **Memory-logic coupling 标准化**：可能需要行业标准来定义 3D compute-logic stacking 的 memory interface。

**开放问题**：
- Memory bandwidth-to-capacity 和 bandwidth-to-FLOPS 的比例与现有系统显著不同，软件如何适配？
- 多种 memory 类型共存的系统中，如何高效映射 LLM？
- 如何在 bandwidth、power、thermal、reliability 之间权衡各种设计选择（如 compute die 在上还是在下、每 stack 的 memory die 数量等）？

#### ④ Low-Latency Interconnect

**核心思路**：重新审视 datacenter interconnect 的 latency-bandwidth trade-off，因为 inference 对 latency 更敏感。

**四个方向**：
- **High-connectivity topology**：Tree、Dragonfly、高维 Tori 等拓扑减少 hop 数，降低延迟（可能牺牲部分带宽）。
- **Processing-in-network**：LLM 常用的 communication collectives（broadcast, all-reduce, MoE dispatch/collect）适合在网络中加速。例如 tree topology + in-network aggregation 可以同时实现低延迟和高吞吐的 all-reduce。
- **AI chip optimization**：
  - 小包直接存入 on-chip SRAM 而非 off-chip DRAM。
  - Compute engine 靠近 network interface 减少传输时间。
- **Reliability co-design**：
  - 本地 standby spare 减少故障迁移延迟。
  - 若 LLM inference 不需要完美通信，可在消息超时时使用 fake data 或先前结果替代，而非等待 straggler。

### 与现有工作的区别

1. **vs. 传统 GPU/TPU 架构**：论文明确指出当前 full-reticle die + 高 FLOPS 的设计哲学与 Decode inference 需求不匹配。不是增量改进，而是呼吁架构层面的重新思考。
2. **vs. Samsung HBM-PIM / SK Hynix GDDR-PIM**：论文认为 PIM 对 datacenter LLM 不如 PNM，核心原因是 software sharding 和 thermal 限制。
3. **vs. SanDisk HBF 原始提案**：论文将 HBF 放在 LLM inference 的具体场景中讨论其适用性（weights vs. KV Cache vs. slow-changing context），比原始提案更聚焦。

## 实验评估

### 说明

本文是 vision/position paper，**没有传统意义上的仿真实验**。论文的论据主要基于：
- HBM 与 DDR 的历史成本趋势数据（Figure 3，数据源为 Citi Research 2024）
- 各内存技术的规格参数对比（Table 1, Table 3）
- PIM vs. PNM 的定性对比分析（Table 4）
- LLM 各趋势对硬件需求的定性映射（Table 2）
- 附录中 1957-2024 年 DDR DRAM 历史价格数据（来源 jcmit.net/memoryprice, John C. McCallum）

### 关键定量数据点

- HBM3e 的 $/GB 和 $/GBps 从 2023-2025 增长约 **1.35x**。
- DDR4 的 capacity cost 从 2022-2025 降至 **0.54x**，bandwidth cost 降至 **0.45x**。
- NVIDIA GPU 64-bit FLOPS 2012-2022 增长 **80X**，bandwidth 仅增长 **17X**。
- HBF 提供约 **10X** 于 HBM 的容量（512 GB vs 48 GB per stack），带宽相当（1638 GB/s）。
- 3D stacking 的 compute-on-HBM-base-die 方案可降低功耗约 **2-3X**。
- PNM 的 sharding 粒度约为 PIM 的 **1000x**（16-32 GB vs 32-64 MB）。

## 审稿人视角

### 优点

1. **问题定位精准且及时**：准确抓住了 LLM inference（尤其是 Decode 阶段）的核心硬件瓶颈——memory 和 latency 而非 compute。这与业界的实际痛点高度吻合，且随着 MoE、Reasoning model 等趋势的发展，问题只会更加严峻。

2. **PIM vs PNM 的清晰区分具有指导意义**：学术界经常混淆 PIM 和 PNM 的概念，论文给出了简洁但严格的定义（same die vs. separate dies），并从 software sharding、thermal、PPA 等多维度论证了 PNM 对 datacenter LLM 更优的判断，具有实践价值。

3. **HBF 作为研究方向的洞察力强**：将 flash 的容量优势与 HBM 的带宽优势结合，并具体分析了 LLM 中哪些数据适合放在 HBF（frozen weights, slow-changing context）、哪些不适合（KV Cache），展示了对 inference workload 特性的深入理解。

4. **Performance/cost metrics 的强调**：呼吁使用 performance/TCO、performance/power、performance/CO2e 等现代指标，而非单纯追求 FLOPS，这对引导学术研究关注实际部署场景非常重要。

5. **作者影响力与行业视角**：David Patterson 作为计算机体系结构领域的泰斗级人物（RISC 先驱、UC Berkeley 荣休教授），其行业洞察和号召力能够有效推动学术界关注这些方向。论文开头指出 ISCA 2025 工业界论文占比不足 4%（vs. 1976 年的约 40%），呼吁弥合学术与工业之间的鸿沟。

### 不足

1. **缺乏定量评估和仿真验证**：作为 vision paper 可以理解，但论文提出的四个方向都没有任何 roofline analysis、cycle-level simulation 或 analytical modeling 的定量支撑。例如：
   - HBF 的 page-based read（4096 bytes, 微秒级延迟）对 LLM Decode 中 weight loading pattern 的实际影响有多大？是否会成为新的瓶颈？
   - 3D stacking 的 thermal budget 限制了多少 FLOPS？在 Decode 的 arithmetic intensity 下是否足够？
   - PNM 的 2X-5X bandwidth 提升在实际 LLM workload 下能转化为多少 throughput/latency 改善？
   论文自己在 Conclusion 中也提到需要 roofline-based performance simulator，但未提供任何初步结果。

2. **对 HBF 的挑战讨论不够深入**：
   - Flash 的 read latency 是微秒级（比 DRAM 高 10-100x），论文未讨论这对 Decode 每步的 iteration latency 影响。即使 bandwidth 足够，高 latency 可能导致 pipeline stall。
   - Write endurance 的具体约束未量化：inference 时 weights 确实是 frozen 的，但 MoE 的 expert routing、model update（如 LoRA fine-tuning）等场景下的 write 频率是否在 flash endurance 承受范围内？
   - 未讨论 flash 的可靠性问题（read disturb、data retention）在高频读取场景下的影响。

3. **PIM vs PNM 的对比过于倾向 PNM**：
   - 论文承认 PIM 在 bandwidth 和 data movement power 上更优，但以 software sharding 难度为由否定了 PIM 在 datacenter 的价值。然而，近年的 PIM 研究（如 Samsung AIM、UPMEM）已经在 LLM inference 中展示了可行的 sharding 方案，论文未充分讨论这些进展。
   - PIM 的 bank-level 并行性对于 attention 计算中的 KV Cache 访问可能恰好是合适的粒度，论文未考虑这一点。

4. **互连方向的讨论较为浅显**：四个方向中，Low-Latency Interconnect 的讨论最为粗略，基本停留在列举已知技术（dragonfly、in-network reduction、SHARP）的层面，缺乏对 LLM inference 特定 communication pattern 的深入分析。例如：
   - MoE 的 all-to-all dispatch/collect 与传统 all-reduce 有本质区别，对拓扑和路由有不同要求，论文未展开。
   - "用 fake data 替代 straggler message" 这一建议过于简略，未讨论对模型输出质量的影响。

5. **Mobile 场景的讨论浮于表面**：论文多次提到这些方向也适用于 mobile 设备，但每次都只用一两句话带过，未深入分析 mobile 的功耗、面积、热约束下的具体 trade-off。

6. **未讨论 software-hardware co-design 的具体路径**：论文聚焦硬件，但多次承认 software 适配是关键（如 HBF 的 page-based read 需要 software 处理、PNM 的 programming model、3D stacking 的 memory mapping）。如果 software stack 跟不上，硬件创新的价值将大打折扣。论文在 Conclusion 中提到需要 simulator，但未讨论 programming model、compiler support、runtime system 等同样关键的软件层面。

### 疑问或值得追问的点

- HBF 的 read latency（微秒级）如何与 Decode 的 iteration time（通常也是微秒到毫秒级）匹配？是否需要 prefetching 或 caching 机制来隐藏 latency？
- 在多种 memory 类型共存的异构系统中（HBM + HBF + DDR），data placement 和 migration 策略如何设计？这是否会引入新的复杂性？
- 3D stacking 的 thermal 问题是否可以通过 chiplet 架构（如 AMD 的方案）更灵活地解决？论文引用了 AMD 的 concept 但未深入对比。
- 论文提出的四个方向是否存在优先级？对于资源有限的学术研究者，应该优先投入哪个方向？
- 论文是否低估了 software optimization（如 speculative decoding、quantization、KV Cache compression）对 hardware pressure 的缓解作用？如果 software 足够好，是否还需要如此激进的硬件变革？
