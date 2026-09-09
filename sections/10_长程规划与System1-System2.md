# 10. 长程任务规划与 System 1 / System 2

本分块解决的核心问题是：**单个端到端 VLA 在几十秒以上的任务上会因缺乏任务级记忆与纠错而崩溃，如何用一个"慢的、会推理的"上层与一个"快的、会控制的"下层组合起来。** 高低层之间的**接口形式**是这条线的真正技术分歧点，目前有五种：**可执行代码**（Code as Policies）、**技能名 + 可供性打分**（SayCan）、**3D 价值图**（VoxPoser）、**自然语言子任务**（Hi Robot、π0.5）、**语言推理链 + 中间视觉表示**（ECoT、Gemini Robotics 1.5）。

一个重要的经验事实：**加入显式推理几乎总是提升长程与新任务表现，但会降低控制频率**；因此近期工作的共同方向是**异步双系统**（上层低频、下层高频）与**推理的按需触发**（简单场景不推理）。

未解决的瓶颈：（1）**闭环重规划的触发条件**没有原则——多数系统固定频率重规划，不会因"察觉失败"而重规划；（2）**长时记忆缺失**——绝大多数系统只看当前帧，无法处理"把刚才拿走的那个放回去"；（3）**推理链的正确性无监督**——CoT 可能自洽但与实际执行脱节（推理说"先开抽屉"而动作在抓杯子）；（4）**高低层的责任归属难诊断**——失败时无法自动判断是规划错还是执行错。

**技术演进链：**

`SayCan（LLM 提供语义可行性 × 价值函数提供物理可行性）→ Code as Policies（LLM 直接生成可执行策略代码）→ VoxPoser（LLM+VLM 生成 3D 价值图，绕开固定技能库）→ ECoT（把推理链直接嵌入 VLA 的自回归输出）→ π0.5（高层输出自然语言子任务，与低层动作在同一模型内共训）→ Gemini Robotics 1.5（多层内部推理 + 跨本体 Motion Transfer）`

- **SayCan → Code as Policies**：从"在**固定技能库**中打分选择"变为"**现场组合出新程序**"（含循环、条件、反馈），表达力大幅提升，但要求所有原子操作都有可调用的 API。
- **Code as Policies → VoxPoser**：不再依赖预定义 API，改为让 LLM 写代码去**操纵 3D 观测生成价值图**，由运动规划器求解轨迹——把"技能库"的约束换成"感知 + 优化"。
- **VoxPoser → ECoT**：从"上层调用下层"转为"**同一个 VLA 内部先输出推理再输出动作**"（依次生成任务改写、子任务、可见物体、末端位置、夹爪动作），使推理与动作共享参数与梯度。
- **ECoT → π0.5**：把推理简化为**高层自然语言子任务预测**，并与低层 flow matching 动作在**同一模型内分两阶段推理**，从而在真实家庭中完成十分钟量级的清洁任务。
- **π0.5 → Gemini Robotics 1.5**：引入**多层次内部推理**（把长任务分解并在动作之间穿插思考），并用 **Motion Transfer** 让不同本体共享运动知识，把"能思考"和"能跨本体"合到一个模型。

## 论文表

| 年份 | 论文 | 作者 / 机构 | Venue | 论文类型 | 核心贡献 | 标签 | 归类理由 | 与同类工作的关系 / 演进 | 数据集或 benchmark | 开源情况与链接 | 注意事项 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 2022 | Do As I Can, Not As I Say: Grounding Language in Robotic Affordances (SayCan) | Ahn, Brohan, Brown 等 / Google, Everyday Robots | CoRL 2022 | 方法 + 系统 | 提出把 LLM 的**语义似然**（这个技能对完成指令有多相关）与学到的**价值函数/可供性**（在当前状态下这个技能有多可能成功）**相乘**来选择下一个技能，从而让 LLM 的建议"落地"到机器人真实能做的事上。**动作表示**：从固定技能库中选择一个预训练技能（每个技能是一个独立策略） | `奠基性论文` | 它是"**LLM 做高层、学习策略做低层**"这一分层范式的原型，"grounding language in affordances"的表述定义了整个子领域的问题形式。归类依据是范式开创性 | 被 Code as Policies 在表达力上超越（技能库 → 任意程序）；被 VoxPoser 在灵活性上超越（技能库 → 现场生成价值图）；其"可供性打分"思想在今天的失败检测与重规划中仍在使用 | Everyday Robots 真机（101 个长程厨房指令） | 代码**部分开源**（推理框架，不含真机技能）。主页：https://say-can.github.io/ ；arXiv: https://arxiv.org/abs/2204.01691 | **强依赖预定义技能库**，无法完成库外任务；技能与价值函数需逐个训练，扩展成本高；LLM 只在**任务开始时规划一次**，闭环纠错弱；真机平台已停产，**无法复现** |
| 2022 | Code as Policies: Language Model Programs for Embodied Control | Liang, Huang, Xia 等 / Google | ICRA 2023 | 方法 | 让 LLM 直接**生成可执行的 Python 策略代码**：通过 few-shot 提示与"分层代码生成"（遇到未定义函数就递归再生成），组合感知 API、控制原语、第三方库（NumPy/Shapely）以及控制流，从而表达空间关系（"把它放在最左边的块右边 10cm"）、循环与反馈闭环。**动作表示**：调用参数化控制原语（末端位姿等） | `奠基性论文` | 它证明了"**语言模型可以直接写出机器人策略**"，把空间/几何/逻辑约束的表达从"选技能"提升到"写程序"，是 VoxPoser、以及后续所有 LLM-as-planner 工作的直接源头。归类依据是范式转折 | 是 SayCan 的**替代路线**（生成 vs 选择）；被 VoxPoser 扩展（代码不再直接控制机器人，而是构造 3D 价值图）；与今天 agent/tool-use 范式同源 | 真机 UR5e + 桌面任务；HumanEval 风格的策略代码评测 | **开源**（提示与仿真代码）。代码：https://github.com/google-research/google-research/tree/master/code_as_policies ；主页：https://code-as-policies.github.io/ ；arXiv: https://arxiv.org/abs/2209.07753 | **完全依赖感知 API 的质量**（"检测到杯子"这一步若错，代码再对也没用）；生成代码可能不安全/不可执行，需沙箱与校验；无法处理**接触丰富的连续控制**（只能调用现成原语）；对未提供 API 的能力无能为力 |
| 2023 | VoxPoser: Composable 3D Value Maps for Robotic Manipulation with Language Models | Huang, Wang, Xia 等 / Stanford | CoRL 2023 | 方法 | LLM 写代码调用 VLM，在观测的 **3D 体素空间中合成"可供性图 + 约束图"**（哪里该去、哪里要避开、旋转/速度偏好），再用无模型的运动规划器在该价值图上求轨迹。**动作表示**：由价值图上的轨迹优化得到的 6-DoF 末端轨迹；**输入**：RGB-D + 语言；**输出**：末端轨迹（零样本，无需任何机器人训练数据） | `当前代表作` | 它是"**LLM/VLM 的开放世界语义 → 显式 3D 空间表示 → 动作**"这条链路最清晰的实现，且**完全零样本**（不需要机器人演示数据），至今仍是 grounding-to-action 方向最常被引用的非学习型基线。对本库研究方向价值极高 | 是 Code as Policies 的**直接后继**（把"调用技能 API"换成"构造价值图"，摆脱技能库依赖）；与第 6 类的 RoboPoint 是同一问题的两种输出（价值图 vs 关键点）；被后续 ReKep（关系关键点约束）等工作继承与改进 | 真机 Franka + 真机移动平台；自建 zero-shot 日常任务集 | **开源**。代码：https://github.com/huangwl18/VoxPoser ；主页：https://voxposer.github.io/ ；arXiv: https://arxiv.org/abs/2307.05973 | **依赖准确的 RGB-D 与开放词表检测**，感知错误直接传导；价值图 + 运动规划的组合**无法处理接触丰富与柔性物体**；LLM 生成代码的延迟使其**不适合高频闭环**；论文中的任务多为准静态 |
| 2024 | Robotic Control via Embodied Chain-of-Thought Reasoning (ECoT) | Zawalski, Chen, Pertsch 等 / UC Berkeley, Warsaw University of Technology, Stanford | CoRL 2024 | 方法 + 数据 | 让 VLA **在输出动作之前先自回归地输出一串具身推理**：任务改写 → 子任务分解 → 可见物体及其像素坐标 → 末端当前位置 → 下一步动作原语 → 动作 token。通过自动标注管线（用检测器与 VLM 给 BridgeData V2 生成推理链）低成本构造训练数据。**动作表示**：离散动作 token（前置推理 token） | `当前代表作` | 它是"**把推理写进 VLA 自身输出序列**"的代表作，且提供了可复用的**推理链自动标注管线**（这是最实用的贡献）；成功率提升明显（在 OpenVLA 上绝对提升约 28%），并使失败可解释、可通过修改推理链在线纠正 | 相对 SayCan/CaP 的"外挂 LLM"是**内化路线**（推理与动作共享参数）；被 CoT-VLA（视觉子目标作为推理链）、π0.5（自然语言子任务）等以不同推理模态推广；与 Gemini Robotics 1.5 的"thinking"是同一思想的工业规模版本 | 训练：BridgeData V2 + 自动生成的推理链；评测：真机 WidowX（含未见物体/场景/技能） | **开源**（代码 + 权重 + 推理链数据）。代码：https://github.com/MichalZawalski/embodied-CoT ；主页：https://embodied-cot.github.io/ ；arXiv: https://arxiv.org/abs/2407.08693 | **推理 token 显著拖慢推理**（论文报告控制频率下降，需异步/裁剪缓解）；推理链由自动管线生成，**含噪且可能与实际执行不一致**；只在 WidowX 单平台验证；推理正确不保证动作正确（二者的因果链未被强制） |
| 2025 | π0.5: A Vision-Language-Action Model with Open-World Generalization | Physical Intelligence（Black, Brown, Darpinian 等） | arXiv（**venue 待核验**） | 方法 + 系统 | 用**异构数据共训**（多本体机器人数据 + 高层子任务预测 + 网页图文/VQA/物体检测数据）训练单一模型，推理时分两阶段：**先以离散 token 预测下一个语义子任务（如"拿起盘子"），再以该子任务为条件用 flow matching 输出连续动作块**。展示在**训练中从未见过的真实家庭**中完成十分钟量级的整理厨房/卧室任务。**动作表示**：高层离散语义 token + 低层 flow matching 连续动作块 | `当前代表作` | 它是目前**最有说服力的开放世界长程操作演示**（未见过的真实住宅、十分钟量级任务），并把"分层"从两个模型合并为**一个模型两阶段推理**，这一设计已被多个后续工作采用。归类依据是能力展示的量级 + 架构影响 | 是 π0（第 1 类）的**数据与分层扩展**；相对 SayCan/VoxPoser 的外挂 LLM 是**内化替代**；与 ECoT 的差别是推理粒度（子任务 vs 细粒度推理链）；与 Hi Robot（同团队，分层交互式指令跟随）是配套工作 | 训练：PI 内部多本体数据 + 网页数据 + 高层标注；评测：真实住宅（未见环境）的长程清洁任务 | **权重开源**（openpi 中提供 π0.5 系列），**训练数据与完整配方闭源**。主页：https://www.physicalintelligence.company/blog/pi05 ；arXiv: https://arxiv.org/abs/2504.16054 | 旗舰结果依赖**闭源的大规模内部数据**，开源权重无法复现论文中的家庭泛化水平；评测为**自建协议、无第三方复现**；"十分钟任务"的成功判据与部分完成计分方式需查原文；仍无长时记忆机制（不记得几分钟前做过什么） |
| 2025 | Gemini Robotics 1.5: Pushing the Frontier of Generalist Robots with Advanced Embodied Reasoning, Thinking, and Motion Transfer | Google DeepMind | arXiv（**venue 待核验**） | 方法 + 系统 | 提出配对的两个模型：**Gemini Robotics-ER 1.5**（具身推理模型，负责空间理解、任务规划、进度估计，可调用数字工具如搜索）与 **Gemini Robotics 1.5**（VLA，在动作之间穿插多层次自然语言"思考"）；核心新机制是 **Motion Transfer**，使在一种本体上学到的运动知识可迁移到另一种本体。**动作表示**：连续动作块（细节未公开）；**输入**：多视角图像 + 语言 +（ER 侧）工具调用结果 | `近期候选代表作` | 它把"agent 式规划 + 具身推理 + 跨本体运动迁移"整合到工业级系统，并首次把 ER 模型**通过 API 开放给开发者**；但 **VLA 部分权重未公开、评测协议自建、无第三方复现**，因此按本库规则只能标为候选而非当前代表作 | 是 Gemini Robotics（2025 初）的迭代；把 ECoT 的"内嵌推理"与 SayCan 的"高层规划"在工业规模上统一；Motion Transfer 与第 4 类的跨本体方案（软提示、latent action）是并行解法 | 评测：多本体真机（ALOHA、Apptronik Apollo、双臂 Franka）+ 具身推理 benchmark（ERQA 等） | **部分开放**：Gemini Robotics-ER 1.5 通过 API 提供；VLA 权重**未开源**。主页：https://deepmind.google/models/gemini-robotics/ ；arXiv: https://arxiv.org/abs/2510.03342 | **不可复现**（权重与数据均不公开，评测 setup 为内部真机）；性能主张全部为厂商自评；"Motion Transfer"的机制细节披露有限，**待核验**；API 版本仅提供 ER（推理），不提供动作输出 |

> **未入表但需跟踪的同类工作**：**Hi Robot**（Physical Intelligence，2025，arXiv 2502.19417，分层交互式指令跟随，支持人类中途插话纠正）；**CoT-VLA**（NVIDIA/Stanford，CVPR 2025，用生成的**子目标图像**作为视觉推理链）；**OneTwoVLA**（2025，自适应决定何时推理何时执行）；**RoboDual**（2024，通用+专用双系统）；**Figure Helix**（2025，商业系统，7–9 Hz System 2 + 200 Hz System 1，**仅有博客、无论文，性能待核验**）；**OpenHelix**（开源双系统复现）。这一子方向 2025–2026 论文极多但**同质化严重、真机证据普遍薄弱**，收录需谨慎。

## 交叉标签

- **SayCan** → 交叉：`affordance（价值函数即可供性）`、`失败检测`
- **Code as Policies** → 交叉：`开放词表感知 API`、`agent / tool use`
- **VoxPoser** → 交叉：`第 6 类 2D-to-3D embodied grounding`、`零样本、无需机器人数据`
- **ECoT** → 交叉：`动作解码-token`、`可解释性与在线纠错（第 11 类）`、`自动标注管线`
- **π0.5** → 交叉：`跨本体共训（第 4 类）`、`flow matching（第 2 类）`、`openpi（第 8 类）`
- **Gemini Robotics 1.5** → 交叉：`跨本体 Motion Transfer`、`具身推理 benchmark`、`安全（ASIMOV，第 12 类）`

## 高低层接口形式对照（选型参考）

| 接口形式 | 代表 | 表达力 | 对感知的依赖 | 闭环能力 | 是否需要机器人数据 |
|---|---|---|---|---|---|
| 技能名 + 可供性分数 | SayCan | 低（受技能库限制） | 中 | 弱（开始时规划一次） | 需要（每技能一个策略） |
| 可执行代码 | Code as Policies | 高（含控制流） | **极高**（依赖 API 正确性） | 中（可写反馈循环） | 不需要（调用现成原语） |
| 3D 价值图 | VoxPoser | 中高（几何/空间约束强） | **极高**（依赖 RGB-D 与检测） | 中 | 不需要 |
| 自然语言子任务 | π0.5, Hi Robot | 中（语义粒度） | 低（端到端） | 强（每 chunk 可重判） | 需要（大规模） |
| 内嵌推理链 | ECoT, Gemini Robotics 1.5 | 高 | 低 | 强（可改推理纠正） | 需要（含推理标注） |
