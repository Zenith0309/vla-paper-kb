# 9. 数据集、数据采集与 Scaling

本分块解决的核心问题是：**机器人领域没有互联网规模的"天然"动作数据，如何以可承受的成本获得足够多样、足够高质量的轨迹，以及数据的哪些维度真正决定泛化。** 采集方式分五类，成本与质量差异极大：**遥操作**（最高质量、最贵：DROID、AgiBot World）、**手持采集器**（去掉机器人本体，成本降一个量级：UMI）、**跨机构数据聚合**（零采集成本、但异构：Open X-Embodiment）、**人类视频**（几乎免费、但无动作标签：Ego4D + LAPA/UniVLA）、**合成数据**（仿真或视频生成：RoboCasa、RoboTwin 2.0、DreamGen）。

已被反复验证的核心结论（本分块最重要的知识）：**多样性（物体数 × 场景数）比单一场景下的演示数量重要得多**——Data Scaling Laws 论文的结论是"每个场景/物体只需相对少量演示，但要覆盖尽可能多的场景与物体"，且**泛化能力随场景与物体数量呈幂律提升**。这直接推翻了"在目标场景里疯狂采数据"的直觉做法。

未解决的瓶颈：（1）**数据质量没有可计算的度量**——什么是"好演示"仍靠人工抽检；（2）**数据配比全靠试**——OXE 各子集权重、真机/仿真/人类视频的比例缺乏原则；（3）**语言标注贫乏**——绝大多数数据集的语言是模板化任务名，几乎没有指代性、关系性表达（这是第 6 类方向缺训练数据的根源）；（4）**状态转移级标注缺失**——很少有数据集标注"这一步在做什么子动作/接触发生在何时"。

**技术演进链：**

`BridgeData V2（单一实验室的多样场景）→ Open X-Embodiment / RT-X（跨 22 机构的数据联合与正迁移证明）→ DROID（分布式采集的"in-the-wild"多样性）→ UMI（去机器人化的手持采集，把成本降一个量级）→ Data Scaling Laws（回答"该采什么"而非"采多少"）→ AgiBot World（单机构百万级高质量长程数据 + 配套基座模型）`

- **BridgeData V2 → OXE**：从"一个实验室的多样性"扩展为"**22 个机构、22 种本体、100 万+ 轨迹的联合数据集**"，并用 RT-1-X/RT-2-X 证明合并训练带来正迁移。
- **OXE → DROID**：解决 OXE 的异构性问题——**统一硬件平台（Franka + 标准相机架）在 13 个机构、564 个场景**采集，得到"同构但场景极其多样"的数据。
- **DROID → UMI**：把采集设备从"机器人 + 遥操作"换成**手持夹爪 + GoPro**，使数据可以在任意真实环境（野外、他人厨房）采集，并通过标定与延迟匹配迁移回机器人。
- **UMI → Data Scaling Laws**：从"怎么采"转向"**采什么才有效**"，用受控实验给出"环境数与物体数是主导因素、单场景演示数很快饱和"的幂律结论。
- **Data Scaling Laws → AgiBot World**：在上述认知指导下做**工业化采集**——标准化厂房、统一硬件、百万级轨迹、含长程与双臂任务，并配套 GO-1 基座模型验证数据规模收益。

## 论文表

| 年份 | 论文 / 数据集 | 作者 / 机构 | Venue | 论文类型 | 核心贡献 | 标签 | 归类理由 | 与同类工作的关系 / 演进 | 数据集或 benchmark | 开源情况与链接 | 注意事项 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 2023 | BridgeData V2: A Dataset for Robot Learning at Scale | Walke, Black, Zhao 等 / UC Berkeley | CoRL 2023 | 数据 | 6 万条轨迹、24 个环境、13 种技能的 WidowX 250 数据集，其中约 5 万条为遥操作、1 万条为脚本化；**每条轨迹带自然语言指令**，专门设计用于检验对新物体、新环境、新任务的泛化 | `奠基性论文` | 它是 VLA 时代**引用最广的单一实验室开源数据集**，WidowX 平台成为 SimplerEnv、OpenVLA、Octo、π0 等的标准真机/仿真评测 setup 之一。归类依据是作为"评测底座"的不可替代性 | 是 OXE 的重要组成部分；相对 RT-1 数据集是**开源版本**；被 DROID 在场景多样性与硬件统一性上超越；其对应的仿真环境见 SimplerEnv-WidowX | BridgeData V2（6 万条轨迹） | **完全开源**（数据 + 采集代码）。主页：https://rail-berkeley.github.io/bridgedata/ ；arXiv: https://arxiv.org/abs/2308.12952 | 单一低成本机械臂（WidowX），**精度与负载有限**，结论难以外推到工业机器人；语言标注为简短任务描述，**无指代性表达**；采集集中在少数几个实验室房间，视觉多样性弱于 DROID |
| 2023 | Open X-Embodiment: Robotic Learning Datasets and RT-X Models | Open X-Embodiment Collaboration（60+ 机构联合） | ICRA 2024（**Best Paper Award**） | 数据 + 方法 | 汇聚 **21–22 个机构、22 种本体、60+ 数据集、约 100 万条轨迹**为统一的 RLDS 格式数据集；并训练 RT-1-X / RT-2-X 证明**跨本体联合训练带来显著正迁移**（在参与机构的原任务上平均提升约 50%，且出现原数据中不存在的涌现技能） | `奠基性论文` | 它是整个 VLA 领域**数据侧的分水岭**：此后几乎所有通用策略（Octo、OpenVLA、π0、RDT、UniVLA）都以 OXE 为预训练数据。"跨本体正迁移"这一结论是第 4 类整个分块存在的经验基础。归类依据显而易见 | 建立在 RLDS（第 8 类）之上；被 DROID、AgiBot World 在单一数据集质量上超越，但在**多样性与社区共识**上不可替代；其 RT-1-X/RT-2-X 模型是第 4 类跨本体工作的起点 | Open X-Embodiment（约 100 万轨迹）；评测在各参与机构的真机上 | **完全开源**（数据 + 部分权重）。代码：https://github.com/google-deepmind/open_x_embodiment ；主页：https://robotics-transformer-x.github.io/ ；arXiv: https://arxiv.org/abs/2310.08864 | **异构性极强**：动作空间、控制频率、相机配置、语言标注质量差异巨大，直接全量训练效果差，**各家使用的子集与权重都不同且很少公开**，这是 VLA 论文数字不可比的主要来源之一；大量子集的语言标注是自动生成的模板；部分子集质量低（脚本化、重复度高） |
| 2024 | DROID: A Large-Scale In-the-Wild Robot Manipulation Dataset | Khazatsky, Pertsch, Nair 等 / 13 个机构联合（Stanford、UC Berkeley、TRI 等） | RSS 2024 | 数据 | 用**完全统一的硬件平台**（Franka Panda + 标准化相机与遥操作栈）在 13 个机构、**564 个场景、86 个任务、7.6 万条演示（350 小时）** 采集；核心设计是"硬件同构 + 场景极度异构"，并证明加入 DROID 共训可提升策略在新场景的成功率与鲁棒性 | `奠基性论文` | 它解决了 OXE 最大的痛点（异构导致难以利用），提供了**唯一大规模且硬件统一的真机数据集**；π0、π0.5、openpi 的 DROID 微调配方已成为标准入门路径。归类依据是"高质量同构大数据"这一空缺的填补 | 相对 OXE 是**质量与一致性的改进**（牺牲本体多样性换取可用性）；相对 BridgeData V2 是场景多样性的量级提升；与 UMI 是"标准化平台"vs"去平台化"的两条路 | DROID（7.6 万条演示，564 场景）；配套 DROID 真机评测协议 | **完全开源**（数据 + 硬件方案 + 代码）。主页：https://droid-dataset.github.io/ ；arXiv: https://arxiv.org/abs/2403.12945 | 单一本体（Franka + Robotiq），**不覆盖双臂、灵巧手、人形**；任务以短时程桌面操作为主，**长程任务少**；语言标注为众包简短描述，指代性表达稀少；数据规模仍远小于视觉/语言领域（350 小时 vs 互联网规模） |
| 2024 | Universal Manipulation Interface (UMI): In-The-Wild Robot Teaching Without In-The-Wild Robots | Chi, Xu, Pan 等 / Stanford, Columbia, TRI | RSS 2024 | 数据 + 系统 | 提出**手持式数据采集器**（平行夹爪 + GoPro 广角 + 侧镜 + IMU），使人可以在任意真实环境中直接演示技能，无需机器人在场；配套解决了**延迟匹配、相对轨迹表示、推理时的观测对齐**三个关键 sim-to-robot 问题，使采集的数据可直接训练可部署的 diffusion policy。展示了动态抛接、双臂折叠、洗碗等技能的零样本环境迁移 | `奠基性论文` | 它把"采集成本"这一根本约束**降低了一个数量级**，并催生了一大批手持/穿戴式采集工作（UMI-on-Legs、FastUMI、各类外骨骼采集器）。归类依据是采集范式的开创性 | 相对 DROID 的"标准化机器人平台"是**替代路线**（去机器人化）；与人类视频（Ego4D）路线的差别是**保留了精确的夹爪位姿与开合量**，因此有真实动作标签；其相对轨迹表示被后续多个跨本体工作借鉴 | 自采 UMI 数据（多环境、多技能）；评测为真机 UR5/ARX | **完全开源**（硬件 BOM + 代码 + 数据）。代码：https://github.com/real-stanford/universal_manipulation_interface ；主页：https://umi-gripper.github.io/ ；arXiv: https://arxiv.org/abs/2402.10329 | 只适用于**平行夹爪**形态，灵巧手/吸盘/工具无法采集；依赖 SLAM 定位，**纹理稀疏或强反光环境下位姿漂移**；无本体力/力矩信息；人手动作的动力学与机器人不同，高速动作迁移会失真 |
| 2024 | Data Scaling Laws in Imitation Learning for Robotic Manipulation | Lin, Hu, Li 等 / 清华大学（叉院 / MARS Lab 等） | ICLR 2025（**待核验**） | 数据 + 方法 | 用 UMI 采集了覆盖 32 个物体、上百个环境的受控数据，系统研究**演示数量 / 物体数量 / 环境数量**三者对泛化的独立影响。核心结论：**策略对新物体与新环境的泛化能力随"训练物体数"与"训练环境数"呈幂律提升，而单个环境-物体组合内的演示数很快饱和**；据此给出"用一天时间采集即可获得对任意新环境近乎零样本部署"的实践配方 | `当前代表作` | 这是本领域**唯一被广泛引用的、关于"该采什么数据"的受控实验研究**，直接改变了很多团队的采集策略（从"深挖单场景"改为"广铺多场景"）。归类依据是对实践的直接指导价值 | 建立在 UMI 采集范式之上；与 HPT（第 4 类）的"模型/数据集数量 scaling"是互补的两个维度；被 AgiBot World、RoboTwin 2.0 等在采集与生成策略上引用 | 自采 UMI 数据（32 物体 × 上百环境）；真机评测（倒水、鼠标归位、折叠、拔插） | **开源**（数据 + 代码）。代码：https://github.com/Fanqi-Lin/Data-Scaling-Laws ；主页：https://data-scaling-laws.github.io/ ；arXiv: https://arxiv.org/abs/2410.18647 | venue **待核验**；结论基于 **4 个相对简单的准静态任务**，是否外推到长程、高精度、接触丰富任务**未验证**；使用 UMI 采集，因此结论可能与遥操作数据的分布特性绑定；幂律的指数在不同任务上差异大 |
| 2025 | AgiBot World Colosseo: A Large-scale Manipulation Platform for Scalable and Intelligent Embodied Systems | AgiBot-World Team（智元机器人 / 上海 AI Lab / OpenDriveLab 等） | arXiv → **IROS 2025 Best Paper Award Finalist**；据检索另有 IEEE T-RO 2026 版本（**待核验**） | 数据 + 系统 + 方法 | 工业化采集平台与大规模数据集：标准化数据采集工厂、统一的 AgiBot G1 双臂本体（含灵巧手与视触觉），发布**百万级轨迹**（检索显示最新版本约 100 万条轨迹 / 约 2976 小时 / 217 任务 / 106 场景），覆盖长程、双臂协作、接触丰富任务；配套 **GO-1** 基座模型（latent action + VLA），并报告随演示数量从 10 万增至 100 万的性能提升曲线 | `当前代表作` | 它是目前**规模最大、任务复杂度最高的开源真机数据集之一**，且首次系统展示"长程双臂高质量数据的规模收益"；已成为中文机器人社区的主要数据来源与基线。归类依据是数据规模、开放性与配套模型的验证 | 相对 DROID 的"多机构同构短任务"是**长程与双臂维度的补充**；相对 OXE 的异构聚合是**单一高质量来源**；其 latent action 思路继承 LAPA/UniVLA（第 1、4 类）；与 Genie Envisioner（第 5 类）共享同一数据底座 | AgiBot World（约 100 万轨迹）；配套 AgiBot World Challenge 评测 | **开源**（数据 + GO-1 相关代码）。代码：https://github.com/OpenDriveLab/AgiBot-World ；主页：https://agibot-world.com/ ；arXiv: https://arxiv.org/abs/2503.06669 | **单一本体（AgiBot G1）**，跨本体迁移证据有限；数据采自**标准化厂房环境**，家庭/野外场景多样性不如 DROID；版本迭代快（初版约 100 万帧级到最新百万轨迹级），**引用时必须注明具体版本与统计口径**；GO-1 的性能数字为作者自评 |

> **其他重要数据集（未单独入表）**：**RT-1 dataset**（Google，13 万条，已并入 OXE，见第 2 类 RT-1）；**RoboMIND**（多本体规范化数据，arXiv 2412.13877，**venue 待核验**）；**RH20T**（上海交大，多技能大规模数据，SpatialVLA 的预训练来源之一）；**Ego4D / Something-Something V2**（人类视频，LAPA、GR-1、UniVLA 的预训练来源）；**RoboSet / RoboAgent**（CMU/Meta，多任务活动数据）；**BC-Z dataset**。这些在特定用途上重要，但作为"通用预训练底座"的共识度低于表中六项。

## 交叉标签

- **BridgeData V2** → 交叉：`SimplerEnv-WidowX 评测底座`、`Octo/OpenVLA 训练数据`
- **Open X-Embodiment** → 交叉：`跨本体（RT-1-X / RT-2-X）`、`RLDS 格式`、`几乎所有 VLA 的预训练数据`
- **DROID** → 交叉：`π0/π0.5 微调标准路径`、`真机评测协议`
- **UMI** → 交叉：`Diffusion Policy 部署`、`Data Scaling Laws 的采集工具`
- **Data Scaling Laws** → 交叉：`采集策略`、`泛化研究`
- **AgiBot World** → 交叉：`latent action（GO-1）`、`Genie Envisioner 数据底座`、`双臂长程`

## 采集方式成本-质量对照

| 方式 | 代表 | 相对成本 | 动作标签质量 | 场景多样性上限 | 主要局限 |
|---|---|---|---|---|---|
| 遥操作（标准平台） | DROID、AgiBot World | 高 | 最高（含 proprioception） | 受限于机器人可到达的场所 | 贵、慢 |
| 手持采集器 | UMI | 中 | 高（夹爪位姿+开合） | 几乎无限（任何真实环境） | 仅平行夹爪、无力信息 |
| 跨机构聚合 | Open X-Embodiment | 极低（复用） | 参差 | 高 | 异构、配比难调 |
| 人类视频 | Ego4D + LAPA/UniVLA | 极低 | 无（需 latent/IDM 反推） | 最高 | 伪动作噪声大 |
| 仿真 / 生成 | RoboCasa、RoboTwin 2.0、DreamGen | 低（算力） | 精确但非真实分布 | 受资产库限制 | sim-real gap |

## 对本库研究方向（referring/grounding）的特别提示

上述所有数据集的**语言标注都是"任务级"而非"指代级"**（例："pick up the cup"，而不是"pick up the leftmost red cup behind the bowl"）。这意味着：

1. 直接在这些数据上训练的 VLA **没有机会学习指代消歧**，其语言能力上限由 VLM 预训练决定。
2. 若要研究 grounding 驱动的操作，需要**自己构造带关系表达的指令标注**，或借用第 6 类的 RoboSpatial / RefSpatial 这类 QA 数据做联合训练。
3. **状态转移级标注（state-transition annotation）几乎完全缺失**——没有数据集标注"哪一帧发生了接触""哪一段是接近、哪一段是抓取"，这限制了子目标级监督与失败检测的研究。这是一个明确的数据侧空白。
