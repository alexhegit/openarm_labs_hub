# Playbook — GraspGenX 对 OpenArm 夹爪做 mesh 抓取推理

目标：给定物体 mesh，用 GraspGenX 生成 openarm 夹爪的 6-DOF 抓取位姿。

## 前置（首次）
见实验记录 `docs/experiments/2026-06-17-graspgenx-openarm-inference.md` 的安装 + checkpoint 步骤，
或直接用已固化镜像 `openarm-rocm:graspgen`（若已 commit）。

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
后续阶段 8 由 `openarm_mp_labs` 读取 top-1 位姿转成轨迹目标。

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
