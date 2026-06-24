# ADR 0005 — GraspGenX 与 openarm_mp_labs 用文件契约解耦，不做代码集成

- 日期：2026-06-24
- 状态：已采纳
- 领域：架构 / 抓取流程

## 背景
希望 `openarm_mp_labs` 能对**新物体**泛化抓取，而不只是回放已 commit 的抓取 YAML。
自然的想法是把 GraspGenX 的抓取姿态估计"集成进" `openarm_mp_labs`。

## 问题 / 权衡
- **环境冲突**：planner 侧是 jax-rocm7（openarm_control + mujoco），GraspGenX 侧是
  torch-rocm7。两个 ROCm 框架同环境易冲突（当初分双 venv 即因此）。
- 抓取模型**必须**有 torch + GPU + checkpoint，这部分无法消除。
- 评估过的形态：
  - A. torch+GraspGenX 直接装进 openarm_mp_labs（单环境）→ 冲突风险 + 破坏轻量自包含仿真。
  - B. ZMQ 客户端/服务端（GraspGenX 自带 `serving.zmq_server/zmq_client`，客户端不依赖 torch）。
  - C. 子进程调用 GraspGenX CLI。
  - D. 文件契约 + 文档化流程（不写集成代码）。

## 决策
采用 **D：文件契约 + 文档化**。GraspGenX 与 `openarm_mp_labs` 只通过 `isaac_grasp` YAML
交接；如何为新物体生成该文件、如何在 `openarm_mp_labs` 引用它跑 demo，全部写成文档，
不在代码里耦合 GraspGenX。

## 理由
- 最灵活：换抓取生成器、换机器、批处理、离线复算都不受影响。
- 解耦彻底：planner 环境保持纯 jax，无 torch；仿真项目可独立运行。
- 成本最低：无需维护 RPC/服务/打包；B/C 可作为后续可选增强，不影响现状。

## 影响
- 权威流程文档：`openarm_mp_labs/docs/new_object_pick_place.md`（README 链接）。
- hub playbook `run-graspgenx-inference.md` 串起跨项目闭环并指向该文档。
- 代价：抓取约定（物体帧、+z 接近、米制居中）是**隐式契约**，无 schema 校验；
  约定变更需手动同步 `grasp_io.py`。后续若需"一键对新物体抓取"，再评估方案 B（ZMQ 服务）。
