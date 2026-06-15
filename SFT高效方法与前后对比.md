# SFT 中的「省算力」与「前后对比」:从 loss 信号到 MeZO / LoRA / GaLore

> 主题:在监督微调(SFT)里,哪些「提前知道的信息」能简化 forward/backward?如何高效比较 SFT 前后的 logits / hidden states / 参数?以及 MeZO、LoRA、GaLore 三类方法的原理、数学与取舍。

---

## 0. 一句话地图

训练一步的本质是:**算出梯度方向 → 沿方向更新参数**。能否「简化」取决于你掌握了什么信息:

| 你提前知道的东西 | 能否简化更新 | 对应方法 / 工具 |
|---|---|---|
| loss 的**标量数值** | ❌ 没用,标量不含方向信息 | —— |
| **梯度本身**(或其低秩/稀疏结构) | ✅ 限定更新子空间 | LoRA、GaLore |
| loss 对**扰动的变化** | ✅ 用 forward 差分估计梯度 | MeZO(零阶) |
| 某样本 loss 已经很低(可跳过) | ✅ 减少要 backward 的样本数 | selective backprop / OHEM |
| 已有 θ_before 和 θ_after | ✅ 直接相减,免再训练 | task vector / checkpoint diff |
| 单步小更新下的输出变化 | ✅ 一阶泰勒,一次 JVP | forward-mode AD / NTK |

---

## 1. 为什么「知道 loss 数值」救不了你

反向传播要的不是 loss 标量 $L$,而是 $L$ 对每个参数的**梯度** $\partial L/\partial\theta$ —— 一个和参数同维度(几十亿维)的向量,它编码「往哪个方向走、走多少」。

类比:

- **海拔**(loss 值)= 现在多高;
- **坡度方向**(gradient)= 往哪边是下坡。

知道海拔 1000 米,推不出脚下哪边下坡。所以即便 forward 前神谕般知道 $L=0.37$,你仍要:

1. 做完整 forward —— backward 的链式法则在**每一层都需要该层 forward 时的中间激活值**(这也是训练显存大头);
2. 做完整 backward —— 把梯度从输出链式传回每层。

**结论:loss 的数值省不掉任何一步。有用的永远是「方向 / 变化」,不是「数值」。**

---

## 2. 训练显存的四大件(理解后面所有方法的基准)

| 组件 | 含义 | 量级(Adam, 以参数量 N 计) |
|---|---|---|
| 模型权重 W | 参数本身 | N(fp16) |
| 梯度 G | $\partial L/\partial\theta$ | N |
| 优化器状态 | Adam 的一阶/二阶矩 m, v | **2N**(常是最大头) |
| 激活值 | forward 缓存,供 backward 用 | 随 batch / 序列长度增长 |

后面三种方法,本质都是在**砍掉其中某几件**:

- **LoRA** → 砍梯度 + 优化器状态(只对极少量 adapter 参数保留)
- **GaLore** → 砍优化器状态(梯度压到低秩子空间)
- **MeZO** → 砍梯度 + 优化器状态 + **激活值**(只前向,最狠)

---

## 3. MeZO:零阶优化,真正「省掉 backward」

**Memory-efficient Zeroth-Order optimizer**(Malladi et al., 2023)。这是唯一真正**不做反向传播**的方法 —— 它用「loss 对扰动的变化」来估计梯度。

### 3.1 核心:SPSA 梯度估计

用同时扰动随机近似(Simultaneous Perturbation Stochastic Approximation),只靠**两次 forward** 估计梯度:

$$
\hat g = \frac{L(\theta + \epsilon z) - L(\theta - \epsilon z)}{2\epsilon}\, z,
\qquad z \sim \mathcal N(0, I)
$$

更新照常:$\theta \leftarrow \theta - \eta\,\hat g$。

直觉:$z$ 是随机方向,中括号里是沿 $z$ 的**有限差分方向导数**(一个标量),它告诉你「这个随机方向是上坡还是下坡、多陡」,再把这个标量乘回 $z$,得到一个对真实梯度的无偏(在 $\epsilon\to0$ 极限下)估计。

> 注意:这里用的依然是 loss 的**差分**,不是单个 loss 值 —— 再次印证第 1 节的结论。

### 3.2 为什么省内存:O(1) 额外显存

关键工程技巧:**不存 $z$,只存随机种子**,需要时用同一种子重新生成 $z$。配合「原地扰动」:

```
1) 用 seed 生成 z,θ ← θ + εz,  forward 得 L+
2) 用同一 seed 再生成 z,θ ← θ - 2εz,forward 得 L-
3) 投影标量 c = (L+ - L-) / (2ε)
4) 用同一 seed 再生成 z,θ ← θ + εz(复原),并 θ ← θ - η·c·z 完成更新
```

整个过程**没有激活缓存、没有梯度张量、没有优化器状态**,额外显存 ≈ 0。微调所需显存 ≈ **推理显存**。论文报告可在单卡上微调 OPT-13B,省内存约 12×。

### 3.3 为什么对 LLM 微调居然能 work

零阶估计的方差理论上随参数维度 $d$ 线性增长 —— 几十亿维理应灾难。但 MeZO 的洞见是:**预训练好的 LLM 在下游任务上的 loss landscape 有很低的「有效秩 / 本征维度」**,真实需要探索的方向远少于 $d$,于是方差被压住,能收敛。

### 3.4 代价与适用场景

- **代价**:迭代步数远多于一阶(收敛慢、噪声大),wall-clock 不一定省 —— 省的是**显存**,不是**步数**。
- **额外能力**:目标函数**不需要可导**,可以直接优化 accuracy、F1、甚至黑盒奖励(因为只要能 forward 出一个标量)。
- **用它当**:显存极度受限、或目标不可微的场景。**不要用它当**通用提速手段。

---

## 4. LoRA:限定「权重更新」为低秩

**Low-Rank Adaptation**(Hu et al., 2021)。不动预训练权重,只学一个低秩增量。

### 4.1 数学

冻结 $W_0 \in \mathbb R^{d\times k}$,学习增量:

$$
\Delta W = BA, \quad B\in\mathbb R^{d\times r},\ A\in\mathbb R^{r\times k},\ r \ll \min(d,k)
$$

前向:

$$
h = W_0 x + \frac{\alpha}{r} B A x
$$

初始化 $A\sim\mathcal N$、$B=0$,使得起点 $\Delta W = 0$(不破坏预训练)。**只有 $A,B$ 参与训练**。

### 4.2 省在哪 / 不省在哪

- ✅ **省梯度 + 优化器状态**:只对 $A,B$(占总量 <1%)保留梯度和 Adam 矩。Adam 状态是 2N 大头,这里直接砍到 ~0。
- ⚠️ **不省激活显存**:backward 仍要穿过整个网络(adapter 嵌在各层里,要算到它们的输入梯度),冻结层的激活照样要缓存。激活省显存得另配 gradient checkpointing。
- ✅ **推理零延迟**:可把 $W = W_0 + \frac{\alpha}{r}BA$ 合并回去,推理与原模型同构。

### 4.3 根本限制

权重更新被**强制限制在秩 $\le r$ 的子空间**。如果任务真正需要的改动是高秩的,LoRA 表达力不足,达不到全量微调的解。这正是 GaLore 想突破的点。

### 4.4 常见变体

- **QLoRA**:base 权重 4-bit 量化 + LoRA,进一步省权重显存,单卡微调 65B。
- **DoRA**:把权重分解为幅度 + 方向,方向上用 LoRA,更接近全量微调表现。
- **AdaLoRA**:按重要性动态分配各层的秩预算。
- **rsLoRA**:修正缩放因子,让大 $r$ 时训练更稳。

---

## 5. GaLore:全参数训练,但把「梯度」压成低秩

**Gradient Low-Rank Projection**(Zhao et al., 2024)。和 LoRA 思路正交:**权重是全秩、全参数都更新**,只把**梯度 / 优化器状态**压到低秩子空间。

### 5.1 核心洞见

训练过程中,**梯度矩阵 $G$ 本身会趋于低秩**。于是:把梯度投影到低秩子空间里做 Adam,再投影回来更新全秩权重。

### 5.2 数学

对某层梯度 $G \in \mathbb R^{m\times n}$:

1. 周期性地对 $G$ 做 SVD,取前 $r$ 个左奇异向量组成投影 $P \in \mathbb R^{m\times r}$;
2. 投影:$R = P^\top G \in \mathbb R^{r\times n}$(低秩);
3. **Adam 的一阶/二阶矩只在 $R$ 空间维护**(显存从 $2mn$ 降到 $2rn$);
4. 投影回去更新全秩权重:$\theta \leftarrow \theta - \eta\, P\,\text{Adam}(R)$。

### 5.3 子空间切换(为什么能达到全秩解)

每隔 $T$ 步重新做一次 SVD、换一组 $P$。**单步在低秩子空间内走,但跨步累积起来覆盖全秩** —— 这是它和 LoRA 的本质区别:LoRA 的解永远困在一个固定低秩子空间,GaLore 的累积更新可以是全秩的。

### 5.4 省在哪 / 不省在哪

- ✅ **省优化器状态**:Adam 矩压到低秩,这是主要收益。论文报告可在 24GB 消费级显卡上**预训练**(不只是微调)LLaMA-7B。
- ⚠️ **不省 backward 计算**:梯度 $G$ 仍要正常反向传播算出来(才能投影)。
- ⚠️ **不直接省激活显存**:全网络前向 + 反向照旧。
- ➕ **额外开销**:周期性 SVD 的计算成本(可用低成本近似缓解)。

---

## 6. 三方法对比总表

| 维度 | 全量 FT | LoRA | GaLore | MeZO |
|---|---|---|---|---|
| 是否做 backward | ✅ | ✅ | ✅ | ❌(只 forward) |
| 权重更新的秩 | 全秩 | **低秩**(受限) | 全秩(累积) | 全秩 |
| 训练参数量 | 全部 | 极少(adapter) | 全部 | 全部 |
| 省**梯度**显存 | — | ✅ | ❌(算了再压) | ✅(无梯度) |
| 省**优化器**状态 | — | ✅ | ✅ | ✅(几乎无) |
| 省**激活**显存 | — | ❌ | ❌ | ✅(无缓存) |
| 主要代价 | 显存最大 | 表达力受限于秩 | SVD 开销 | 步数多、噪声大、慢 |
| 目标需可微 | 是 | 是 | 是 | **否** |
| 典型定位 | 上限基线 | 通用高效微调 | 显存受限的全参训练/预训练 | 极端省显存 / 不可微目标 |

**记忆锚点**:

- **LoRA** 压的是「更新的秩」(权重低秩);
- **GaLore** 压的是「梯度/状态的秩」(权重仍全秩);
- **MeZO** 干脆不要梯度(forward-only,省到激活)。

---

## 7. 高效比较 SFT「前后差异」(logits / hidden / 参数)

常规四步:forward → backward 更新 → 跑前后两个模型 inference → 逐层相减。能不能简化?**取决于 Δθ 大小。**

设 $\theta' = \theta + \Delta\theta$,想要任意输出(logits、某层 hidden)的变化 $\Delta f = f(\theta') - f(\theta)$。一阶泰勒:

$$
\Delta f \approx J_f(\theta)\,\Delta\theta
$$

近似在 **Δθ 小时准、大时崩**。SFT 一步 = 小;整段 SFT = 大。

### 7.1 Regime A:单步 / 小更新 → 一次 JVP 省掉「第二个模型」

$J_f\Delta\theta$ 是 **Jacobian-vector product**,用**前向模式自动微分**一次前向即可算出,**无需构造和运行更新后的模型**。而且 JVP 把切向量逐层向前传播,**一次调用同时给出 logits 和每一层 hidden 的预测变化**(第 $\ell$ 层的 tangent = 该层 $\Delta h_\ell$ 的一阶预测)。

| 常规 | JVP |
|---|---|
| backward 拿梯度 g | backward 拿 g(Δθ = −η·g,省不掉) |
| 更新 → θ′ | —— |
| 跑两个模型(2 次完整 inference) | **1 次 JVP**(≈1 次前向) |
| 逐层相减 | 直接读每层 tangent |

```python
import torch
from torch.func import functional_call, jvp

def f(params):
    # 让 model 返回 (logits, *hidden_states)
    return functional_call(model, params, (input_ids,))

g = compute_grad(...)                      # 一次 backward 拿 ∂L/∂θ
delta = {k: -lr * g[k] for k in params}    # Δθ = -η g

out, tangent_out = jvp(f, (params,), (delta,))
# out         : θ 处的 logits / hiddens
# tangent_out : 预测的 Δlogits / Δhiddens(逐层一次拿全)
```

**省掉**:更新参数、加载第二个模型、第二次完整 inference、逐层 diff。**保留**:一次 backward + 一次前向模式 JVP。

#### 理论捷径:只看 logits 时 = 经验 NTK

在样本 $x$ 上走一步,样本 $x'$ 的 logits 变化:

$$
\Delta z(x') \approx -\eta\,\Theta(x', x)\,\frac{\partial L}{\partial z(x)},
\qquad \Theta(x',x) = J_z(x')\,J_z(x)^\top
$$

$\Theta$ 是经验 NTK(神经切核)。即 logits 的变化可直接用核预测,不必真更新再前向。

### 7.2 Regime B:整段 SFT(多步、大位移)→ 训练省不掉,但「比较」能简化

一阶近似失效,必须真训练。但:

1. **Δθ = checkpoint 相减,免费**。训练时 backward 早做完,手上已有 $\theta_{\text{before}}$、$\theta_{\text{after}}$,直接相减就是全部参数变化 —— **不必再 backward**。这个差就是 task-arithmetic 的 **task vector**(Ilharco et al.),可拿来分析甚至做模型加减/编辑。
2. **「哪几层动得最多」近乎零成本**:逐模块算 $\|\Delta\theta_{\text{layer}}\|$ 或相对范数 $\|\Delta\theta\|/\|\theta\|$,定位 SFT 主要改了哪些层(通常集中在少数层 + 靠后层)。不用前向。
3. **logits/hidden 对比仍需双模型跑**:大位移没有免费午餐。但可用 JVP 做**归因** —— 把真实 $\Delta f$ 分解为「一阶可解释部分 $J_f\Delta\theta$」+「非线性残差」,看 SFT 效果有多少是线性可预测的。

### 7.3 选择指南

- 单步 / 小学习率探针 → **JVP**(省一次 inference,逐层一次拿全)。
- 完整 SFT 前后对比 → 实训练 + **checkpoint diff(task vector)+ 逐层范数**;logits/hidden 双模型跑,可选 JVP 归因。
- 判断一阶还能不能信 → 比 $J_f\Delta\theta$ 与真实 $\Delta f$ 的相对误差,或看 $\|\Delta\theta\|$ 相对参数尺度。误差大 = 进入非线性区,只能实跑。

---

## 7+. Regime 1 深入:单步更新的三层成本阶梯

> 核心论点一句话:**一次梯度步对任何输出的影响,是该输出沿「负梯度方向」的方向导数** $-\eta J_f(x')g$;方向导数有比「跑两个模型再相减」廉价得多的算法。配套可运行脚本见 `single_step_probe.py`(§B)。

### 7+.1 要算的精确量

在训练样本 $x$ 上走一步:$g=\nabla_\theta L(x)$,$\Delta\theta=-\eta g$。探针 $x'$ 上的输出变化:

$$
\Delta f(x') \approx J_f(x';\theta)\,\Delta\theta = -\eta\,J_f(x';\theta)\,g
$$

两个结构性事实派生出所有捷径:① $J_f\Delta\theta$ 是 **JVP**,前向模式 AD 一次前向算出,不构造 $\theta'$;② $g$ 本身是梯度,整体是「Jacobian 乘梯度」—— NTK、影响函数、TracIn 的共同对象。探针选择决定问题:$x'=x$ 问拟合速度,$x'\neq x$ 问泛化/干扰。

### 7+.2 成本阶梯(按需付费)

| 阶梯 | 输出粒度 | 公式 | 成本 | 理论 | FlashAttention |
|---|---|---|---|---|---|
| ① | 探针 loss(标量) | $\Delta L(x')\approx -\eta\langle g(x'),g(x)\rangle$ | 2×backward + 点积 | TracIn / 影响函数 | ✅ 可用 |
| ② | 探针 logits(向量) | $\Delta z(x')\approx -\eta\,\Theta(x',x)\nabla_zL(x)$ | 1×backward + 1×JVP | 经验 NTK | ❌ 需 eager |
| ③ | 每层 hidden | $\widehat{\Delta h_\ell}=J_{h_\ell}(x')\Delta\theta$ | 同②(一次 JVP 全拿) | 前向模式 AD | ❌ 需 eager |

其中 $\Theta(x',x)=J_z(x')J_z(x)^\top$ 是经验 NTK;阶梯①只是把 logits 空间的耦合再收缩成标量:$\langle g(x'),g(x)\rangle=\nabla_zL(x')^\top\Theta(x',x)\nabla_zL(x)$。**三层是同一个量 $-\eta J_f(x')g$ 的不同读出粒度。**

### 7+.3 前向模式 JVP 机制

前向模式用**对偶数** $(h_\ell,\dot h_\ell)$ 逐层流动:主值正常走,切值按算子线性化同步走,切向量取 $\Delta\theta$,则**每层切值 $\dot h_\ell = J_{h_\ell}\Delta\theta$ 恰好是该层 hidden 的一阶预测变化**。一次前向跑完,所有层 $\widehat{\Delta h_\ell}$ 和 $\widehat{\Delta z}$ 全拿。资源优势:前向模式**不缓存激活**(不反传),显存远低于一次 backward,也从不物化第二份权重。

### 7+.4 精度边界

二阶余项 $f(\theta')=f(\theta)+J_f\Delta\theta+\tfrac12\Delta\theta^\top H_f\Delta\theta+\cdots$,误差 $=O(\eta^2\|g\|^2)$。小 $\eta$/单步 → 准(NTK / lazy 区)。**自检指标**(做一次即可):真跑更新后模型,算

$$
\rho=\frac{\|\Delta f_{\text{true}}-\Delta f_{\text{lin}}\|}{\|\Delta f_{\text{true}}\|}
$$

$\rho$ 小则放心用 JVP 批量跑探针;$\rho$ 大则已进非线性区,回 Regime 2 实跑。

### 7+.5 有限差分回退 + 统一视角

前向模式不支持的算子(FlashAttention 内核)用有限差分估 JVP:

$$
J_f\Delta\theta\approx\frac{f(\theta+\epsilon\Delta\theta)-f(\theta-\epsilon\Delta\theta)}{2\epsilon}
$$

只需 forward,FlashAttention 可用。**统一视角**:$\theta+1\cdot\Delta\theta$ 就是真更新后模型,故 $\epsilon\to0$=纯线性(JVP),$\epsilon=1$=真实相减,二者只差二阶余项;扫 $\epsilon$ 即可分解线性/非线性。

### 7+.6 工程坑

- `model.eval()` 关 dropout,否则切值无意义。
- **JVP 必须 eager / SDPA-math**;FlashAttention 仅用于阶梯①与 FD 回退。
- 显存峰值在拿 $g$ 那次 backward;JVP 本身省。8B 全参梯度树 bf16 ≈ 16GB,单卡 80G 可跑,小卡需限制可微参数子集。
- 切值在 fp16 易被吃掉,关键比较用 fp32 / bf16。

---

## 8. 速查总结

- **loss 的数值没用,梯度/变化才有用** —— 一切「简化」的前提。
- **forward 省不掉**(backward 需要激活),除非走零阶(MeZO)。
- **三种省显存路线**:LoRA 压更新秩、GaLore 压梯度秩、MeZO 去掉梯度(连激活一起省)。
- **前后对比**:小更新用 JVP / NTK 一次前向预测;大更新(整段 SFT)用 checkpoint 相减 + 逐层范数,别做多余 backward。

---

## 参考

- Malladi et al., *Fine-Tuning Language Models with Just Forward Passes* (MeZO), 2023.
- Hu et al., *LoRA: Low-Rank Adaptation of Large Language Models*, 2021.
- Zhao et al., *GaLore: Memory-Efficient LLM Training by Gradient Low-Rank Projection*, 2024.
- Ilharco et al., *Editing Models with Task Arithmetic*, 2022.
- Jacot et al., *Neural Tangent Kernel*, 2018.
