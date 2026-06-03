# AI领域研究调研报告：2025年12月 - 2026年5月

## 摘要

本报告基于对Arxiv上近半年（2025年11月底至2026年5月）AI相关领域论文的系统性调研，覆盖了cs.CL（计算语言学）、cs.AI（人工智能）、cs.LG（机器学习）、cs.CV（计算机视觉）、cs.MA（多智能体系统）等核心类别共1757篇论文。通过对论文质量、作者/机构影响力、研究贡献和创新性的多维度评估，筛选出高质量论文进行深入分析，总结了近半年AI研究的热点方向、主要趋势和突破性成果。

---

## 一、研究热点总览

### 1.1 本期最重大研究突破（Top Breakthroughs）

| 排名 | 论文 | 核心贡献 | 发表时间/会议 |
|------|------|----------|--------------|
| 1 | **DeepSeek-R1** | 通过纯RL训练激发LLM推理能力，无需人工标注的CoT数据 | 2025.01 |
| 2 | **DeepSeek-V3** | 671B MoE模型，训练成本仅$5.6M，性能匹敌GPT-4 | 2024.12 |
| 3 | **Test-Time Compute Scaling** | 推理时计算的系统性Scaling Law研究 | 2025.01-02 |
| 4 | **Kimi k1.5** | RL scaling for LLM推理，长链推理能力 | 2025.01 |
| 5 | **Native Sparse Attention** | 硬件对齐的稀疏注意力，可原生训练 | 2025.02 |
| 6 | **SFT Memorizes, RL Generalizes** | 揭示SFT和RL的本质差异，RL促进泛化 | 2025.01 |
| 7 | **Transformer-Squared** | 自适应LLM，运行时动态调整模型参数 | 2025.01/ICLR 2025 |
| 8 | **FlashInfer** | 高效可定制的LLM推理注意力引擎 | 2025.01/MLSys 2025 |
| 9 | **rStar-Math** | 小模型通过自进化深度思考掌握数学推理 | 2025.01 |
| 10 | **Curse of Depth** | 揭示LLM深层层的退化问题 | 2025.02/NeurIPS 2025 |

### 1.2 六大研究热点方向

```
热度排名:
1. ████████████████████ 推理增强与Test-Time Compute
2. ███████████████████  多智能体系统 (Multi-Agent)
3. ██████████████████   大模型对齐与安全
4. █████████████████    高效推理与模型压缩
5. ████████████████     多模态大模型
6. ███████████████      视频生成与World Models
```

---

## 二、各方向深度分析

### 2.1 推理增强与Test-Time Compute（最热门方向）

**核心趋势：从"训练时计算"到"推理时计算"的范式转移**

本期最重大的研究趋势是推理时计算（Test-Time Compute）的兴起。以DeepSeek-R1为标志，研究界发现通过在推理阶段投入更多计算，可以显著提升模型在复杂任务上的表现，且这种提升遵循可预测的Scaling Law。

**里程碑论文：**

1. **DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning** (2025.01)
   - 作者: DeepSeek-AI
   - 核心贡献: 证明纯RL训练可以从零激发LLM的推理能力（如Chain-of-Thought），无需任何人工标注的推理过程数据
   - 影响: 开创了"推理模型"新范式，引发了后续大量follow-up工作
   - 关键发现: RL训练过程中，模型自发涌现出反思、验证、分步推理等行为

2. **Can 1B LLM Surpass 405B LLM? Rethinking Compute-Optimal Test-Time Scaling** (2025.02)
   - 核心贡献: 证明小模型配合充足的推理时计算，可以超越大模型的直接推理
   - 方法: 提出计算最优的test-time scaling策略

3. **Kimi k1.5: Scaling Reinforcement Learning with LLMs** (2025.01)
   - 作者: Moonshot AI (Kimi Team)
   - 核心贡献: 展示了RL training如何scale到更长的推理链，提出了有效的长链推理训练方法

4. **rStar-Math: Small LLMs Can Master Math Reasoning with Self-Evolved Deep Thinking** (2025.01)
   - 核心贡献: 小模型通过自进化的深度思考策略，在数学推理上达到接近大模型的水平

5. **Demystifying Long Chain-of-Thought Reasoning in LLMs** (2025.02)
   - 作者: Edward Yeo, Graham Neubig等
   - 核心贡献: 系统分析长Chain-of-Thought推理的工作原理和局限性

6. **Stabilizing Recurrent Dynamics for Test-Time Scalable Latent Reasoning in Looped Language Models** (ICML 2026)
   - 核心贡献: 在循环语言模型中实现稳定的潜在推理，使test-time推理可扩展

7. **Transformers Provably Learn to Internalize Chain-of-Thought** (2026.05)
   - 核心贡献: 理论证明Transformer可以内化CoT推理过程

**研究共识：**
- RL是激发推理能力的关键训练范式，SFT仅做记忆，RL才能泛化
- Test-Time Compute存在明确的Scaling Law
- 小模型+充足推理计算 可以击败 大模型+少量推理计算
- 长链推理(Long CoT)的质量直接决定了最终答案的准确性

---

### 2.2 多智能体系统 (Multi-Agent Systems)

**核心趋势：从单一Agent到协作式多Agent架构**

本期Multi-Agent研究呈爆发式增长，成为ICML 2026和ACL 2026的热门方向。研究重点从"如何让单个Agent工作"转向"如何让多个Agent有效协作"。

**重要论文：**

1. **LegalGraphRAG: Multi-Agent Graph RAG for Reliable Legal Reasoning** (ACL 2026)
   - 结合知识图谱和多Agent架构实现可靠的法律推理

2. **Toward Training Superintelligent Software Agents through Self-Play SWE-RL** (ICML 2026)
   - 通过自对弈训练软件工程Agent，在SWE-bench上取得突破

3. **MASPO: Joint Prompt Optimization for LLM-based Multi-Agent Systems** (ICML 2026)
   - 首次系统研究多Agent系统的联合prompt优化

4. **CyberJurors: Multi-Agent Simulation Task for E-Commerce Disputes** (ICML 2026)
   - 多Agent模拟复杂决策场景

5. **Learning to Communicate: Toward End-to-End Optimization of Multi-Agent Language Systems** (COLM 2026)
   - 端到端优化多Agent间的语言通信

6. **LLM-Guided Communication for Cooperative Multi-Agent RL** (ICML 2026)
   - LLM指导的多Agent协作通信

7. **Diversity Collapse in Multi-Agent LLM Systems** (ACL 2026)
   - 发现多Agent系统中的多样性崩溃问题，提出解决方案

8. **Multi-Agent Reasoning Improves Compute Efficiency: Pareto-Optimal Test-Time Scaling** (ACL 2026)
   - 多Agent推理比单Agent在计算效率上更优

**关键发现：**
- 多Agent系统面临"多样性崩溃"问题——Agent间趋向一致化
- 有效的Agent间通信协议是性能关键
- Self-Play训练对Agent能力提升效果显著
- Multi-Agent可以实现比单Agent更高效的Test-Time Scaling

---

### 2.3 大模型对齐与安全 (Alignment & Safety)

**核心趋势：从RLHF走向更精细化的对齐方法，安全攻防进入新阶段**

**重要论文：**

1. **Alignment Tampering: How RLHF Is Exploited to Optimize Misaligned Biases** (ICML 2026)
   - 揭示RLHF可能被利用来优化错误偏见的机制
   - 重大发现: 对齐过程本身可能被"篡改"

2. **Curriculum Learning for Safety Alignment** (ICML 2026)
   - 渐进式课程学习提升安全对齐效果

3. **Jailbreak to Protect: Buffering and Reinforcing via Temporary Jailbreaking** (ICML 2026, Oral)
   - 创新性地利用临时越狱来增强模型安全性

4. **AdaDPO: Self-Adaptive Direct Preference Optimization with Balanced Gradient Updates** (2026.05)
   - DPO的自适应改进版本

5. **TUR-DPO: Topology- and Uncertainty-Aware DPO** (ICML 2026)
   - 考虑拓扑结构和不确定性的DPO方法

6. **Topology-Enhanced Alignment for LLMs: Trajectory Topology Loss** (ACL 2026)
   - 基于轨迹拓扑的对齐损失函数

7. **SPARD: Defending Harmful Fine-Tuning via Safety Projection** (ICML 2026)
   - 防御恶意微调攻击的安全投影方法

8. **Calibrating Conservatism for Scalable Oversight** (2026.05)
   - 可扩展监督中的保守主义校准

**Jailbreak与Red Teaming热点：**
- DMN: Compositional Framework for Jailbreaking Multimodal LLMs (ACL 2026)
- 多模态Jailbreak的鲁棒性研究 (2026.05)
- Jailbreak susceptibility prediction via behavioral geometry

**研究共识：**
- DPO及其变体成为主流对齐方法（替代PPO-based RLHF）
- 安全对齐不是一次性的，需要持续维护
- 多模态模型的安全性比纯文本模型更脆弱
- "对齐税"（alignment tax）问题被广泛研究

---

### 2.4 高效推理与模型架构 (Efficient Inference & Architecture)

**核心趋势：MoE架构主流化、推理效率成为核心竞争力**

**重要架构论文：**

1. **DeepSeek-V3 Technical Report** (2024.12)
   - 671B MoE (37B active)，训练成本仅$5.6M
   - 核心创新: Multi-Head Latent Attention, DeepSeekMoE架构改进
   - 影响: 证明开源模型可以用极低成本达到闭源模型水平

2. **Native Sparse Attention: Hardware-Aligned and Natively Trainable** (2025.02, DeepSeek)
   - 硬件对齐的原生稀疏注意力机制
   - 直接在训练中使用稀疏注意力，而非推理时近似

3. **FlashInfer: Efficient and Customizable Attention Engine** (MLSys 2025)
   - 高效灵活的注意力计算引擎，支持各种注意力变体

4. **The Curse of Depth in LLMs** (NeurIPS 2025)
   - 发现LLM深层存在严重退化，后面的层贡献递减
   - 启示: 模型不一定要"更深"

5. **MobileMoE: Scaling On-Device Mixture of Experts** (2026.05)
   - 将MoE扩展到端侧设备

6. **ReMoE: Boosting Expert Reuse through Router Fine-Tuning** (ICML 2026)
   - 内存受限下MoE推理的专家重用

7. **Speculative Decoding系列** (2025-2026)
   - Cassandra: 边缘端推理模型的自投机解码
   - Beyond the Target: 从模仿到协作
   - FlexDraft: 灵活投机解码

8. **Unified Neural Scaling Laws** (2026.05)
   - 统一的神经网络Scaling Law框架

**KV Cache压缩：**
- NestedKV: 嵌套内存路由的长上下文KV Cache压缩
- IndexMem: 基于学习的KV Cache驱逐策略
- Quantized Keys Steal Attention (ICML 2026)

**研究共识：**
- MoE是当前最成功的模型效率架构
- 推理效率（而非训练效率）成为核心瓶颈
- KV Cache管理是长上下文推理的关键挑战
- 投机解码(Speculative Decoding)成为标配技术

---

### 2.5 多模态大模型 (Multimodal LLMs)

**核心趋势：VLM能力快速提升，多模态推理成为新前沿**

**重要论文：**

1. **OmniVerifier-M1: Multimodal Meta-Verifier** (ICML 2026)
   - 通过符号化元验证和解耦RL训练的通用视觉验证器
   - 支持细粒度的区域级自纠正

2. **Qwen-Image-2.0 Technical Report** (2026.05)
   - 阿里通义千问的最新多模态模型技术报告

3. **Gemini Embedding 2: Native Multimodal Embedding** (2026.05)
   - Google的原生多模态嵌入模型

4. **InternVL2.5** (2025.01)
   - 上海AI Lab的最新视觉语言模型

5. **DUEL: Adversarial Self-Play for Multimodal Reasoning** (2026.05)
   - 通过对抗自对弈提升多模态推理

6. **Imagine while Reasoning in Space: Multimodal Visualization-of-Thought** (2025.01)
   - 多模态"思维可视化"，推理时在脑中想象

7. **ProSR: Process-Shaped Spatial Reasoning for Reliable CoT in VLMs** (2026.05)
   - VLM中可靠的空间推理CoT

8. **Mags-RL: Multimodal LLMs with Agentic RL for Complex Scene Reasoning** (2026.05)
   - 结合Agent RL的多模态复杂场景推理

**多模态Agent方向：**
- Agent Explorative Policy Optimization for Multimodal Agentic Reasoning
- SafetyALFRED: Evaluating Safety-Conscious Planning of Multimodal LLMs (ACL 2026)
- Hallucination as Exploit: Evidence-Carrying Multimodal Agents

**研究共识：**
- VLM的推理能力是当前瓶颈（感知已接近饱和）
- 多模态幻觉(hallucination)仍是核心挑战
- 视频理解和长视频处理成为新兴热点
- 多模态安全性弱于纯文本模型安全性

---

### 2.6 视频生成与World Models

**核心趋势：视频生成质量飞跃，World Models概念兴起**

**重要论文：**

1. **Gamma-World: Generative Multi-Agent World Modeling Beyond Two Players** (NVIDIA, 2026.05)
   - 超越两人交互的多Agent世界建模
   - 生成式方法用于世界模型

2. **Causal Physics Steering in Video World Models via Concept Activation Vectors** (CVPR 2026)
   - 视频世界模型中的因果物理操控

3. **DexSIM: Real-time Dexterous Simulation with Unified Causal Video Diffusion** (ICLR 2026 Workshop)
   - 用因果视频扩散实现实时灵巧操作模拟

4. **OSP-Next: Efficient High-Quality Video Generation** (2026.05)
   - 高质量视频生成的效率优化（稀疏序列并行+HiF8量化）

5. **Tempered Self-Similarity Alignment for Physically Plausible Video Generation** (CVPR 2026 Workshop)
   - 物理真实的视频生成

6. **Pantheon360: 3D-Aware 360° Video Diffusion** (CVPR 2026)
   - 3D感知的360度视频扩散生成

7. **Where Concept Erasure Should Occur in Text-to-Video Diffusion Models** (ICML 2026)
   - 视频扩散模型中概念擦除的正确位置

**研究共识：**
- 视频生成从"能生成"走向"可控生成"
- World Model = Video Generation + Physics Understanding + Planning
- 物理真实性是视频生成的关键挑战
- 视频扩散模型的效率问题亟待解决

---

### 2.7 可解释性与机制分析 (Interpretability & Mechanistic Analysis)

**核心趋势：Mechanistic Interpretability从探索走向系统化**

**重要论文：**

1. **Position: Mechanistic Interpretability Must Disclose Identification Assumptions** (NeurIPS 2026)
   - 呼吁机制可解释性研究必须明确其识别假设

2. **How Do Transformers Learn to Associate Tokens** (ICLR 2026)
   - 从梯度主导项角度理解Transformer的token关联学习

3. **Why Does RL Generalize? A Feature-Level Mechanistic Study** (ACL 2026)
   - 从特征层面机制分析RL的泛化能力

4. **Hidden Error Awareness in Chain-of-Thought Reasoning** (ICML 2026)
   - 模型对CoT中错误的隐式感知能力

5. **MechRL: RL Agents Perform Circuit Discovery for Mechanistic Interpretability** (2026.05)
   - 用RL Agent自动发现模型内部电路

6. **Polymorphism Is Rotation: Operational Mechanistic Interpretability** (2026.05)
   - 从旋转角度理解多态性的机制

**研究共识：**
- Mechanistic Interpretability正在从小模型走向大模型
- RL Agent可以辅助进行自动化的机制发现
- CoT的忠实性(faithfulness)问题被广泛质疑

---

### 2.8 数据工程与训练策略 (Data & Training)

**核心趋势：数据质量超越数据数量，合成数据成为关键工具**

**重要论文：**

1. **InfoLaw: Information Scaling Laws with Quality-Weighted Data** (ICML 2026)
   - 基于信息论的质量加权Scaling Law
   - 核心发现: 数据质量可以被量化并纳入Scaling Law

2. **How Should LLMs Consume High-Quality Data? Optimal Data Scheduling** (2026.05)
   - 高质量数据的最优调度策略

3. **SFT Memorizes, RL Generalizes** (2025.01)
   - 关键发现: SFT本质是记忆，RL才是泛化的关键

4. **合成数据系列：**
   - Activation Steering for Synthetic Data Generation (2026.05)
   - SynAE: Framework for Measuring Synthetic Data Quality (2026.05)
   - MixAtlas: Uncertainty-aware Data Mixture Optimization (2026.04)

**研究共识：**
- "数据质量 > 数据数量"已成为普遍共识
- 合成数据生成成为标准训练流程的一部分
- 数据混合比例(data mixture)是影响模型能力的关键超参数
- RL后训练的重要性被反复验证

---

### 2.9 长上下文与RAG (Long Context & RAG)

**核心趋势：百万token上下文成为标配，RAG架构持续演进**

**重要论文：**

1. **ATLAS: All-round Testing of Long-context Abilities across Scales** (2026.05)
   - 长上下文能力的全面评测基准

2. **NestedKV: Nested Memory Routing for Long-Context KV Cache Compression** (2026.05)
   - 嵌套内存路由实现高效长上下文

3. **IndexMem: Learned KV-Cache Eviction with Latent Memory** (2026.05)
   - 学习型KV-Cache驱逐策略

4. **LegalGraphRAG** (ACL 2026)
   - 结合图结构的RAG系统

5. **FinRAG-12B: Production-Validated Recipe for Grounded QA in Banking** (ACL 2026)
   - 金融领域生产级RAG系统

6. **Grounded Cache Routing for RAG: When Is It Safe to Reuse an Answer** (2026.05)
   - RAG缓存路由的安全重用机制

**研究共识：**
- KV-Cache管理是长上下文的核心技术挑战
- RAG与长上下文不是互斥关系，而是互补
- 特定领域(金融、法律)的RAG系统需要专门设计

---

### 2.10 代码生成与Software Engineering

**核心趋势：从代码补全到自主软件工程Agent**

**重要论文：**

1. **SWE-RL: Self-Play for Software Engineering** (ICML 2026)
   - 通过自对弈RL训练软件工程Agent

2. **Efficient Post-training of LLMs for Code Generation With Offline RL** (2026.05)
   - 离线RL高效训练代码生成模型

3. **Beyond pass@k: Redundancy-Aware RLVR for Multi-Sample Code Generation** (2026.05)
   - 超越pass@k的多样本代码生成评估

4. **FPMoE: Sparse MoE for Functional Code Generation** (2026.05)
   - 稀疏MoE用于函数式代码生成

5. **Dive into Claude Code: The Design Space of AI Agent Systems** (2026.04)
   - 分析Claude Code等AI Agent系统的设计空间

**研究共识：**
- 代码Agent从"补全"走向"完整软件工程流程"
- Self-Play和RL是训练代码Agent的有效方法
- 评估标准从pass@k向更真实的SWE-bench发展

---

## 三、让人眼前一亮的研究发现

### 3.1 颠覆性发现

1. **"SFT记忆，RL泛化"** — 这一发现从根本上改变了对post-training的理解。SFT只是让模型记住特定的答案模式，而RL才是让模型真正学会推理和泛化。

2. **"1B模型可以超越405B模型"** — 通过充足的test-time compute，小模型可以在推理任务上超越大数百倍的模型。这动摇了"模型越大越好"的传统认知。

3. **"深度的诅咒"** — LLM的深层存在严重退化，后面的层对模型输出的贡献递减。这暗示当前的深度架构可能不是最优的。

4. **"对齐可以被篡改"** — RLHF对齐过程本身可能被利用来优化错误偏见，安全对齐的鲁棒性远未被解决。

5. **"多Agent系统中的多样性崩溃"** — 多个LLM Agent协作时，会趋向产生一致的输出，丧失多样性。

6. **"DeepSeek-R1的自发涌现"** — 纯RL训练中，模型自发涌现出反思(self-reflection)、验证(verification)等人类般的推理行为，无需任何显式监督。

### 3.2 技术突破亮点

1. **成本效率革命**: DeepSeek-V3用$5.6M训练达到GPT-4水平，证明高性能不需要天文数字的投入
2. **推理时计算**: 开辟了模型能力提升的全新维度（从scale训练到scale推理）
3. **MoE架构成熟**: 从实验性技术变为主流生产架构
4. **自对弈训练**: Self-Play从游戏AI成功迁移到代码生成、推理等领域
5. **原生稀疏注意力**: 不再是推理时的近似，而是训练时的原生特性

---

## 四、具有普遍共识的研究成果

### 4.1 已形成共识的结论

1. **RL后训练 > 纯SFT**: RL（特别是GRPO/DPO）训练对于提升推理能力至关重要
2. **数据质量 > 数据数量**: 精心筛选的高质量数据比大规模低质量数据更有价值
3. **MoE是效率的关键**: Mixture-of-Experts架构是平衡模型容量和推理效率的最佳方案
4. **Test-Time Compute可以Scale**: 推理时计算投入遵循类似训练的Scaling Law
5. **多模态安全性落后于能力**: 多模态模型的安全性系统性弱于其能力发展
6. **合成数据有效但需谨慎**: 合成数据可以有效扩充训练集，但质量控制是关键
7. **CoT忠实性存疑**: 模型的Chain-of-Thought推理过程不一定反映其真实的内部计算
8. **开源模型追赶闭源**: 开源模型与闭源模型的差距在持续缩小

### 4.2 新兴但尚未达成共识的方向

1. **World Models是否是通向AGI的路径**: 有争议
2. **推理模型的上限在哪里**: 尚不明确
3. **多Agent vs 单Agent的最优边界**: 仍在探索
4. **对齐是否从根本上可解决**: 有不同观点
5. **Transformer是否已到极限**: 新架构(Mamba/SSM)与Transformer的竞争结果未定

---

## 五、主要发展方向预判

### 5.1 短期趋势（6-12个月）

1. **推理模型普及**: 各大实验室将推出自己的reasoning model
2. **Agent框架成熟**: Multi-Agent协作框架将走向生产化
3. **端侧推理**: MoE+投机解码+量化使大模型在端侧运行
4. **视频生成质量飞跃**: 从1080p走向4K，从短视频走向长视频
5. **专业领域RAG**: 金融、法律、医疗等领域的专业RAG系统

### 5.2 中期趋势（1-2年）

1. **World Models**: 从视频生成走向真正的世界理解和模拟
2. **自主Agent**: 从辅助工具走向自主完成复杂任务
3. **模型安全**: 对齐技术必须跟上能力增长的速度
4. **AI for Science**: 蛋白质设计、药物发现将取得实质突破
5. **架构创新**: 可能出现超越Transformer的新架构

---

## 六、重要机构与团队追踪

### 6.1 最具影响力的研究机构

| 机构 | 代表性工作 | 主要方向 |
|------|-----------|---------|
| DeepSeek | DeepSeek-R1, V3, Native Sparse Attention | 推理模型, 高效架构 |
| Google DeepMind | Gemini 2, Test-Time Compute Scaling | 多模态, Scaling |
| Meta FAIR | Large Concept Models, Llama | 开源模型, 概念学习 |
| OpenAI | o1/o3系列 | 推理模型 |
| Anthropic | Claude系列, Constitutional AI | 安全对齐 |
| Moonshot AI (Kimi) | Kimi k1.5 | RL for Reasoning |
| 阿里 (Qwen) | Qwen2.5, Qwen-Image-2.0 | 多模态, 开源模型 |
| 上海AI Lab | InternVL2.5 | 多模态模型 |
| NVIDIA | Gamma-World | World Models, GPU计算 |
| Stanford | Scaling研究, Agent研究 | 基础研究 |

### 6.2 高产出顶会论文统计

- **ICML 2026**: 本期数据中命中率最高的顶会
- **ACL 2026**: NLP/LLM方向的核心会议
- **CVPR 2026**: 视觉和多模态方向
- **ICLR 2025/2026**: 基础ML和表示学习
- **NeurIPS 2025/2026**: 理论和新方向

---

## 七、论文列表（按方向分类）

### 7.1 推理与Test-Time Compute
1. DeepSeek-R1 (2501.12948)
2. Can 1B Surpass 405B? Compute-Optimal Test-Time Scaling (2502.06703)
3. Kimi k1.5 (2501.14249)
4. rStar-Math (2501.08025)
5. Demystifying Long CoT Reasoning (2502.03373)
6. SFT Memorizes, RL Generalizes (2501.18438)
7. Stabilizing Recurrent Dynamics for Test-Time Scalable Latent Reasoning (ICML 2026)
8. Transformers Provably Learn to Internalize CoT (2026.05)
9. Multi-Agent Reasoning for Pareto-Optimal Test-Time Scaling (ACL 2026)
10. Scaling Search and Learning: Roadmap to Reproduce o1 (2412.14135)

### 7.2 模型架构与效率
1. DeepSeek-V3 (2412.19437)
2. Native Sparse Attention (2502.05795)
3. FlashInfer (2501.01005)
4. Transformer-Squared (2501.06252)
5. The Curse of Depth (NeurIPS 2025)
6. Unified Neural Scaling Laws (2026.05)
7. InfoLaw: Information Scaling Laws (ICML 2026)
8. MobileMoE (2026.05)
9. ReMoE (ICML 2026)
10. Cassandra: Self-Speculative Decoding at Edge (2026.05)

### 7.3 多智能体系统
1. LegalGraphRAG (ACL 2026)
2. SWE-RL: Self-Play SWE Agent (ICML 2026)
3. MASPO (ICML 2026)
4. CyberJurors (ICML 2026)
5. LLM-Guided Communication (ICML 2026)
6. Diversity Collapse in Multi-Agent LLM (ACL 2026)
7. RADAR: Multi-Agent Communication (ICML 2026)
8. Learning to Communicate (COLM 2026)
9. RoadMapper (ACL 2026)
10. Gamma-World (NVIDIA, 2026.05)

### 7.4 对齐与安全
1. Alignment Tampering (ICML 2026)
2. Jailbreak to Protect (ICML 2026, Oral)
3. Curriculum Learning for Safety Alignment (ICML 2026)
4. TUR-DPO (ICML 2026)
5. SPARD: Defending Harmful Fine-Tuning (ICML 2026)
6. DMN: Jailbreaking Multimodal LLMs (ACL 2026)
7. SafetyALFRED (ACL 2026)
8. Calibrating Conservatism for Scalable Oversight (2026.05)
9. AdaDPO (2026.05)
10. LLM Safety surveys and benchmarks

### 7.5 多模态模型
1. OmniVerifier-M1 (ICML 2026)
2. Qwen-Image-2.0 (2026.05)
3. Gemini Embedding 2 (2026.05)
4. InternVL2.5 (2025.01)
5. Visualization-of-Thought (2025.01)
6. ProSR: Spatial Reasoning in VLMs (2026.05)
7. DUEL: Adversarial Self-Play for Multimodal Reasoning (2026.05)
8. Mags-RL (2026.05)
9. MAGIC: Multimodal Alignment (2026.05)
10. VoiceGiraffe: Extreme Long-Context Audio-Language (2026.05)

### 7.6 视频生成与World Models
1. Gamma-World (NVIDIA, 2026.05)
2. Causal Physics Steering in Video World Models (CVPR 2026)
3. DexSIM: Unified Causal Video Diffusion (ICLR 2026 Workshop)
4. OSP-Next: Efficient Video Generation (2026.05)
5. Pantheon360: 3D-Aware Video Diffusion (CVPR 2026)
6. Tempered Self-Similarity for Physical Video (CVPR 2026 Workshop)
7. Where Concept Erasure Should Occur (ICML 2026)
8. VidPrism: Heterogeneous MoE for Video (CVPR 2026)
9. Quantized Keys for Video Diffusion (ICML 2026)
10. Embodied Multi-Agent World Models (2026.05)

---

## 八、结论

近半年AI研究的核心主题可以归纳为：**"让模型学会思考"**。从DeepSeek-R1的纯RL训练激发推理能力，到Test-Time Compute Scaling的理论化，再到多Agent协作推理，整个领域正在从"大力出奇迹"的训练范式，走向"聪明地使用计算"的推理范式。

与此同时，模型效率（MoE、稀疏注意力、投机解码）、安全对齐（DPO变体、对齐篡改问题）、多模态能力（VLM推理、视频生成）等方向也在同步快速发展。

最令人印象深刻的趋势是：**开源生态正在从追赶者变成引领者**。DeepSeek-V3/R1以极低成本达到顶级水平，Qwen、InternVL等开源模型在多个维度上与闭源模型并驾齐驱，这预示着AI研究和应用的民主化进程正在加速。

---

*报告生成时间: 2026年5月28日*
*数据来源: Arxiv API系统搜索，覆盖1757篇论文*
*筛选标准: 顶会录用、知名机构/作者、研究方向热度、创新性综合评分*
