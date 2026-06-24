# Playbook — GraspGenX 对 OpenArm 夹爪做 mesh 抓取推理

目标：给定物体 mesh，用 GraspGenX 生成 openarm 夹爪的 6-DOF 抓取位姿。

## 前置（首次）
见实验记录 `docs/experiments/2026-06-17-graspgenx-openarm-inference.md` 的安装 + checkpoint 步骤，
或直接用已固化镜像 `openarm-rocm:graspgen`（若已 commit）。

> ⚠️ 用 fork **`alexhegit/GraspGenX`**，不是上游。OpenArm 夹爪与 ROCm 支持上游均未合入，由该 fork 实现并提 PR：
> OpenArm 夹爪 [NVlabs/GraspGenX#3](https://github.com/NVlabs/GraspGenX/pull/3)、ROCm [NVlabs/GraspGenX#1](https://github.com/NVlabs/GraspGenX/pull/1)。
> AMD 安装：`uv sync --extra rocm`（`rocm` 与 `end2end` 互斥，见 ADR 0002）。

## 跑推理（headless）
```bash
docker exec graspgen-dev bash -lc '
  cd /workspace/GraspGenX
  python scripts/demo_object_mesh.py \
    --mesh_file <你的物体.obj> --mesh_scale 1.0 \
    --gripper_name openarm \
    --grasp_threshold -1.0 --return_topk --topk_num_grasps 50 \
    --no-visualization --output_file /workspace/output/<名字>_grasps.yml'
```
支持格式：`.obj/.stl/.ply` 及 USD。

## 输出
`output/<名字>_grasps.yml`（isaac_grasp 格式）：每个 grasp 含 `confidence` + `position` + `orientation`(quaternion w/xyz)。
后续由 `openarm_mp_labs` 读取、选一个抓取转成轨迹目标。

## 接到 openarm_mp_labs 跑 demo（完整闭环）
GraspGenX 与 openarm_mp_labs 通过这个 YAML 文件解耦（ADR 0005）。生成后这样引用：
```bash
cd openarm_mp_labs
uv run openarm-mp-demo \
  --object /path/to/<obj>.xml --grasp-file output/<obj>_grasps.yml \
  --grasp-mode full --record output/<obj>_pick_place.mp4
```
**权威端到端流程（新物体 → demo → 输出）见**
`openarm_mp_labs/docs/new_object_pick_place.md`（含 mesh 准备、帧约定、内置示例 ginger、常见坑）。
换物体的位置/朝向无需重跑本推理；只有换不同形状的物体才需要重新推理。

## 检查结果
- 日志看 `Inferred N grasps, scores: a — b`，置信度合理（目标 0.70–0.99；当前 openarm 缺缓存约 0.42–0.63）。
- yml 文件非空、grasp 数量符合 `--topk_num_grasps`。

## 可选：可视化
```bash
docker exec graspgen-dev bash -lc '
  cd /workspace/GraspGenX
  python scripts/demo_object_mesh_vis.py --gripper_name openarm --mesh_file <物体.obj>'
# viser GUI: http://localhost:8080
```
