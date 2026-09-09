# 6. Visual grounding / affordance / referring（重点分块）

> 本分块是本知识库的核心方向。按用户研究目标细分为 7 个子类，每篇论文在表格前标注其所属子类。

本分块解决的核心问题是：**把一句自然语言中的"那个东西"，稳定、可验证地对应到图像中的唯一像素区域，并进一步对应到机器人可以执行的物理动作上。** 技术路线经历了三次转折：（1）**从两阶段到端到端**——早期用检测器出候选框再打分，MDETR 改为端到端调制检测；（2）**从闭集到开放词表**——GLIP 把检测重述为 grounding 预训练，Grounding DINO 与 SAM 让"任意短语 → 框 → mask"成为可用组件；（3）**从描述到指称再到推理**——LISA/GRES 把任务推进到需要推理与"可能无目标/多目标"的现实设定，RoboSpatial/RoboRefer 进一步引入**空间关系与多步推理**。

当前最突出的瓶颈恰好落在机器人场景：（a）**关系与同类实例消歧**——"最左边那个杯子""被书挡住的那个"仍显著失败，主因是训练数据中关系表达稀疏且标注以"物体类别"为主；（b）**视角依赖的空间语义**——"左边"是相对相机、相对物体还是相对机器人，现有模型经常混淆；（c）**候选验证缺乏显式机制**——多数模型是"一步出答案"，没有"生成候选 → 用属性/关系逐条验证 → 拒绝"的显式流程，因此无法输出可靠的置信度，也无法在无目标时说"不存在"；（d）**从 mask 到动作的最后一公里**——即使 mask 正确，抓哪里、以什么位姿抓仍是另一个问题。

**技术演进链（主线）：**

`MDETR → GLIP → Grounding DINO + SAM → LISA → GRES/gRefCOCO → RoboSpatial → RoboRefer → RoboPoint → SpatialVLA`

- **MDETR → GLIP**：从"在给定图文对上做端到端调制检测"扩展为"**把目标检测重述为短语 grounding 并做大规模预训练**"，从而获得开放词表能力与零样本迁移。
- **GLIP → Grounding DINO**：把 grounding 预训练与强检测架构（DINO）结合，并在**多个层级做跨模态融合**（neck / query 初始化 / decoder），使"任意文本 → 高召回候选框"成为工程上可靠的组件。
- **Grounding DINO → SAM**：分工确立——**检测器负责"是哪个"，SAM 负责"精确到哪些像素"**，box-to-mask 成为标准流水线（Grounded-SAM）。
- **SAM → LISA**：引入 **LLM 的推理能力**，用 `<SEG>` embedding 作为 SAM 的提示，使"给我能用来钉钉子的东西"这类需要推理与世界知识的指令可分割。
- **LISA → GRES**：修正评测设定本身——真实指令中**可能没有目标、也可能指向多个目标**，提出 gRefCOCO 与 ReLA 模型，为"候选验证与拒绝"提供了评测土壤。
- **GRES → RoboSpatial**：把问题从"指称哪个物体"推进到"**空间关系是否成立**"，并明确区分 ego-centric / object-centric / world-centric 三种参考系。
- **RoboSpatial → RoboRefer**：加入**深度编码器与多步推理 + 强化微调**，把空间指称从单步判断变成可解释的多步推理过程。
- **RoboRefer → RoboPoint**：输出形式从"框/mask"变为"**可操作的 2D 关键点**"（包括自由空间中的放置点），直接对接机器人动作。
- **RoboPoint → SpatialVLA**：把空间表征**内嵌进 VLA 本身**（Ego3D 位置编码 + 自适应空间动作栅格），让 grounding 直接影响 action head 的输出空间。

---

## 论文表

| 子类 | 年份 | 论文 | 作者 / 机构 | Venue | 论文类型 | 核心贡献 | 标签 | 归类理由 | 与同类工作的关系 / 演进 | 数据集或 benchmark | 开源情况与链接 | 注意事项 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| REC / phrase grounding | 2021 | MDETR: Modulated Detection for End-to-End Multi-Modal Understanding | Kamath, Singh, LeCun, Misra, Carion, Synnaeve / NYU, Meta AI | ICCV 2021 | 方法 | 首个**端到端文本调制检测器**：把 DETR 的目标 query 与文本 token 在编码器内早期融合，用软 token 预测（每个框预测它对应文本中的哪些 token）+ 对比对齐损失训练，一次前向同时完成检测与短语对齐。取消了"先检测再打分"的两阶段范式 | `奠基性论文` | 它是现代 grounding 模型的结构原型：**跨模态早期融合 + 框-短语软对齐**这一组合被 GLIP、Grounding DINO、几乎所有后续工作继承。归类为奠基基于结构范式的开创性 | 是 GLIP 的直接前身（GLIP 把它 scale 到检测预训练）；相对两阶段 MAttNet/ViLBERT 类方法是**替代**；其"软 token 对齐"是后来 phrase-to-region 监督的标准形式 | Flickr30k Entities、RefCOCO/+/g、GQA、CLEVR-Ref+、PhraseCut | **开源**。代码：https://github.com/ashkamath/mdetr ；arXiv: https://arxiv.org/abs/2104.12763 | **非 VLA 直接工作**；训练依赖密集的框-短语对齐标注，成本高；对**长句中的复杂空间关系**处理弱（这是它与后续 RoboSpatial 类工作的主要差距）；输出为框，无 mask |
| 开放词表 grounding | 2021 | Grounded Language-Image Pre-training (GLIP) | Li, Zhang, Zhang 等 / UCLA, Microsoft | CVPR 2022（Oral） | 方法 | 提出**把目标检测统一重述为短语 grounding**，从而可以同时用检测数据（框标注）与海量图文对（弱监督、自训练生成伪框）做统一预训练，得到语义丰富的开放词表检测能力。在 COCO 上零样本即达到有监督基线水平 | `奠基性论文` | "检测 = grounding"这一重述是开放词表感知的关键概念转折，直接催生了 GLIPv2、Grounding DINO、以及后续所有以文本为类别提示的检测器。归类依据是概念层面的转折意义 | 是 MDETR 的**规模化改写**（从任务模型变为预训练范式）；被 Grounding DINO 在架构与性能上超越；与 CLIP 的差别是**区域级而非图像级对齐** | 预训练：Objects365 + GoldG + Cap4M/24M；评测：COCO、LVIS、ODinW（13/35 任务） | **开源**。代码：https://github.com/microsoft/GLIP ；arXiv: https://arxiv.org/abs/2112.03857 | **非 VLA 直接工作**；自训练伪框带来标注噪声，长尾类别可靠性差；对"关系"与"指代唯一性"无建模——它擅长"找出所有杯子"，不擅长"找出最左边那个杯子" |
| 开放词表 grounding / 候选生成 | 2023 | Grounding DINO: Marrying DINO with Grounded Pre-Training for Open-Set Object Detection | Liu, Zeng, Ren 等 / IDEA Research | ECCV 2024 | 方法 | 在 DINO 检测器的**三个层级**（特征增强 neck、语言引导的 query 初始化、跨模态解码器）做紧密的图文融合，实现任意文本提示下的高质量开放集检测；支持 REC 输入。是当前最常用的"**语言 → 高召回候选框**"工业级组件 | `当前代表作` | 它是"候选生成"这一步的事实标准工具：Grounded-SAM、大量机器人 grounding 流水线、以及自动标注管线都以它为第一级。归类依据是工程复用度与被集成频率，而非单纯的 benchmark 数字 | 是 GLIP 的**直接改进**（更强检测骨干 + 更深融合）；与 SAM 组合成 Grounded-SAM（box-to-mask）；被 Grounding DINO 1.5/2.0、T-Rex 等后续商用版本迭代 | 预训练：COCO、O365、GoldG、Cap4M、OpenImages、RefC；评测：COCO、LVIS、ODinW、RefCOCO/+/g | **开源**（Apache-2.0，权重可商用）。代码：https://github.com/IDEA-Research/GroundingDINO ；arXiv: https://arxiv.org/abs/2303.05499 | **非 VLA 直接工作**；对**关系型指代的判别力有限**——它常把满足类别的所有实例都高分召回，无法区分"左边那个"；文本编码器为 BERT，长句理解弱；**这恰恰使它非常适合作为"高召回候选生成器"，而把消歧交给后续验证模块**（本库总结 C 的核心设计依据） |
| phrase-to-mask / box-to-mask | 2023 | Segment Anything (SAM) | Kirillov, Mintun, Ravi 等 / Meta AI | ICCV 2023 | 方法 + 数据 | 提出可提示分割（promptable segmentation）任务、SAM 模型与 SA-1B 数据集（1100 万图、10 亿 mask）。给定点/框/粗 mask 提示即可输出高质量实例 mask，零样本迁移到大量分割任务。**在本方向的作用是"box → mask"的通用后端** | `奠基性论文` | 它把"精确 mask 生成"变成了可即插即用的基础设施，使 grounding 研究可以专注于"选对目标"而不必重训分割器。归类依据是基础设施级的普及度（几乎所有 phrase-to-mask 流水线都用它或其后继） | 与 Grounding DINO 组成标准两段式（Grounded-SAM）；被 SAM 2（ICLR 2025，加入视频与记忆机制）与后续可提示概念分割模型迭代；被 LISA 作为 mask 解码器复用 | SA-1B；零样本评测覆盖 23 个分割数据集 | **开源**（模型 Apache-2.0，SA-1B 有使用条款）。代码：https://github.com/facebookresearch/segment-anything ；arXiv: https://arxiv.org/abs/2304.02643 | **非 VLA 直接工作、且不理解语言**——SAM 本身没有语义，给错框就出错 mask；对**透明/反光物体与严重遮挡**分割质量下降；在机器人桌面场景中会过分割（把杯子和把手分开），需要后处理合并 |
| referring segmentation + 推理 | 2023 | LISA: Reasoning Segmentation via Large Language Model | Lai, Tian, Chen 等 / CUHK, SmartMore, MSRA 等 | CVPR 2024（Highlight） | 方法 + benchmark | 提出 **reasoning segmentation** 任务（指令是隐式的、需要世界知识与推理，如"图中富含维生素 C 的食物"），并提出 **embedding-as-mask** 范式：让多模态 LLM 输出一个 `<SEG>` token，其隐藏向量作为 SAM 解码器的提示，从而把 LLM 的推理能力接到像素级输出上。同时发布 ReasonSeg 评测集 | `当前代表作` | `<SEG>` token 这一接口设计被后续绝大多数"MLLM + 分割"工作沿用（GLaMM、PixelLM、GSVA、SAM4MLLM 等），是把语言推理引入像素级任务的标准做法。归类依据是接口设计的普及度 | 相对传统 RES（LAVT 等）是**能力扩展**（从显式短语到隐式推理指令）；相对 Grounded-SAM 的两段式是**端到端替代**；被 GLaMM（grounded conversation）、GSVA（多目标/空目标）扩展 | 训练：RefCOCO 系列 + ADE20K/COCO-Stuff + LLaVA 指令数据；评测：ReasonSeg、RefCOCO/+/g | **开源**。代码：https://github.com/dvlab-research/LISA ；arXiv: https://arxiv.org/abs/2308.00692 | **非 VLA 直接工作**；原始版本**只能输出单个目标 mask**（无法处理"所有杯子"或"没有匹配"）；推理链不显式，**无法给出可核查的置信度**；LLM 推理成本使其难以用于高频机器人闭环 |
| 广义 referring / 候选验证 | 2023 | GRES: Generalized Referring Expression Segmentation（含 gRefCOCO 数据集） | Liu, Ding, Jiang / 南洋理工大学 | CVPR 2023 | benchmark + 方法 | 指出经典 RES 的两个不现实假设：**每条表达恰好对应一个目标**。提出 GRES 任务（允许**多目标**与**无目标**表达）、gRefCOCO 数据集，以及 ReLA 模型（把图像划分为区域、显式建模区域间关系与区域-语言依赖，并预测"是否存在目标"） | `当前代表作` | 它把"**拒绝**"和"**多目标**"变成了一等公民，是评估候选验证/伪标签过滤能力的唯一主流公开设定。对本库的研究方向而言，它提供了"什么时候该说不存在"的评测土壤。归类依据是任务设定层面的必要性 | 与 GREC（广义 REC，框级版本）配套；相对 RefCOCO 系列是**设定修正**；LISA 系列的 GSVA 等后续工作直接针对 GRES 的空目标问题改进；是本库总结 C 中"候选验证—拒绝"环节的评测基础 | gRefCOCO（基于 RefCOCO 扩展的多目标/无目标标注）；同时报告 RefCOCO/+/g | **开源**（代码 + 数据集）。代码：https://github.com/henghuiding/ReLA ；主页：https://henghuiding.github.io/GRES/ ；arXiv: https://arxiv.org/abs/2306.00968 | **非 VLA 直接工作**；gRefCOCO 的无目标样本构造方式偏"人工替换名词"，与机器人场景中真实的"物体不在视野内"分布不同；ReLA 的区域划分是固定网格，对小物体不友好 |
| 关系感知 grounding | 2024 | RoboSpatial: Teaching Spatial Understanding to 2D and 3D Vision-Language Models for Robotics | Song, Blukis, Tremblay, Tyree, Su, Birchfield / 俄亥俄州立大学, NVIDIA | CVPR 2025（Oral） | 数据 + benchmark | 提供大规模室内/桌面**空间关系问答数据集**（100 万张图、300 万空间关系标注，含 3D 扫描），并明确区分三种参考系：**ego-centric（相对相机/观察者）、object-centric（相对物体自身朝向）、world-centric（相对世界坐标）**。任务覆盖空间构型判断（左右/前后/上下）、空间上下文（是否可放置）与空间兼容性 | `当前代表作` | 它是**第一个把"参考系歧义"作为核心问题正面处理**的机器人空间理解数据集，直击"左右/前后在谁的视角下"这一在指代消歧中最常见的错误来源。归类依据是问题定义的准确性与数据规模，且已被多项后续空间 VLM 工作用作训练/评测源 | 相对 RefCOCO 系列（关系表达稀疏且以 2D 图像坐标为准）是**能力补充**；被 RoboRefer 在"多步推理 + 深度输入"上扩展；与 SpatialVLM/BLINK 等空间推理工作是并行路线 | RoboSpatial（train / RoboSpatial-Home 评测集）；在 2D 与 3D VLM 上微调验证，并接入真机操作 | **开源**（数据 + 代码）。代码：https://github.com/NVlabs/RoboSpatial ；主页：https://chanh.ee/RoboSpatial/ ；arXiv: https://arxiv.org/abs/2411.16537 | 数据以**问答形式**为主，不直接产出 mask，需与分割模块组合；场景以室内桌面/家居为主，工业场景覆盖不足；**遮挡关系（"被挡住的那个"）覆盖有限**，这是本方向仍然的空白 |
| 关系感知 + 多步推理 | 2025 | RoboRefer: Towards Spatial Referring with Reasoning in Vision-Language Models for Robotics | Zhou, An 等 / 北京智源人工智能研究院（BAAI）、北京大学等（**作者列表待核验**） | arXiv（**venue 待核验**；检索中出现 2026 版本，需确认） | 方法 + 数据 + benchmark | 首个面向**多步空间指称**的 3D 感知 VLM：引入**解耦的专用深度编码器**做 SFT，再用**面向度量的过程奖励**做强化微调（RFT），使模型能显式输出多步（最多 5 步）空间推理过程。同时发布 RefSpatial（约 2000 万 QA，31 种空间关系）与 RefSpatial-Bench | `近期候选代表作` | 技术上把"关系感知 grounding"从单步判别推进到**可解释的多步推理 + 过程奖励**，与本库研究方向高度契合；但截至检索日 venue、第三方复现与独立评测均不足（检索中同时出现 2025 与 2026 两个版本号），故标为候选 | 是 RoboSpatial 的**直接后继**（更大数据、加入深度模态与推理链）；与 Molmo/RoboPoint 的差别是**输出推理过程而非仅坐标**；可作为本库"候选验证"环节的推理器组件 | RefSpatial（2000 万 QA）、RefSpatial-Bench；下游接入 UR5、Unitree G1 真机 | **开源**（代码 + 数据 + 权重）。主页：https://zhoues.github.io/RoboRefer/ ；arXiv: https://arxiv.org/abs/2506.04308 | **多项元信息待核验**（venue、版本、作者列表）；性能数字（如超越 Gemini-2.5-Pro 17.4%）为作者自评，**无第三方复现**；深度编码器依赖可靠深度输入，对反光/透明物体退化；真机任务数量有限 |
| affordance grounding（关键点） | 2024 | RoboPoint: A Vision-Language Model for Spatial Affordance Prediction for Robotics | Yuan, Duan, Blukis 等 / 华盛顿大学, NVIDIA, 艾伦研究所 | CoRL 2024 | 方法 + 数据 | 让 VLM 直接输出**图像空间中的 2D 关键点集合**（归一化坐标），既可指向物体上的可操作区域，也可指向**自由空间中的放置点**（"把它放在盘子和杯子之间"）。用仿真自动生成的指令-关键点数据微调，无需人工标注，输出可直接接入运动规划器 | `当前代表作` | 它确立了"**点（point）作为 VLM 与机器人之间的通用动作接口**"，比框/mask 更直接可执行，被 Molmo、MolmoAct、RoboRefer、以及多个 VLA 的高层模块采用为输出格式。归类依据是接口形式的广泛采纳 | 相对 Grounded-SAM 的框/mask 输出是**面向执行的改造**；与 Molmo/PixMo 的 pointing 能力是并行发展（Molmo 面向通用 VLM，RoboPoint 面向机器人 affordance）；被 MolmoAct 等 action reasoning 模型直接复用其数据配方 | 训练：仿真自动生成的空间 affordance 数据 + 通用 VQA 共训；评测：自建 RoboPoint benchmark + 真机（导航到点、抓取、放置） | **开源**（代码 + 数据 + 权重）。代码：https://github.com/wentaoyuan/RoboPoint ；主页：https://robo-point.github.io/ ；arXiv: https://arxiv.org/abs/2406.10721 | **点是稀疏表示**，不提供物体范围与朝向，对需要 6-DoF 位姿的抓取仍不充分（需接抓取网络）；论文自身与后续工作均指出其**难以刻画细粒度可操作区域边界**（如"把手上可握的一段"）；仿真生成的数据与真实场景存在外观差距 |
| 2D→3D / grounding 进入 action head | 2025 | SpatialVLA: Exploring Spatial Representations for Visual-Language-Action Model | Qu, Song, Chen 等 / 上海 AI Lab, 上海交大等 | RSS 2025（**待核验**） | 方法 | 在 VLA 内部显式引入空间表征：**Ego3D Position Encoding**（把单目深度估计得到的 3D 点位置编码注入视觉 token，使表征与相机内参解耦）+ **Adaptive Spatial Action Grids**（把连续动作离散到自适应的 3D 空间栅格上，栅格可按新本体重新标定）。**动作表示**：空间栅格离散 token；**输入**：RGB + 单目深度导出的 3D 位置编码 + 语言；**输出**：栅格化的末端平移/旋转 token | `当前代表作` | 它是"**grounding 直接进入 action head 的输出空间**"这一思路最清晰的实现——动作词表本身是空间栅格，因此空间理解与动作生成共享表示。对本库的研究路线（grounding → 动作）具有直接方法论价值 | 相对 OpenVLA 的一维分箱动作 token 是**空间化改造**；相对 RoboPoint 的"点 → 外部规划器"是**端到端替代**；与 MolmoAct（用深度 token + 视觉轨迹做中间表示）是并行路线 | 预训练：Open X-Embodiment + RH20T（约 110 万条）；评测：SimplerEnv（Google Robot / WidowX）、LIBERO、真机 Franka/WidowX | **开源**（代码 + 权重）。代码：https://github.com/SpatialVLA/SpatialVLA ；主页：https://spatialvla.github.io/ ；arXiv: https://arxiv.org/abs/2501.15830 | venue **待核验**；依赖单目深度估计，**深度误差直接转化为动作误差**；自适应栅格在跨本体时需重新标定，非零成本；对**语言中的关系消歧**没有专门处理（它解决的是"空间表征"而非"指代歧义"） |

---

## 子类覆盖检查表

| 用户要求的子类 | 本表覆盖 | 覆盖强度 |
|---|---|---|
| referring expression comprehension / segmentation | MDETR、LISA、GRES | 强 |
| open-vocabulary grounding / detection | GLIP、Grounding DINO | 强 |
| relation-aware grounding（左右/前后/最左/遮挡/同类消歧） | RoboSpatial、RoboRefer、GRES(ReLA 的区域关系建模) | 中——**遮挡与同类实例消歧仍是公开空白** |
| phrase-to-mask / box-to-mask / candidate verification | SAM（box→mask）、GRES（多目标/拒绝的评测设定）、Grounding DINO（候选生成） | 中——**"显式候选验证模块"尚无公认代表作，见下方空白说明** |
| affordance grounding（grasp/place/push/pull/tool use） | RoboPoint | 中——**仅覆盖关键点级；细粒度可操作区域与工具使用覆盖弱** |
| 2D-to-3D / embodied grounding | RoboRefer（深度编码器）、SpatialVLA（Ego3D 编码）、PerAct/DP3（见第 3 类） | 中 |
| uncertainty / confidence calibration / 伪标签过滤 | **本表未覆盖**——最相关工作 KnowNo（conformal prediction，主归属第 11 类） | 弱——**本方向最大空白之一** |
| grounding 进入 VLA 的目标选择 / 动作规划 / action head | SpatialVLA、RoboPoint | 中 |

## 交叉标签

- **MDETR / GLIP / Grounding DINO** → 交叉：`开放词表感知（非 VLA）`、`自动标注管线`
- **SAM** → 交叉：`基础设施`、`伪标签生成`
- **LISA** → 交叉：`MLLM 推理`、`System2 高层语义`
- **GRES** → 交叉：`benchmark`、`候选验证/拒绝`
- **RoboSpatial / RoboRefer** → 交叉：`数据集`、`benchmark`、`System2 空间推理`
- **RoboPoint** → 交叉：`长程规划的高层接口`、`仿真数据生成`
- **SpatialVLA** → 交叉：`基础架构`、`动作解码-空间栅格 token`、`跨本体`

## 本方向的公开空白（对应总结 C）

1. **没有一个公开的"候选生成 → 候选验证 → 拒绝"框架**：主流模型是一步出答案；PropVG 等 proposal-driven 工作（2025，**待核验**）方向对，但尚未形成公认基线。
2. **同类实例消歧缺乏专门评测**：RefCOCO 中"同类多实例 + 仅靠关系区分"的样本比例低，模型可以靠类别先验蒙对，导致评测高估真实能力。
3. **遮挡关系几乎无人处理**："被 X 挡住的那个 Y"需要 amodal 推理 + 关系推理，现有数据集覆盖极少。
4. **grounding 的置信度不可用**：分割/检测的分数不是校准概率，无法作为伪标签过滤阈值；KnowNo 的 conformal 思路尚未被移植到 mask 级。
5. **affordance 与 referring 是两套割裂的体系**：RoboPoint 类工作只回答"点哪里"，referring 类工作只回答"是哪个"，二者的联合评测缺失。
