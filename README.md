# VLA（Vision-Language-Action）领域论文知识库

> 检索基准日期：**2026-09-08**
> 建库日期：2026-09-09
> 目标：可持续更新、可检索的个人文献知识库（面向 referring / grounding 驱动的机器人操作研究方向）

---

## 0. 使用说明

### 0.1 目录结构

| 文件 | 内容 |
|---|---|
| [sections/01_VLM-VLA基础架构.md](sections/01_VLM-VLA基础架构.md) | 第 1 类：VLM backbone + action head 范式、多模态融合、MoE / action expert / latent action |
| [sections/02_动作解码范式.md](sections/02_动作解码范式.md) | 第 2 类：autoregressive token / continuous regression / diffusion / flow matching / chunking |
| [sections/03_模仿学习基础方法.md](sections/03_模仿学习基础方法.md) | 第 3 类：BC Transformer、ACT、Diffusion Policy、视觉运动策略（非 VLA 直接工作） |
| [sections/04_跨本体学习与泛化.md](sections/04_跨本体学习与泛化.md) | 第 4 类：cross-embodiment 联合训练、动作空间对齐、morphology transfer |
| [sections/05_WorldModel与机器人.md](sections/05_WorldModel与机器人.md) | 第 5 类：视频 / 视觉 world model、future latent 预测、imagined rollout |
| [sections/06_Grounding-Affordance-Referring.md](sections/06_Grounding-Affordance-Referring.md) | 第 6 类（**重点**）：REC/RES、开放词表 grounding、关系感知、phrase-to-mask、affordance、2D-to-3D embodied grounding |
| [sections/07_仿真环境与评测基准.md](sections/07_仿真环境与评测基准.md) | 第 7 类：LIBERO / SimplerEnv / RoboCasa / CALVIN / RLBench / RoboTwin |
| [sections/08_开源训练框架与生态.md](sections/08_开源训练框架与生态.md) | 第 8 类：Prismatic、LeRobot、openpi、StarVLA、RLDS |
| [sections/09_数据集与Scaling.md](sections/09_数据集与Scaling.md) | 第 9 类：Open X-Embodiment、BridgeData、DROID、UMI、AgiBot World、data scaling law |
| [sections/10_长程规划与System1-System2.md](sections/10_长程规划与System1-System2.md) | 第 10 类：LLM/VLM 高层规划 + VLA 低层执行、CoT for action、闭环重规划 |
| [sections/11_后训练与鲁棒闭环.md](sections/11_后训练与鲁棒闭环.md) | 第 11 类：RL post-training、test-time adaptation、failure recovery、鲁棒性评测 |
| [sections/12_实时部署与安全控制.md](sections/12_实时部署与安全控制.md) | 第 12 类：推理延迟、异步执行、力/触觉、双臂全身、安全约束、端侧部署 |
| [sections/A_全领域演进总览.md](sections/A_全领域演进总览.md) | 总结 A：Mermaid 演进图 + 关系表 |
| [sections/B_必读核心论文Top20.md](sections/B_必读核心论文Top20.md) | 总结 B：Top 20 优先阅读顺序 |
| [sections/C_Grounding驱动机器人操作阅读路径.md](sections/C_Grounding驱动机器人操作阅读路径.md) | 总结 C：referring/grounding → 候选验证 → mask → VLA 动作 的专项阅读路线 + 研究空白 |
| `zotero/`（**本地目录，已 gitignore，不随仓库同步**） | Zotero 可导入文件（`VLA-library.ris` / `VLA-library.bib`）+ 导入说明，含分块标签与中文注释 |

### 0.2 收录原则

1. 每篇论文只有**一个主归属分块**，跨类关系通过「交叉标签」列出，不重复罗列。
2. 标签只用三种：`奠基性论文` / `当前代表作` / `近期候选代表作`。
   - `近期候选代表作`：2025–2026 年发表、思路重要但**引用/复现/第三方评测证据尚不充分**的工作。
3. Venue 未正式发表一律写 `arXiv`，不虚构会议。无法在本次检索中确认的写 **待核验**。
4. 论文类型区分：`方法` / `数据` / `benchmark` / `系统` / `工程框架` / `综述`。
5. 涉及 action policy 的论文，必须注明**动作表示 / 输入模态 / 输出形式**。

### 0.3 可信度与局限（必读）

- 本库对 **2018–2025 年**的工作核对较充分；对 **2026 年上半年**的工作，主要依据检索到的公开条目建立索引，多数标注为 `近期候选代表作` 并附 **待核验**。
- 检索中出现但**本次未能可靠确认**的条目（例如用户提到的 `VLAFlow` 训练框架），已在对应分块的「注意事项」中显式记为 **未能核实**，不作猜测性描述。
- arXiv 编号与 venue 已尽力核对；凡带 **待核验** 标记者，请以 arXiv / 官方项目页为准后再回填本库。
- 部分工业界模型（Gemini Robotics 系列、Figure Helix、π 系列部分版本）**未开源权重**，其性能数字来自厂商自评，缺乏第三方复现，已在「注意事项」中标明。

### 0.4 更新流程建议

```
新论文 → 判定主归属分块（只选一个）→ 填 12 列表格 → 更新该分块「技术演进链」
      → 若进入 Top20 或阅读路径则同步更新 B/C → 追加到 zotero/VLA-library.ris → 重新导入 Zotero
```

---

## 1. 领域一句话概览

VLA 的核心命题是：**把互联网规模视觉-语言预训练的语义泛化能力，迁移到需要连续、实时、物理可行动作输出的机器人控制上**。
过去四年的主线是三条并行的 scaling：**模型 scaling**（VLM backbone 从 RT-1 的 35M 到 RT-2/OpenVLA 的 7B 再到 Gemini Robotics 级别）、**数据 scaling**（Open X-Embodiment → DROID → AgiBot World 百万级轨迹）、**动作表示 scaling**（离散 token → 频域 token → 连续回归 → diffusion → flow matching → 异步 chunk）。
当前尚未解决的三个硬瓶颈：**（a）真机评测不可复现**（仿真-真机差距 + 各家自建 setup）；**（b）长程任务的闭环纠错**（失败后不会恢复）；**（c）语言中的空间关系与同类实例消歧**（"最左边那个红杯子" 类指令仍显著失败）——第三点正是本库第 6 类与总结 C 的重点。
