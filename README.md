# OpenArm Labs Hub

OpenArm 仿真 / 强化学习 / 模仿学习 / ROS 控制 探索的**知识中枢**。

这个仓库**不存放任何子项目的实现代码**，只沉淀跨项目的：
- 仓库地图与可复现清单（`repos.yaml`）
- 踩坑速查（`docs/gotchas.md`）
- 架构决策记录 ADR（`docs/decisions/`）
- 实验记录（`docs/experiments/`）
- 环境知识与可复现手册（`docs/environments/`、`docs/playbooks/`）

> 原则：**代码留在各自仓库，经验集中在这里。** 上游 enactic 克隆是只读参考，绝不往里写笔记（重拉即丢、也推不上去）。

---

## 仓库地图

工作根目录：`/DATA/AMD-Sim/OpenArm_Labs/`

| 目录 | 来源 | 分支 | 角色 | 领域 |
|---|---|---|---|---|
| `openarm_labs_hub` | **本仓库（fork: alexhegit）** | main | 知识中枢 | 横切 |
| `openarm_mp_labs` | **fork: alexhegit** | venv | 运动规划 labs（pick-and-place 轨迹+MuJoCo 回放+双机位录制） | 仿真 / 运动规划 |
| `GraspGenX` | **fork: alexhegit** | visual-mesh-demo | 跨形态 6-DOF 抓取生成（已 onboard openarm 夹爪 + ROCm） | 抓取生成 |
| `openarm` | 上游 enactic | main | 主项目/总入口 | 参考 |
| `openarm_description` | 上游 enactic | main | URDF / 模型描述 | 模型 |
| `openarm_control` | 上游 enactic | main | 控制 | 控制 |
| `openarm_mujoco` | 上游 enactic | master | MuJoCo 模型/资产 | 仿真 |
| `openarm_simulation` | 上游 enactic | main | 仿真 | 仿真 |
| `openarm_isaac_lab` | 上游 enactic | main | Isaac Lab（RL） | 强化学习 |
| `openarm_maniskill_simulation` | 上游 enactic | rocm | ManiSkill 仿真（RL/IL） | 强化学习 / 模仿学习 |
| `openarm_dataset` | 上游 enactic | main | 数据集（模仿学习） | 模仿学习 |
| `openarm_teleop` | 上游 enactic | main | 遥操作（采集示教数据） | 模仿学习 |
| `openarm_ros2` | 上游 enactic | main | ROS 2 集成 | ROS 控制 |
| `output/` | 共享产物目录 | — | 视频/checkpoint/grasp yml（已 gitignore，路径登记在实验记录里） | — |

精确 commit 见 [`repos.yaml`](./repos.yaml)。还原整套实验室：

```bash
cd /DATA/AMD-Sim/OpenArm_Labs
vcs import . < openarm_labs_hub/repos.yaml      # 需要 vcstool: pip install vcstool
```

---

## 怎么用这个中枢（约定）

1. **每碰到一个坑** → 记一条到 `docs/gotchas.md`（含现象 + 根因 + 解法）。
2. **每做一个"为什么这么选"的判断** → 写一篇 ADR 到 `docs/decisions/`（一条一篇，几句话即可）。
3. **每做一次成规模的探索** → 复制 `docs/experiments/TEMPLATE.md` 写一篇实验记录。
4. **每跑通一条可复现流程** → 整理成 `docs/playbooks/` 手册。
5. **环境/镜像变更** → 更新 `docs/environments/`，并在 `repos.yaml` 同步 commit。

## 目录

```
openarm_labs_hub/
├── README.md            # 本文件：仓库地图 + 约定
├── repos.yaml           # vcstool 清单（钉 url/branch/commit）
├── docs/
│   ├── gotchas.md       # 踩坑速查（回报最高）
│   ├── decisions/       # ADR 架构决策记录
│   ├── experiments/     # 实验记录（含 TEMPLATE.md）
│   ├── environments/    # Docker/ROCm 环境知识
│   └── playbooks/       # 可复现 how-to
```

## 当前进度速览

- ✅ ROCm 统一镜像 `openarm-rocm:unified`（双 venv：torch / jax+mujoco），三栈 GPU 验证通过
- ✅ openarm_mp_labs pick-and-place 容器内跑通（osmesa 软渲染录像）
- ✅ GraspGenX + openarm 夹爪 mesh 推理跑通（50 个 6-DOF 抓取，isaac_grasp yml）
- ✅ 阶段 8：GraspGenX 位姿 → openarm_mp_labs 轨迹 → MuJoCo 回放**端到端闭环跑通**（topdown 112mm / full 侧抓 115mm）
- ⏳ 阶段 7（可选）：PyRoki IK 进 venv-planner
- ⏳ 后续：非立方体物体、side-grasp 接触力附着、夹爪缓存提分
