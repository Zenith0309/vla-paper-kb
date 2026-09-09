# 1. VLM/VLA 基础架构

本分块回答的核心问题是：**一个在互联网图文数据上预训练好的 VLM，如何被改造成能输出机器人动作的策略网络，同时不丢失其语义泛化能力。** 主流技术路线有两条：一是「**统一词表**」路线，把动作离散化成 token 直接混进 VLM 的语言词表做联合微调（RT-2、OpenVLA）；二是「**backbone + action expert**」路线，VLM 只负责编码，另接一个专用的连续动作专家（π0 的 flow matching expert、GR00T N1 的 DiT）。后者已成为 2025 年之后的事实主流，因为它把「语义理解频率」和「控制频率」解耦。
当前未解决的瓶颈：（1）**VLM 预训练的语义能力在动作微调后会显著退化**（catastrophic forgetting），如何在动作数据上继续保持开放世界识别能力仍无公认方案；（2）**多帧历史与本体状态（proprioception）的融合方式**尚无定论，多数工作证明简单加历史帧反而掉点（因果混淆 / copycat 问题）；（3）**参数量与性能的关系不单调**，7B 级 VLA 在很多任务上打不过 100M 级专用策略，说明 backbone 容量并非当前瓶颈。

**技术演进链：**

`RT-2 → OpenVLA → π0 → GR00T N1 → LAPA`

- **RT-2**：首次证明「把动作写成文本 token、与 VQA 数据共同微调 VLM」可行，带来了 emergent 语义泛化（认识没见过的物体、能执行需要常识的指令）。相对前作 RT-1 的关键变化是 **backbone 从头训练 → 复用互联网预训练 VLM**。
- **OpenVLA**：把 RT-2 的闭源配方**完全开源化并跑通** —— 7B Prismatic 骨干 + OXE 97 万条轨迹，附带 LoRA/量化微调方案。关键变化是 **可复现性与社区基线的建立**，而非性能本身。
- **π0**：放弃离散化，改用 **flow matching action expert** 输出 50Hz 连续动作块。关键变化是 **动作表示从「离散 token 的分辨率瓶颈」转向「连续分布建模」**，首次做到叠衣服等高频灵巧任务。
- **GR00T N1**：把 π0 的思路系统化为 **双系统架构**（VLM 慢系统 + DiT 快系统 + embodiment-specific 编解码头），并把训练数据扩展到「真机 + 仿真 + 人类视频」三源混合。关键变化是 **人形机器人多本体的统一架构与开放权重**。
- **LAPA**：不再要求训练数据带动作标签，从**无动作的人类/机器人视频中学习 latent action**，再用少量真机数据把 latent 映射回真实动作。关键变化是 **把 VLA 预训练的数据门槛从「带动作标注」降到「只要视频」**。

## 论文表

| 年份 | 论文 | 作者 / 机构 | Venue | 论文类型 | 核心贡献 | 标签 | 归类理由 | 与同类工作的关系 / 演进 | 数据集或 benchmark | 开源情况与链接 | 注意事项 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 2023 | RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control | Brohan, Brown, Carbajal 等 / Google DeepMind | CoRL 2023 | 方法 | 提出 VLA 概念本身：把 7 自由度动作离散成 256 bin 的字符串 token，与 VQA 数据 co-fine-tune PaLI-X / PaLM-E。**动作表示**：离散 token（自回归）；**输入**：单帧图像 + 语言指令；**输出**：末端位姿 delta + 夹爪开合的离散 token 串。首次展示 emergent 能力（识别未见物体、符号推理、数字/图标理解） | `奠基性论文` | 定义了整个领域的问题形式与命名；后续几乎所有 VLA 论文都以它为叙事起点，且 "web knowledge transfer" 的论证方式被反复沿用。归类为奠基不是因为成功率高（真机成功率有限），而是因为它确立了「VLM 微调即策略」的范式 | 继承 RT-1 的动作离散化与数据管线，替换其 from-scratch backbone 为互联网预训练 VLM；被 OpenVLA 开源复现、被 π0 在动作表示上取代 | RT-1 内部数据集（13 台机器人 / 17 个月）+ WebLI；评测为 Google 内部真机 setup | **未开源**（模型、数据、代码均闭源）。论文页：https://robotics-transformer2.github.io/ ；arXiv: https://arxiv.org/abs/2307.15818 | 评测完全在 Google 内部真机 setup，**外部无法复现**；离散 token 分辨率限制其无法做高频灵巧任务；推理频率仅 1–3 Hz |
| 2024 | OpenVLA: An Open-Source Vision-Language-Action Model | Kim, Pertsch, Karamcheti 等 / Stanford, UC Berkeley, TRI, Google DeepMind | CoRL 2024 | 方法 + 工程框架 | 7B 开源 VLA：Prismatic（SigLIP + DINOv2 双视觉编码器 + Llama-2 7B）骨干，在 OXE 的 97 万条轨迹上训练。**动作表示**：离散 token（覆盖 Llama 词表中最少用的 256 个 token）；**输入**：单帧第三人称图像 + 语言；**输出**：7-DoF 末端 delta。提供 LoRA 微调与 4-bit 量化部署方案 | `奠基性论文` | 它是**社区事实标准基线**：2024–2026 年几乎每一篇 VLA 方法论文都必须与 OpenVLA 对比，LIBERO / SimplerEnv 的排行榜以它为原点。归类为奠基的理由是「可复现基础设施」的贡献而非算法新颖性 | 是 RT-2 的开源复现与改进（双视觉编码器、更大数据混合）；被 OpenVLA-OFT（并行解码）、SpatialVLA（3D 位置编码）、UniVLA（latent action）等一系列工作作为直接底座改造 | 训练：Open X-Embodiment；评测：BridgeData V2 真机、Google Robot、LIBERO、SimplerEnv | **完全开源**（代码 + 权重 + 训练配方）。代码：https://github.com/openvla/openvla ；主页：https://openvla.github.io/ ；arXiv: https://arxiv.org/abs/2406.09246 | 原始版本推理慢（~6 Hz，单动作自回归解码），不支持 action chunking，长程/高频任务表现弱；对本体状态（proprioception）默认不使用；单帧输入使其对遮挡与部分可观测场景脆弱 |
| 2024 | π0: A Vision-Language-Action Flow Model for General Robot Control | Black, Brown, Driess 等 / Physical Intelligence | arXiv（**venue 待核验**） | 方法 + 系统 | 首个把 **flow matching** 用作 VLA 动作头的工作。3B PaliGemma 骨干 + 300M action expert（独立权重、共享注意力），输出 50 步动作块。**动作表示**：flow matching 连续动作 chunk（H=50）；**输入**：多路相机图像 + 语言 + proprioception；**输出**：连续关节/末端动作块，最高 50 Hz。在叠衣服、装盒、清桌等灵巧任务上给出首个可信的长时程演示 | `当前代表作` | 它把「连续动作生成」确立为高频灵巧操作的必要条件，之后 GR00T、SmolVLA、X-VLA 等主流架构全部采用 action expert + 连续/flow 输出。归类为当前代表作的依据是：架构被广泛继承、openpi 开源后被大量第三方微调复现 | 相对 RT-2/OpenVLA 的离散 token 是**替代路线**；相对 Diffusion Policy 是把 diffusion/flow 从小型专用策略**放大到 VLM 尺度**；其自身的离散 token 版本见 π0-FAST（第 2 类） | 训练：Physical Intelligence 内部 ~10000 小时多本体数据 + OXE；评测：内部真机任务 + 后续社区在 LIBERO / SimplerEnv 上复现 | **权重与推理代码开源**（openpi），**训练数据闭源**。代码：https://github.com/Physical-Intelligence/openpi ；主页：https://www.physicalintelligence.company/blog/pi0 ；arXiv: https://arxiv.org/abs/2410.24164 | 论文中的旗舰结果依赖**闭源的内部大规模数据**，开源权重上无法完全重现；真机评测协议为自建，跨机构不可比；flow matching 需多步去噪，未做加速时推理开销高于单步回归 |
| 2025 | GR00T N1: An Open Foundation Model for Generalist Humanoid Robots | NVIDIA GEAR | arXiv | 方法 + 系统 | 面向人形机器人的开放基础模型：**双系统**架构 —— System 2 为 Eagle-2 VLM（低频语义推理），System 1 为 Diffusion Transformer 动作专家（高频控制），二者端到端联合训练；用 embodiment-specific 的 state/action 编解码头统一异构本体。**动作表示**：diffusion / flow 连续动作块；**输入**：多视角图像 + 语言 + 本体状态；**输出**：各本体自身关节空间动作块。训练数据为「真机 + 仿真 + 人类第一人称视频」金字塔混合 | `当前代表作` | 是首个**权重、代码、部分数据全开放**的人形机器人基础模型，并被 N1.5 / N2 迭代持续验证；在 RoboCasa GR-1、社区人形平台上被广泛用作起点。归类依据是开放生态影响力 + 双系统架构被后续大量沿用 | 把 π0 的 backbone+expert 思路系统化为显式双系统，并首次把**人类视频**纳入 VLA 预训练金字塔；与 Helix（闭源）是并行路线；后继 N1.5、N1.6、N2 持续迭代 | 训练：内部 GR-1 遥操数据 + DexMimicGen 仿真 + Ego4D 类人类视频；评测：RoboCasa GR-1 Tabletop、DexMimicGen 仿真套件、真实 GR-1 | **开源**（权重 + 代码 + 仿真数据）。代码：https://github.com/NVIDIA/Isaac-GR00T ；主页：https://developer.nvidia.com/isaac/gr00t ；arXiv: https://arxiv.org/abs/2503.14734 | 真机评测集中在 Fourier GR-1 单一平台；论文报告的仿真增益依赖 NVIDIA 自建数据生成管线，其他实验室复现成本高；N1.5 / N1.6 / N2 的技术报告成熟度不一，**N2 截至检索日仍为预览状态，性能主张待核验** |
| 2024 | Towards Generalist Robot Policies: What Matters in Building Vision-Language-Action Models (RoboVLMs) | Li, Li, Zhang 等 / 清华、字节跳动等 | arXiv（**venue 待核验**） | 方法 + 工程框架 | 系统性消融实验：在统一代码框架下交叉比较 4 类 VLM backbone × 4 类 action head（one-step / interleaved / policy-head / latent），回答「backbone 怎么选、动作头接在哪、历史帧要不要、什么时候引入机器人数据」。结论包括：**带历史的 policy-head 结构总体最优**、连续动作优于离散、VLM 的视觉-语言对齐质量比参数量更关键 | `当前代表作` | 这是本分块少有的**控制变量式实证研究**，为后续架构选择提供了可引用的经验依据，而非仅提出一个新模型。归类理由是它被大量后续工作用作设计决策的引证来源 | 与 RT-2 / OpenVLA / π0 是「元层面」关系：不提出新范式，而是对已有范式做公平比较；其开源 RoboVLMs 代码库后来演化为多个 VLA 框架的参考实现 | 评测：CALVIN、SimplerEnv、真机 WidowX / Kinova | **开源**。代码：https://github.com/Robot-VLAs/RoboVLMs ；主页：https://robovlms.github.io/ ；arXiv: https://arxiv.org/abs/2412.14058 | 消融规模受算力限制，多数结论在 CALVIN / SimplerEnv 上得出，**是否外推到百万级真机数据未验证**；结论具有时效性（其时 flow matching 尚未普及） |
| 2024 | Latent Action Pretraining from Videos (LAPA) | Ye, Kim, Suhr 等 / KAIST, UW, AI2, NVIDIA | ICLR 2025 | 方法 | 提出**无动作标注的 VLA 预训练**：先用 VQ-VAE 从相邻视频帧中无监督学习离散 latent action，再让 VLM 学「图像+语言 → latent action」，最后用少量带动作的真机数据把 latent 解码为真实动作。**动作表示**：离散 latent action token → 下游解码为连续动作；**输入**：图像 + 语言；**输出**：latent token（预训练）/ 真实动作（微调后） | `当前代表作` | 它把 latent action representation 从概念变为可规模化的预训练手段，直接催生了 UniVLA、GO-1 等一批 latent-action 工作，是「数据门槛下降」这条线的起点。归类依据是明确的技术转折意义与后续继承密度 | 是对 RT-2/OpenVLA「必须有动作标注」假设的**替代路线**；被 UniVLA（task-centric latent action，第 4 类）改进为语言条件化、去除任务无关运动；与 GR00T N1 的人类视频利用是并行思路 | 预训练：Open X-Embodiment + Something-Something V2（人类视频）；评测：LIBERO、SIMPLER、真机 | **开源**。代码：https://github.com/LatentActionPretraining/LAPA ；主页：https://latentactionpretraining.github.io/ ；arXiv: https://arxiv.org/abs/2410.11758 | latent action 的语义可解释性差，难以调试；在需要精细力控的任务上，latent 空间的量化误差会成为上界；下游仍需真机动作数据做解码器对齐，并未完全免除标注 |

## 交叉标签

- **RT-2** → 交叉：`动作解码-离散token`、`跨本体（RT-2-X）`
- **OpenVLA** → 交叉：`开源生态`、`benchmark 基线`、`动作解码-离散token`
- **π0** → 交叉：`动作解码-flow matching`、`开源框架 openpi`、`实时部署`
- **GR00T N1** → 交叉：`System1/System2`、`跨本体`、`仿真数据生成`
- **RoboVLMs** → 交叉：`开源框架`、`动作解码范式对比`
- **LAPA** → 交叉：`跨本体`、`数据 scaling`、`world model（隐式动态建模）`

## 本分块尚未解决的问题（供选题参考）

1. **语义能力保持**：动作微调后 VLM 的开放词表识别能力退化程度缺乏系统度量，也没有标准的「语义-控制」联合评测。
2. **历史与状态融合**：多帧历史在真机上普遍带来 copycat/因果混淆；proprioception 何时该输入、以什么形式输入（绝对/相对/归一化）无共识。
3. **backbone 选择**：是否需要 7B 级 VLM 仍有争议 —— SmolVLA（450M）在多个 benchmark 上接近 7B 模型，暗示当前瓶颈在数据而非容量。
4. **MoE 化**：action expert 已是事实上的两专家 MoE，但真正的稀疏 MoE VLA（按任务/本体路由）仍是开放方向，2026 年出现的相关工作证据不足。
