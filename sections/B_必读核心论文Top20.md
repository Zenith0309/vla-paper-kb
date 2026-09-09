# B. 必读核心论文 Top 20（按优先阅读顺序）

> **排序原则**：不是按重要性排，而是按"**读了它之后，下一篇才读得懂**"的依赖关系排。
> **覆盖校验**：模仿学习基础 3 篇（#1–3）；VLA 主线 5 篇（#4–8）；跨本体或大规模数据 4 篇（#9–12）；benchmark / 开源生态 3 篇（#13–15）；grounding / affordance / referring 5 篇（#16–20）。合计 20 篇，满足全部下限要求。

| 优先级 | 论文 | 推荐原因 | 建议先读的章节 | 与后续研究的关系 |
|---|---|---|---|---|
| 1 | **ACT / ALOHA**（Zhao et al., RSS 2023）[第 3 类] | 只有理解了 action chunking 与 temporal ensembling 为什么有效，才能看懂之后每一篇 VLA 的动作头设计——它们全都默认使用这两个机制 | §3 Method（CVAE 结构与 chunk 定义）、§4.3 消融（chunk size 与 temporal ensembling 的独立贡献） | 是 π0、GR00T N1、OpenVLA-OFT 动作头的共同前提；其硬件是 LeRobot、Mobile ALOHA 的基础 |
| 2 | **Diffusion Policy**（Chi et al., RSS 2023 / IJRR 2024）[第 3 类] | 当代所有 diffusion / flow action expert 的数学与实现来源；不读它，π0 的 flow matching 与 CogACT 的 DiT 头都会读成"黑盒" | §3（问题形式化：为什么在动作序列上去噪而不是在单步动作上）、§4（receding horizon 与视觉条件化的三个设计选择） | 直接派生 DP3、CogACT、Dita；被 π0 用 flow matching 替换以减少推理步数；是第 11 类 consistency policy 的起点 |
| 3 | **RoboMimic / What Matters in Learning from Offline Human Demonstrations**（Mandlekar et al., CoRL 2021）[第 3 类] | 唯一系统回答"人类演示数据的哪些属性决定成败"的工作；它的结论（数据质量 > 算法、离线 RL 在人类数据上失效）解释了此后整个领域为什么走模仿而非 RL | §5 全部（五个实验问题的结论）、附录中的数据集统计 | LIBERO 直接建构于其代码之上；其"数据质量优先"的论断在 DROID、Data Scaling Laws、AgiBot World 中被反复验证 |
| 4 | **RT-2**（Brohan et al., CoRL 2023）[第 1 类] | 定义了 VLA 这个问题本身；读它是为了理解"为什么要把动作写成 token"以及"web knowledge transfer"的论证方式 | §3（动作 token 化与 co-fine-tuning）、§5 emergent capabilities（这是全文最有价值的部分） | 被 OpenVLA 开源复现；其离散 token 表示被 FAST 改进、被 π0 替代；其评测方式（emergent 能力分类）被后续广泛模仿 |
| 5 | **OpenVLA**（Kim et al., CoRL 2024）[第 1 类] | 社区事实标准基线，也是唯一能从零跑通的 7B VLA 训练栈；读论文的同时应把代码跑起来 | §4（模型与训练配方：视觉编码器、动作 token 映射、OXE 数据混合权重）、§6（LoRA 与量化微调） | 是 OpenVLA-OFT、SpatialVLA、UniVLA、几乎所有 RL post-training 工作的 base policy；其 OXE 混合权重是很多论文默认沿用的 |
| 6 | **OpenVLA-OFT / Fine-Tuning VLAs**（Kim et al., CoRL 2025，待核验）[第 2 类] | 最有效率的"设计空间地图"：一篇文章讲清并行解码、chunk、连续回归三者各自的贡献，并给出反直觉结论（L1 回归可以打败 diffusion） | §4 全部消融表（这是全文核心）、§5 延迟与吞吐分析 | 改变了后续工作的默认设置；是 SimpleVLA-RL、RIPT-VLA 的 base policy；读完它再回看 CogACT/Dita 的 diffusion 主张会更有判断力 |
| 7 | **π0**（Black et al., 2024，venue 待核验）[第 1 类] | 第一个真正做到高频灵巧长时程任务的 VLA；flow matching action expert 已成为工业主流架构 | §3（VLM 骨干 + action expert 的注意力共享方式）、§4（flow matching 训练目标与推理步数）、附录中的数据混合说明 | 其架构被 GR00T N1、SmolVLA、X-VLA 继承；其开源实现 openpi 是最实用的真机微调起点；π0-FAST 与 π0.5 是它的两个分支 |
| 8 | **π0.5**（Physical Intelligence, 2025，venue 待核验）[第 10 类] | 目前最有说服力的开放世界长程操作证据（未见过的真实住宅、十分钟量级任务）；同时展示"高层子任务 + 低层动作"如何合并进一个模型 | §3（异构数据共训的数据构成表——这是全文信息量最大的地方）、§4（两阶段推理）、§6 泛化实验 | 是第 10 类分层范式的现代标准；π*0.6 在其上做 RL 后训练；其数据配方是"数据混合"研究的重要参考 |
| 9 | **Open X-Embodiment / RT-X**（OXE Collaboration, ICRA 2024 Best Paper）[第 9 类] | 整个领域数据侧的分水岭；不读它就无法理解"为什么大家都在同一批数据上训练"以及"跨本体正迁移"这一前提 | §3（数据集构成表，重点看各子集的本体、动作空间与语言标注方式）、§5（RT-1-X/RT-2-X 的正迁移证据与失败案例） | 是 Octo、OpenVLA、π0、RDT、UniVLA 的预训练数据；其异构性是第 4 类所有对齐方案的问题来源 |
| 10 | **DROID**（Khazatsky et al., RSS 2024）[第 9 类] | 唯一大规模且硬件同构的真机数据集；是把模型部署到自己 Franka 上时最现实的起点 | §3（硬件与采集协议——若要自建采集流程，这一节是模板）、§5（加入 DROID 共训的收益与失败分析） | π0/π0.5/openpi 的标准微调路径；DROID 评测协议正在成为真机对比的少数共识之一 |
| 11 | **Data Scaling Laws in Imitation Learning for Robotic Manipulation**（Lin et al., ICLR 2025，待核验）[第 9 类] | 唯一直接回答"该采什么数据"的受控实验；结论（多样性 > 单场景演示数）会直接改变你的实验设计 | §4 全部（三组受控实验）、§5 的实践配方（"一天采集 = 新环境近零样本"） | 指导 UMI/AgiBot World 的采集策略；与 HPT 的模型侧 scaling 构成互补的两个维度 |
| 12 | **UniVLA**（Bu et al., RSS 2025，待核验）[第 4 类] | 跨本体路线中效率论证最清晰的一篇（1/20 算力、1/10 下游数据超过 OpenVLA）；同时是 latent action 这条线的当前最佳表述 | §3（任务中心 latent action 的分解方式——如何用语言把任务无关运动过滤掉）、§5 跨本体迁移实验 | 是 LAPA 的改进、AgiBot GO-1 的技术近亲；若你要用人类视频做预训练，这是首选参考 |
| 13 | **LIBERO**（Liu et al., NeurIPS 2023 D&B）[第 7 类] | 你的每一个实验都会在它上面报数字；必须理解四个套件各自检验什么、以及它已经饱和这一事实 | §3（四套件的构造逻辑）、§5（终身学习协议——很多人只用它的四套件而忽略了这部分） | 几乎所有 VLA 方法论文的主表；LIBERO-Plus 在其上加扰动；引用他人数字前需核对分辨率与状态输入设置 |
| 14 | **SimplerEnv**（Li et al., CoRL 2024）[第 7 类] | 唯一系统研究"仿真评测能否预测真机排名"的工作；它教你如何判断一个仿真数字值不值得相信 | §3（视觉匹配与系统辨识的做法）、§5（sim-real 排名相关性的定量结果与失效情形） | 让没有对应真机的团队也能做可信对比；其 real-to-sim 思路是构建自有评测环境的模板 |
| 15 | **LeRobot / SmolVLA**（Hugging Face, 2025，venue 待核验）[第 8 类] | 最短的"从零到真机"路径；SmolVLA 同时证明 450M 模型可用，对算力有限的个人研究极其重要 | SmolVLA §3（架构：跳层视觉特征 + flow matching 专家）、§4（异步推理栈）、LeRobot 文档中的 LeRobotDataset 格式说明 | 决定你的数据格式选型（LeRobotDataset vs RLDS）；异步推理栈与 RTC 是部署环节的两块拼图 |
| 16 | **Grounding DINO**（Liu et al., ECCV 2024）[第 6 类] | 你的流程中"高召回候选生成"这一步的默认工具；理解它的融合结构才知道它为什么召回高但消歧弱 | §3（三处跨模态融合的位置与作用）、§4.3（REC 任务上的表现与失败模式） | 与 SAM 组成 Grounded-SAM；在你的"候选生成 → 候选验证"流程中充当第一级，其召回率上限决定整个流程的上限 |
| 17 | **Segment Anything (SAM)**（Kirillov et al., ICCV 2023）[第 6 类] | box-to-mask 的通用后端；也是所有 mask 伪标签生成流程的基础设施 | §2–3（可提示分割任务定义与 mask 解码器的歧义处理——**多 mask 输出与置信度这一节对伪标签过滤最关键**）、§5 零样本迁移 | 被 LISA 复用为 mask 解码器；其"输出多个候选 mask + IoU 预测"的机制是你做候选验证时可直接利用的信号 |
| 18 | **LISA**（Lai et al., CVPR 2024 Highlight）[第 6 类] | 把 LLM 推理接到像素级输出的标准做法（embedding-as-mask）；你若要做"用全句一致性验证候选"，这是最直接的实现范式 | §3（`<SEG>` token 与 SAM 解码器的接法）、§4（ReasonSeg 构造方式）、§5 消融（LLM 规模对推理分割的影响） | 后续 GLaMM / GSVA / PixelLM 全部沿用其接口；其局限（单目标、无拒绝）正是 GRES 要解决的问题 |
| 19 | **RoboPoint**（Yuan et al., CoRL 2024）[第 6 类] | "点作为 VLM 与机器人之间的接口"的确立者；是你把 grounding 结果交给机器人执行的最短路径 | §3（仿真自动生成指令-关键点数据的管线——这一节可直接复用到你的数据构造）、§5 真机实验（放置任务中的自由空间点预测） | 被 Molmo/MolmoAct、RoboRefer 等继承；与 SpatialVLA（把空间直接编进 action head）构成两种 grounding-to-action 的路线 |
| 20 | **VoxPoser**（Huang et al., CoRL 2023）[第 10 类，交叉标签 grounding] | 完全零样本（不需要任何机器人数据）地把语言变成 3D 空间约束再变成轨迹；是"grounding → 动作"链路最完整、最可读的实现 | §3（价值图与约束图的合成方式）、§4（轨迹优化如何消费价值图）、§5 失败案例分析 | 是 Code as Policies 的后继、ReKep 等关系关键点工作的前身；在你的流程中可作为"拿到 mask 之后怎么变成动作"的参考实现 |

---

## 阅读节奏建议

| 阶段 | 论文 | 目标 | 预计投入 |
|---|---|---|---|
| 第 1 周 | #1–3 | 建立动作头与数据的基本直觉；把 Diffusion Policy 或 ACT 在 LIBERO 上跑通一次 | 3 篇精读 + 1 次复现 |
| 第 2–3 周 | #4–8 | 理解 VLA 主线的四次转折；把 OpenVLA/OpenVLA-OFT 在 LIBERO 上跑通 | 5 篇精读 + 1 次复现 |
| 第 4 周 | #9–12 | 建立数据侧判断力；决定自己的数据来源与采集策略 | 4 篇泛读 + #11 精读 |
| 第 5 周 | #13–15 | 确定评测协议与工程栈；此时应能判断他人论文数字是否可信 | 3 篇泛读 + 环境搭建 |
| 第 6–8 周 | #16–20 | 主攻方向精读；此阶段应同步开始构建"候选生成 → 验证 → mask"原型 | 5 篇精读 + 原型 |

## 三条使用提醒

1. **先跑通再精读**：#1、#2、#5、#6 都有可运行代码，先复现一次再回头读消融表，理解会深一个量级。
2. **带着评测清单读**：每读一篇涉及成功率的论文，用第 7 类末尾的"引用 benchmark 数字时的检查清单"过一遍，很多惊人的数字会显著缩水。
3. **本表不含 2026 年工作**：Top 20 全部是有充分复现与引用证据的工作。2026 年的候选代表作（StarVLA、SimpleVLA-RL、π*0.6、RoboRefer 等）应在核实元信息后作为"跟踪列表"单独阅读，不建议在打基础阶段投入。
