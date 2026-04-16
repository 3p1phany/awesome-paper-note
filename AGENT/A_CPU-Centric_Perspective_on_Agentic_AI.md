---
title: "A CPU-Centric Perspective on Agentic AI"
authors: "Ritik Raj, Hong Wang, Tushar Krishna"
venue: "arXiv preprint (arXiv:2511.00739v2)"
year: 2025
---

# A CPU-Centric Perspective on Agentic AI

## 基本信息

- **作者**：Ritik Raj (Georgia Institute of Technology, 实习于 Intel), Hong Wang (Intel), Tushar Krishna (Georgia Institute of Technology)
- **发表**：arXiv preprint, 2025年11月（v2版本 2025-11-29）
- **代码开源**：https://github.com/ritikraj7/cpu-centric-agentic-ai

## 一句话总结

> 首次系统性地从 CPU 视角剖析 Agentic AI 的性能瓶颈，揭示 tool processing 占延迟高达 90.6%，并提出 CGAM 和 MAWS 两种调度优化。

## 问题与动机

**要解决的核心问题**：Agentic AI 框架在 monolithic LLM 之上叠加了 orchestrator、external tools（web search、Python interpreter、database retrieval 等），形成多组件闭环。然而，现有 AI 系统优化工作几乎全部聚焦于 GPU 端（GPU kernel、KV-cache scheduling），完全忽略了 agentic workload 中大量 CPU-bound 的 tool processing 环节对系统整体延迟、吞吐和能耗的影响。

**问题的重要性**：

- 论文实测表明 tool processing（运行在 CPU 上）可占端到端延迟的 **90.6%**（Haystack RAG + ENNS retrieval 场景），远超 LLM inference 本身。
- Quinn et al. (2025) 的研究显示，在 200 GB 文档语料上的 ENNS 占 RAG 端到端延迟超过 75%。
- Xu et al. (2024) 证明 tool partial execution 可将请求完成延迟降低 38.8%，间接说明 tool execution 是延迟的显著组成。

**现有工作的不足**：

- Kim et al. (2025) 从 GPU-centric 视角 profiling agentic workload，未暴露 CPU bottleneck，且所用 tool 多为可轻松并行化的 API call。
- Asgar et al. (2025) profiling 了 agentic AI 但仅关注 external tool call，缺乏本地 CPU overhead 的系统性分析。
- Recasens et al. (2025) 提出了 decode 阶段的 micro-batching，但未考虑 CPU 端的 micro-batching。

## 核心方法

### 关键思路

论文的核心洞察是：**Agentic AI 的性能瓶颈已经从 GPU inference 转移到了 CPU-bound 的 tool processing**。基于这一观察，论文首先建立了一套从系统视角出发的 agentic AI 三维分类体系，然后选取五种代表性 workload 进行 latency/throughput/energy 的全栈 profiling，最终提出两种针对 CPU-GPU 协同的调度优化策略。

### 技术细节

#### 1. 系统级三维分类体系（Characterization）

论文提出三个**正交的分类维度**，每个维度都直接影响系统级性能指标：

| 维度 | 分类 | 典型代表 | 系统影响 |
|------|------|---------|---------|
| **Orchestrator** | LLM-orchestrated vs. Host-orchestrated | ReAct, AutoGPT vs. LangChain, Haystack | 决定控制流开销在 GPU 还是 CPU |
| **Path** | Static vs. Dynamic | LangChain, Haystack vs. Tree-of-Thoughts, Reflexion | 静态路径可预测、可优化；动态路径引入运行时决策开销 |
| **Flow/Repetitiveness** | Single-step vs. Multi-step | RAG, CoT prompting vs. WebArena, AgentBench | Multi-step 带来迭代式 CPU-GPU 交互，放大瓶颈 |

这一分类的价值在于从**算法特征映射到系统行为**——orchestrator 类型决定了谁承担调度开销，path 类型决定了流水线是否可静态优化，repetitiveness 决定了 CPU-GPU 交互的频率。

#### 2. 五种代表性 Workload

| Workload | Orchestrator | Path | Flow | AI Model | 核心 CPU Tool |
|----------|-------------|------|------|----------|--------------|
| Toolformer | LLM | Dynamic | Single-step | GPT-J 6B | WolframAlpha API |
| SWE-Agent | LLM | Dynamic | Multi-step | Qwen2.5-Coder-32B | Bash/Python execution |
| Haystack RAG | Host | Static | Single-step | GPT-OSS-20B | ENNS retrieval (FAISS, 305GB C4) |
| ChemCrow | LLM | Dynamic | Multi-step | GPT-4-0613 | Arxiv/Pubmed literature search |
| LangChain | Host | Static | Single-step | GPT-OSS-20B | Web search + LexRank summarization |

选择理由：覆盖了 factual QA、coding、scientific research 等高难度应用场景；跨越不同 model size（6B 到 GPT-4）、不同 orchestration pattern 和不同 tool integration 策略；来源于 peer-reviewed 研究和广泛使用的开源仓库。

**关于 LexRank summarizer 的设计选择**值得注意：论文选择 CPU-based LexRank 而非 LLM-based summarizer，理由有三——(1) LLM summarizer 在 XSum 上 73–79% 的摘要含幻觉；(2) LexRank 在 DUC-2004 上 ROUGE-1 与 LLM 差距仅 0.05，在 BillSum 上甚至超越；(3) GPU 是昂贵资源，CPU summarization 更经济。

#### 3. Profiling 关键发现

**Latency（延迟）**：
- Haystack RAG：retrieval 占 84.5%–90.6%，LLM inference ≤ 0.5 s
- Toolformer：WolframAlpha API 调用 1.4–1.7 s，两次 inference 各约 1 s
- ChemCrow：literature search 4.0–10.1 s，GPT-4 inference 5.6–7.6 s，总延迟 12.6–18.8 s
- LangChain：web search（最高 4.2 s）或 summarization（最高 3.5 s）可占超半数延迟
- SWE-Agent：Bash/Python execution 占 43.8%–78.7%

**Throughput（吞吐）**：
- **CPU parallelism 对比**：对 LangChain workload，batch size 128 时 multi-processing 比 sequential 加速 26.8×，比 multi-threading 加速 1.6×。原因是 multi-processing 绕过了 Python GIL，减少了同步开销。
- **例外**：Haystack 因 ~300GB 内存占用，multi-processing 的独立内存空间导致低效，反而需要 multi-threading 的 shared memory。
- **GPU bottleneck**：vLLM throughput 在 batch size 64 后趋于饱和，原因是 KV cache 扩张引发 HBM 容量压力和 PCIe bandwidth bottleneck。FlexGen 的数据表明 OPT-175B 的 KV cache 需 1.2 TB，是模型权重的 3.8×。
- **CPU bottleneck**：core over-subscription（进程数超过物理核数）导致 OS scheduler contention 和 context switching 开销。LangChain 在 batch size 128 时 summarization 平均延迟从 2.9 s 涨到 6.3 s。此外还有 cache coherence traffic（MESI 协议下的 cache-line ping-pong）和 NUMA 效应。

**Energy（能耗）**：
- LangChain on FreshQA：batch size 1→128，总动态能耗从 108 J 增至 4114 J（38.1×）
- GPU 动态能耗：86→2307 J（26.8×）
- CPU 动态能耗：22→1807 J（**86.7×**）
- CPU 动态能耗占比从小 batch 的 20% 升至大 batch 的 **44%**
- 结论：CPU multi-processing 的能效远低于 GPU parallelism

#### 4. 优化策略

##### CGAM（CPU and GPU-Aware Micro-batching）—— 面向同构 workload

**核心思想**：识别 throughput saturation point，设定 batch cap $B_{cap}$，将大 batch 拆分为多个 micro-batch 顺序执行。

**Batching Cap 选择**：定义 throughput gain ratio $r(B) = T(B) / T(B/2)$，选取阈值 $\lambda = 1.1$（即 doubling batch size 带来 < 10% 提升时停止）：

$$B_{cap} = \max\{B \in \{2^k : k \in \mathbb{N}\} : r(B) > \lambda\}$$

实验中三个 workload（LangChain r(128)=1.09, Haystack r(128)=1.08, SWE-Agent r(128)=1.10）均选定 $B_{cap} = 64$。

**CGAM 的三重收益**：
1. **~2× P50 延迟改善**：第一个 micro-batch 在约一半总时间内完成，对 tiered serving（分层服务）场景有利
2. **~0.5× KV cache 使用**：任何时刻仅一半 batch 在 GPU 上运行，缓解 GPU memory 压力
3. **~2× CPU 动态能耗降低**：限制活跃核数为 $B_{cap}$

**CGAM$_{overlap}$ 变体**：在第一个 micro-batch 的 CPU 阶段完成后，立即启动第二个 micro-batch 的 CPU 阶段，与第一个 micro-batch 的 GPU 阶段并行执行。Trade-off：P50 延迟略差于 CGAM（因 CPU contention 增加），但 P90 延迟更优（因第二个 micro-batch 更早开始）。对 CPU 和 GPU 延迟接近的 workload（如 LangChain）效果最佳。

##### MAWS（Mixed Agentic Workload Scheduling）—— 面向异构 workload

**动机**：实际场景中同时存在 CPU-heavy（tool-intensive）和 LLM-heavy 的 agentic 请求。若对所有请求统一使用 multi-processing，LLM-heavy 任务会抢占 CPU 核心，导致 CPU-heavy 任务的 over-subscription。

**策略**：自适应地对 CPU-heavy workload 使用 multi-processing（最大化 CPU 利用率），对 LLM-heavy workload 使用 multi-threading（轻量级 vLLM API I/O 并行，释放 CPU 资源给 CPU-heavy 任务）。

**MAWS + CGAM 组合**：在异构场景下先用 MAWS 分流，再对 CPU-heavy 部分施加 CGAM micro-batching。

### 与现有工作的区别

| 对比工作 | 关键差异 |
|---------|---------|
| Kim et al. (2025) | GPU-centric profiling，未暴露 CPU bottleneck；tools 多为可轻松并行的 API call |
| Asgar et al. (2025) | 仅关注 external tool call 的 orchestration 优化，本地 CPU overhead 接近零 |
| Recasens et al. (2025) | 提出 GPU decode 阶段 micro-batching，但未考虑 CPU 端 batching cap 选择、CPU/GPU overlap 和 adaptive multi-processing/multi-threading |

本文的独特之处在于：(1) 首次系统性地将 CPU 作为 agentic AI 的一等公民进行 profiling；(2) 三维分类体系将算法特征映射到系统行为；(3) 优化同时考虑 CPU 和 GPU 两端的资源约束。

## 实验评估

### 实验设置

- **硬件平台**（Profiling + Evaluation）：48-core Intel Emerald Rapids CPU（DDR5，2 threads/core）+ NVIDIA B200 GPU（HBM3e）
- **能耗测量平台**（因设施限制单独测量）：AMD Ryzen Threadripper PRO 7985WX（64 cores）+ NVIDIA H200 GPU。CPU 能耗通过 pyRAPL（RAPL counters），GPU 能耗通过 nvidia-smi 每 100 ms 采样后梯形积分。扣除 idle power（CPU 113W, GPU 115W）得到动态功耗。
- **LLM Serving**：本地 vLLM server（v0.11.0），ChemCrow 使用 OpenAI API（GPT-4-0613）
- **软件环境**：PyTorch 2.8.0, langchain 0.3.27, haystack-ai 2.18.1, chemcrow 0.3.24, mini-swe-agent 1.9.1

- **Workload 与 Benchmark**：
  - Haystack RAG：NQ, HotpotQA, TriviaQA（ENNS top-5 retrieval, FAISS FLAT, 305GB C4 corpus）
  - Toolformer：ASDiv, SVAMP, MAWPS（WolframAlpha calculator）
  - ChemCrow：nicotine, warfarin, caffeine, aspirin 相关 chemistry QA（Arxiv/Pubmed search）
  - LangChain：FreshQA, MusiQue, QASC（web search + LexRank summarization + GPT-OSS-20B）
  - Mini-SWE-Agent：APPS, BigCodeBench, DS-1000（Bash/Python execution + Qwen2.5-Coder-32B）

- **Evaluation 配置**：Closed-loop arrival（所有 B 个请求在 t=0 同时到达）；CGAM 和 MAWS 分别在 B=128 和 B=128 评估；MAWS+CGAM 在 B=256 评估。统计方差约 5%。

- **Baseline**：
  - LangChain, SWE-Agent: multi-processing（每个 process 处理 1 个请求）
  - Haystack: multi-threading（因 ~300GB 共享内存需求）

### 关键结果

1. **CGAM P50 延迟改善**：LangChain 2.11×（11.21s→5.32s），Haystack 1.94×（42.87s→22.12s），SWE-Agent 1.72×（65.08s→37.82s），使用 3/4 的 CPU 核（96 cores），P90 基本持平。

2. **CGAM$_{overlap}$ trade-off**：LangChain P50 1.69×/P90 1.33×，Haystack P50 1.82×/P90 1.15×，SWE-Agent P50 1.37×/P90 1.16×。CPU-GPU 延迟越接近的 workload（LangChain）P90 改善越大。

3. **MAWS（128 混合任务，半数 CPU-heavy 半数 LLM-heavy）**：P99 延迟改善 1.17×，P50 基本持平。

4. **MAWS+CGAM（256 混合任务）**：CPU-heavy 任务 P50 加速 2.10×，LLM-heavy 任务 P50 加速 1.20×，全局 P50 加速 1.41×，全局 P99 改善 1.15×。

### 结果分析

- CGAM 的 P50 改善接近理论预期的 ~2×，偏差来自 CPU-GPU 执行比例不均和 throughput saturation 对两端组件的不同影响。
- CGAM$_{overlap}$ 在 LangChain 上效果最好，因为其 CPU（summarization）和 GPU（inference）延迟相对接近，overlap 收益最大；而 Haystack 和 SWE-Agent 的 CPU 延迟远大于 GPU，overlap 收益有限。
- MAWS 的核心价值在于避免 LLM-heavy 任务的 multi-processing 抢占 CPU 核心，从而保护 CPU-heavy 任务的性能。
- 论文未做 sensitivity analysis on $\lambda$ threshold，仅测试了 $\lambda = 1.1$。
- 能耗方面，CGAM 估算可节省 ~1.5× CPU 动态能耗（假设执行期间功耗均匀）。

## 审稿人视角

### 优点

1. **Problem formulation 新颖且实用**：在 agentic AI 快速发展的背景下，首次系统性地将 CPU 作为性能分析的一等公民。随着 agent 框架的广泛部署，CPU-bound tool processing 确实是一个被严重忽视的瓶颈。三个 Key Takeaway（tool 延迟占主导、CPU/GPU 双侧吞吐瓶颈、CPU 能耗非线性增长）都有实测数据支撑，具有很强的实践指导意义。

2. **Characterization 体系设计合理**：三维正交分类（orchestrator × path × flow）从系统行为角度建模 agentic workload 的多样性，比纯算法分类更有利于系统优化。五个 workload 的选取覆盖了不同分类组合，代表性较好。

3. **Full-stack profiling 比较全面**：同时覆盖 latency timeline、batch throughput scaling 和 energy breakdown 三个维度，且使用了 SOTA 硬件（Emerald Rapids + B200），数据具有时效性和参考价值。

4. **优化方案简洁有效**：CGAM 的 batching cap selection 基于简单的 throughput gain ratio 阈值，易于实现和推广。MAWS 的 adaptive multi-processing/multi-threading 策略直觉清晰且实用。

### 不足

1. **优化方案的 novelty 有限**：CGAM 本质上是对 micro-batching 的简单应用——将大 batch 拆成两个 sub-batch 顺序执行。Batching cap selection 的公式虽然形式化了，但方法本身较为 straightforward（找 throughput gain 拐点）。MAWS 的"CPU-heavy 用 multi-processing、LLM-heavy 用 multi-threading"也是相对直观的 heuristic。作为体系结构/系统领域的贡献，优化深度不够——没有涉及 NUMA-aware scheduling、cache partitioning、memory bandwidth allocation 等更深层的 CPU microarchitecture 优化。

2. **实验方法论存在显著缺陷**：
   - **Closed-loop arrival model**：假设所有请求同时到达（t=0），这与实际 online serving 的 open-loop/Poisson arrival 模式差异很大。在真实场景下，请求到达是随机的，CGAM 的 micro-batching 策略的有效性需要在更现实的 arrival pattern 下验证。
   - **Single run**：论文承认仅报告 single run 结果，虽声称方差约 5%，但没有给出 error bar 或 confidence interval，对于系统研究来说不够严谨。
   - **能耗测量在不同硬件上进行**：Profiling 用 Emerald Rapids + B200，能耗用 Threadripper + H200，虽然论文声称"relative trends 架构上一致"，但这一假设未经验证，削弱了 cross-metric 分析的可信度。

3. **Workload 选取的局限性**：
   - ChemCrow 使用 OpenAI API（GPT-4-0613），其 inference 延迟包含网络 RTT 和 API 服务端排队，无法反映真实的 GPU inference 特征，与其他使用本地 vLLM 的 workload 不可直接对比。
   - 五个 workload 中缺少 multi-agent 协作场景（如 MetaGPT、AutoGen），而这是 agentic AI 的重要发展方向，可能引入更复杂的 CPU-GPU 交互模式。
   - 所有 workload 均为 single-agent，未涉及 multi-agent communication overhead。

4. **缺乏对 real-world deployment 的考虑**：
   - 论文未讨论 multi-tenant 场景下多个用户的 agentic workload 共享 CPU/GPU 资源时的干扰效应。
   - 未考虑 network I/O 延迟（web search、API call 的实际网络延迟在 profiling 中如何处理不够明确）。
   - MAWS 需要预先知道请求是 CPU-heavy 还是 LLM-heavy，论文未讨论如何在 online 场景中进行此分类。

5. **硬件配置单一**：论文仅在一种 CPU + GPU 配置上进行实验（Intel Emerald Rapids + NVIDIA B200），未覆盖 ARM 服务器、AMD EPYC、不同 GPU（A100、H100）等，限制了结论的普适性。论文在 Conclusion 中承认这是 future work。

### 疑问或值得追问的点

- **$B_{cap}$ 的选择是否应该是动态的？** 论文假设 $B_{cap}$ 是静态的（实验中固定为 64），但实际运行中 CPU 和 GPU 的负载会动态变化（如 CPU 上其他进程的干扰），动态调整 $B_{cap}$ 可能带来更大收益。
- **Tool processing 的可加速性**：论文将 tool processing 视为 CPU-bound 的既定事实，但部分 tool（如 dense retrieval、summarization）是否可以迁移到 GPU 或 NPU 上？如果可以，CPU bottleneck 论点的长期有效性如何？
- **CGAM 与 continuous batching 的交互**：论文声称与 vLLM 的 continuous batching 正交使用，但 CGAM 的 sequential micro-batching 可能影响 vLLM 的 scheduling 效率（如 prefill-decode 调度），需要更深入分析。
- **Haystack 的 305GB ENNS retrieval 是否具有代表性？** 这是一个极端 case（内存远超 GPU memory），论文选择它来展示 CPU bottleneck 的上界，但多数 production RAG 系统会使用 ANNS（如 HNSW）而非 ENNS，延迟分布会有显著不同。
- **能耗分析中"扣除 idle power"的方法是否准确？** 在 multi-processing 高负载下，CPU 的 idle power 本身可能发生变化（如 C-state 转换），简单减去固定 idle power 可能引入误差。
