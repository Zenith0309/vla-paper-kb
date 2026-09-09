# 5. World Model 与机器人结合

本分块解决的核心问题是：**能否让机器人先"想象"动作的后果，再决定做什么；以及能否用生成式视频模型来替代昂贵的真机数据与真机评测。** 目前有四种结合方式，界限清晰：（a）**world model 作为规划器**——生成未来图像/子目标，再由低层策略追踪（UniPi、SuSIE）；（b）**world model 作为表示预训练**——用未来帧预测这一辅助任务给策略学出动态感知的表示（GR-1/GR-2）；（c）**world model 作为数据引擎**——生成合成轨迹视频，再用逆动力学或 latent action 反推动作标签，扩充训练集（DreamGen）；（d）**world model 与 policy 统一建模**——图像、语言、动作共享一个自回归/扩散主干，互为条件（WorldVLA、Genie Envisioner）。

未解决的瓶颈：（1）**生成视频的物理一致性不足**（穿模、物体消失、质量不守恒），直接用于规划会产生不可执行的目标；（2）**评估困难**——FVD/PSNR 与下游控制成功率相关性弱，缺乏面向控制的 world model 评测指标；（3）**推理成本**——视频生成的开销比动作生成高 2–3 个数量级，实时闭环规划仍不现实；（4）**逆动力学的误差累积**——由生成视频反推的伪动作噪声大，长程序列尤甚。

**技术演进链：**

`UniPi（文本条件视频生成 → 逆动力学 → 动作）→ SuSIE（图像编辑生成子目标 → 低层策略追踪）→ GR-1（视频生成式预训练作为策略的辅助任务）→ DreamGen（world model 当数据引擎，合成新场景/新行为）→ Genie Envisioner（统一的视频世界基座：策略 + 仿真 + 评测）→ WorldVLA（动作与图像互为预测目标的统一自回归模型）`

- **UniPi → SuSIE**：从"生成完整未来视频"降级为"**只生成一个关键子目标图像**"，大幅降低生成负担与物理不一致风险，同时保留语义规划能力。
- **SuSIE → GR-1**：不再在推理时生成，而是把视频预测作为**预训练辅助任务**，让策略网络内化动态先验，推理时零额外开销。
- **GR-1 → DreamGen**：把 world model 从"训练技巧"变成"**数据生产设施**"——在没有真机的新场景/新行为上生成视频，再反推动作，直接解决真机数据稀缺。
- **DreamGen → Genie Envisioner**：把生成、策略、仿真评测统一到一个视频世界基座上，尝试用 world model **替代仿真器**做策略评估。
- **Genie Envisioner → WorldVLA**：结构上的统一——同一个自回归 Transformer 同时建模"动作 → 未来图像"和"图像 → 动作"，让世界模型与策略互相正则。

## 论文表

| 年份 | 论文 | 作者 / 机构 | Venue | 论文类型 | 核心贡献 | 标签 | 归类理由 | 与同类工作的关系 / 演进 | 数据集或 benchmark | 开源情况与链接 | 注意事项 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 2023 | Learning Universal Policies via Text-Guided Video Generation (UniPi) | Du, Yang, Dai 等 / MIT, Google DeepMind, UC Berkeley | NeurIPS 2023 | 方法 | 把序贯决策重述为**文本条件的视频生成问题**：先用扩散视频模型生成"完成任务的未来帧序列"，再用逆动力学模型从相邻帧反推动作。**动作表示**：由逆动力学模型输出的连续动作；**输入**：当前观测图像 + 文本任务；**输出**：先视频、后动作。展示组合泛化与跨环境迁移 | `奠基性论文` | 它确立了"**视频生成即策略**"这一今天仍在演进的完整路线（DreamGen、Genie Envisioner 都是其后代），并首次论证视频作为"通用的、与本体无关的行为接口"。归类依据是范式开创性 | 是 SuSIE、DreamGen、Genie Envisioner 的共同祖先；与 GR-1 的"生成作为辅助任务"是**并行路线**（推理时生成 vs 训练时生成）；与传统 model-based RL（Dreamer 系列）的区别在于**在像素空间而非潜空间规划** | 组合式仿真任务（CLIPort 风格）、真机迁移实验 | 代码**未完整开源**（官方未发布训练代码）。主页：https://universal-policy.github.io/ ；arXiv: https://arxiv.org/abs/2302.00111 | 推理需完整视频生成，**开销极高、无法闭环实时**；生成视频的物理一致性差，逆动力学误差累积严重；主要在受控仿真环境验证，真机结果有限 |
| 2023 | Zero-Shot Robotic Manipulation with Pretrained Image-Editing Diffusion Models (SuSIE) | Black, Nakamoto, Atreya 等 / UC Berkeley, Stanford | ICLR 2024 | 方法 | 用预训练的**图像编辑模型（InstructPix2Pix）**在语言指令下把当前观测"编辑"成一个近期子目标图像，再由目标条件的低层策略（diffusion policy）追踪该子目标；高层每隔若干步重新生成一次子目标，形成闭环。**动作表示**：低层 diffusion 动作块；**输入**：当前图像 + 语言；**输出**：子目标图像 → 连续动作 | `当前代表作` | 它给出了 world-model 路线中**成本最低、最实用的形式**（生成单帧而非视频），并证明互联网图像编辑先验可直接迁移到机器人子目标生成，实现对未见物体的零样本操作。归类依据是该"子目标图像"接口被大量后续分层工作复用 | 是 UniPi 的**轻量化改进**（单帧代替视频序列）；与 GR-1 是并行路线；其"子目标图像作为高低层接口"的设计被后续 hierarchical VLA（含 π0.5 的子任务语义接口）在不同模态上重演 | 训练：BridgeData V2 + Something-Something（人类视频）；评测：CALVIN、真机 WidowX | **开源**。代码：https://github.com/kvablack/susie ；主页：https://rail-berkeley.github.io/susie/ ；arXiv: https://arxiv.org/abs/2310.10639 | 子目标图像可能生成**物理不可达**的状态（穿模、物体瞬移），低层策略会被误导；高层生成频率是强超参；对精细操作（插入、拧紧）无能为力，因为差异在像素上不可见 |
| 2023 | Unleashing Large-Scale Video Generative Pre-training for Visual Robot Manipulation (GR-1) | Wu, Yin, Wang 等 / 字节跳动研究院 | ICLR 2024 | 方法 | 提出 GPT 风格的统一 Transformer：在**大规模人类视频（Ego4D）上以"预测下一帧"做生成式预训练**，再在机器人数据上微调为"同时预测未来图像 + 未来动作"的多任务模型。**动作表示**：连续末端位姿 delta + 夹爪（后续 GR-2 扩展为动作块）；**输入**：多帧图像历史 + 语言 + 本体状态；**输出**：未来图像 token + 连续动作。在 CALVIN 长程任务上大幅刷新当时结果 | `奠基性论文` | 它确立了"**视频预测作为机器人策略的预训练目标**"这一今天最被广泛采用的 world-model 结合方式（GR-2、GR-3、Genie Envisioner、多数国产 VLA 均沿用），并给出人类视频→机器人的清晰迁移证据。归类依据是该训练目标的普及度 | 相对 UniPi/SuSIE 的"推理时生成"是**替代路线**（把生成成本移到训练期）；被 GR-2（更大数据、动作块）与 GR-3（VLA 化、加入视觉-语言共训）迭代；与 DreamGen 的差别是"内化动态"vs"生产数据" | 预训练：Ego4D；评测：CALVIN（ABC→D 等）、真机 Kinova | 代码**开源**（GR-1）；GR-2 / GR-3 **未开源**。代码：https://github.com/bytedance/GR-1 ；主页：https://gr1-manipulation.github.io/ ；arXiv: https://arxiv.org/abs/2312.13139 | GR-1 本身**不是 VLA**（语言编码为 CLIP 文本嵌入，无 LLM）；CALVIN 上的强结果**未在真机上等比例复现**；后续 GR-2/GR-3 的技术报告细节与数据均不公开，**性能主张待核验** |
| 2025 | DreamGen: Unlocking Generalization in Robot Learning through Video World Models | NVIDIA GEAR（Jang, Liang, Zhu 等） | arXiv（**venue 待核验**） | 方法 + 数据 | 提出四步"神经轨迹"流水线：①在目标机器人数据上微调视频世界模型 →②用新的语言提示生成大量新行为/新场景的机器人视频 →③用潜动作模型或逆动力学模型提取伪动作 →④用这些合成轨迹训练策略。核心主张：**用生成数据可获得对未见行为与未见环境的泛化**，且只需单一场景的真机数据启动。**动作表示**：由 IDM/latent action 反推的连续动作块 | `近期候选代表作` | 思路上把 world model 明确定位为"数据引擎"，是解决真机数据瓶颈的重要方向，并配套发布 DreamGen Bench；但截至检索日**独立复现与第三方评测有限**，且效果高度依赖 NVIDIA 的 Cosmos 视频模型，故标为候选 | 是 UniPi 路线的**现代化重构**（不再在推理时生成，而是离线生成训练数据）；与 GR-1 的辅助任务路线**互补而非竞争**；与仿真数据生成（RoboCasa、RoboTwin 2.0，第 7 类）是同一目标的两条路径（生成模型 vs 物理仿真） | 训练/生成：Cosmos / WAN 视频模型 + 少量真机数据；评测：DreamGen Bench、RoboCasa、真机 GR-1 与 Franka | **开源**（代码 + benchmark）。代码：https://github.com/NVIDIA/GR00T-dreams ；主页：https://research.nvidia.com/labs/gear/dreamgen/ ；arXiv: https://arxiv.org/abs/2505.12705 | 伪动作噪声是主要误差源，**长程任务上退化明显**；视频生成成本高（每条轨迹分钟级 GPU 时间）；"泛化到新行为"的评测由作者自定义，**缺乏跨机构可比协议**；对接触丰富、需力反馈的任务无效（视频里看不见力） |
| 2025 | Genie Envisioner: A Unified World Foundation Platform for Robotic Manipulation | AgiBot（智元机器人）/ 上海 AI Lab 等 | arXiv（**venue 待核验**） | 系统 + 方法 | 提出统一的视频世界基座平台，三个组件共享一个潜空间：**GE-Base**（指令条件的多视角视频世界模型）、**GE-Act**（把世界表示映射为动作的轻量流匹配策略头）、**GE-Sim**（把世界模型当作动作条件的神经仿真器做闭环 rollout 与评测）。**动作表示**：flow matching 连续动作块；**输入**：多视角图像 + 语言；**输出**：未来视频 + 动作块 | `近期候选代表作` | 它是"**用 world model 同时做策略、仿真与评测**"这一激进主张的最完整实现，若成立将改变策略评测方式；但截至检索日**第三方复现与跨机构评测缺失**，其神经仿真器的评估保真度尚未被独立验证，故标为候选 | 综合了 GR-1（生成式预训练）、UniPi（生成即策略）与 DreamGen（生成即数据）三条线；与 WorldVLA 的差别是**模块解耦**（三组件共享潜空间但可独立使用）而非单一序列模型 | 训练：AgiBot World 大规模真机数据；评测：自建 EWMBench 等世界模型评测 + 真机 | **开源**（代码 + 权重）。代码：https://github.com/AgibotTech/Genie-Envisioner ；arXiv: https://arxiv.org/abs/2508.05635 | **神经仿真器用于策略评测的可靠性未经独立验证**——这是全文最强也最需谨慎对待的主张；训练依赖 AgiBot 自有的大规模数据，其他实验室难以复现；视频生成开销使闭环控制频率受限 |
| 2025 | WorldVLA: Towards Autoregressive Action World Model | Cen, Yu, Yuan 等 / 阿里巴巴达摩院 | arXiv（**venue 待核验**） | 方法 | 用**单一自回归 Transformer 统一图像、语言、动作三种 token**：既学"给定图像+语言 → 动作"（VLA 方向），也学"给定图像+动作 → 下一帧图像"（world model 方向），二者互为条件形成闭环。同时提出 **attention mask 策略**缓解动作块自回归生成中的误差传播。**动作表示**：离散动作 token（自回归）；**输入**：图像 token + 语言 token +（可选）动作 token；**输出**：动作 token 或图像 token | `近期候选代表作` | "动作与世界互为预测目标"的统一形式是本分块最简洁的表述，且论文给出了双向增益的消融证据（加世界模型任务提升动作、加动作任务提升视频预测）；但评测集中在 LIBERO 单一 benchmark，**真机与跨机构证据不足**，故标为候选 | 相对 GR-1 的"辅助任务"是**更彻底的统一**（共享同一词表与同一自回归目标）；相对 Genie Envisioner 是**单模型 vs 平台化**的并行路线；动作 token 化沿用 RT-2/OpenVLA 谱系 | 评测：LIBERO 四套件（**主要证据来源**） | **开源**。代码：https://github.com/alibaba-damo-academy/WorldVLA ；arXiv: https://arxiv.org/abs/2506.21539 | **仅在 LIBERO 上验证，无真机实验**——这是其最主要的局限；自回归图像 token 生成开销大，实时性未讨论；venue 待核验 |

> **未入表但需跟踪**：Dreamer / DayDreamer（潜空间 world model + 在线 RL，非 VLA 且非生成式视频路线，是本分块的"另一支血脉"）；NVIDIA Cosmos World Foundation Model（视频世界模型基座，DreamGen 的底层依赖）；GR-2 / GR-3（字节跳动，未开源，**技术报告细节待核验**）；2026 年出现的 WorldArena、World-Language-Action 等统一模型（**均待核验**）。

## 交叉标签

- **UniPi** → 交叉：`长程规划`、`跨本体（视频作为本体无关接口）`
- **SuSIE** → 交叉：`System1/System2（子目标接口）`、`模仿学习基础（低层 DP）`
- **GR-1** → 交叉：`基础架构（生成式预训练）`、`人类视频数据`、`CALVIN benchmark`
- **DreamGen** → 交叉：`数据 scaling`、`仿真/合成数据`、`GR00T 生态`
- **Genie Envisioner** → 交叉：`benchmark（神经仿真评测）`、`AgiBot World 数据`
- **WorldVLA** → 交叉：`动作解码-token`、`基础架构统一序列建模`

## 四种结合方式的取舍

| 结合方式 | 代表工作 | 推理开销 | 主要收益 | 主要风险 |
|---|---|---|---|---|
| 生成即策略（推理时生成未来） | UniPi, SuSIE | 高 / 中 | 语义规划能力强、可零样本组合 | 物理不一致、逆动力学误差 |
| 生成作为预训练辅助任务 | GR-1, GR-2 | 零额外开销 | 内化动态先验、样本效率高 | 收益隐式、难以定位失败原因 |
| 生成作为数据引擎 | DreamGen | 离线高、在线零 | 直接缓解真机数据稀缺 | 伪动作噪声、评测协议不统一 |
| 世界模型与策略统一建模 | WorldVLA, Genie Envisioner | 中–高 | 双向正则、可做神经仿真评测 | 评测保真度未验证、实时性差 |
