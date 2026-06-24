# 实验：GraspGenX 抓取位姿 → 轨迹 → MuJoCo 端到端闭环（阶段 8）

- 日期：2026-06-22
- 领域：抓取生成 / 运动规划 / 仿真
- 相关仓库与分支：`openarm_mp_labs@venv`（17d78e8）、`GraspGenX@rocm`（6f2e1b9，原 visual-mesh-demo 已并入 rocm）
- 状态：成功

## 目标
把 GraspGenX 生成的 6-DOF 抓取位姿接入 `openarm_mp_labs` 轨迹层（替换写死的俯抓），用现有 mink IK 跑通「推理 → 轨迹 → MuJoCo 回放」。成功标准：抓取由 yml 驱动且能稳定抓起方块。

## 环境
- 容器 `graspgen-dev`（`openarm-rocm:graspgen`）
- 推理：系统 python（torch 2.10-rocm）；回放：`/opt/venv-planner`（mujoco 3.x + mink）
- `MUJOCO_GL=osmesa` 软渲染录像

## 关键设计：坐标系/抓取帧映射
- GraspGenX `isaac_grasp` 的 `position`=夹爪**基座** `openarm_left_ee_base_link`，`+z`=接近方向（指尖在 +z 0.068，见 openarm `config.json`）。
- MuJoCo 侧 EE 控制点用 `TCP_OFFSET_LOCAL=[0.005,0,-0.150]`，指尖方向≈ site **−z**——与 GraspGenX **相反**。
- 实测：每个 grasp 的接近线都过物体质心（最近距离 2–14mm），沿接近偏移 d*≈0.11–0.15m。
- 映射策略：**指尖世界目标 = sim 实测 cube 中心；接近轴/朝向取自 GraspGenX 选中的 grasp**。新增 `grasp_io.py`（解析+选 grasp+帧变换），`build_poses` 泛化为「任意接近轴 + 抓取朝向」，pre-grasp 沿 −approach 退让、lift 恒定竖直。

## 步骤
```bash
# 推理：与 orange_cube 同尺寸的 40mm 方块网格
docker exec graspgen-dev bash -lc 'cd /workspace/GraspGenX && python - <<PY
import trimesh; trimesh.creation.box(extents=[0.04]*3).export("/workspace/output/cube_40mm.obj")
PY
python scripts/demo_object_mesh.py --mesh_file /workspace/output/cube_40mm.obj \
  --gripper_name openarm --grasp_threshold -1.0 --return_topk --topk_num_grasps 50 \
  --no-visualization --output_file /workspace/output/openarm_cube40_grasps.yml'
# 闭环
docker exec graspgen-dev bash -lc 'cd /workspace/openarm_mp_labs && \
  MUJOCO_GL=osmesa /opt/venv-planner/bin/python -m openarm_mp_labs.demo_pick_place \
  --grasp-file /workspace/output/openarm_cube40_grasps.yml --grasp-mode full \
  --record /workspace/output/pick_place_graspgenx_full.mp4'
```

## 结果
- 40mm 方块推理：50 个抓取，置信度 0.565–0.839（比示例 box 高）。
- `--grasp-mode topdown`（复用已验证竖直姿态）：**抬升 112.2mm**，与写死路径一致。
- `--grasp-mode full`（GraspGenX 真实 6-DOF）：选中 conf=0.817 的**水平侧抓**（approach=[+1,0,0]），IK 误差 0.6mm，**抬升 115.7mm**，录像 2150 帧 `output/pick_place_graspgenx_full.mp4`。

## 结论
轨迹层已能消费 GraspGenX 任意朝向抓取并跑通端到端闭环，全程 ROCm/CPU、不碰 CUDA。「写死俯抓」→「抓取生成驱动」的解耦完成。

## 踩坑 / 后续
- 坑（已记 gotchas）：`topdown` 模式必须直接用 home 朝向；早期「强制 approach=[0,0,-1] 再重对齐」引入倾斜→抓空 lift=0。
- 后续：换非立方体物体（其 mesh 推理 + sim 换物体/位姿）；side-grasp 的抓取附着改用接触力而非运动学 seat；阶段 6.2 夹爪缓存提分。
