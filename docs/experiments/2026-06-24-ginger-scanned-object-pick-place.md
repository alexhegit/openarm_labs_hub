# 实验：Scan2Sim 扫描资产 ginger 的 pick-and-place

- 日期：2026-06-24
- 领域：仿真 / 抓取生成
- 相关仓库与分支：`openarm_mp_labs@venv`（665a832）、资产来自 `Scan2Sim`（alexhegit/Scan2Sim）
- 状态：成功

## 目标
把 GraspGenX 驱动的 pick-and-place 从固定方块泛化到真实 3D 扫描的不规则物体（ginger），全程用 GraspGenX 在 ginger 网格上推理出的抓取位姿，无任何硬编码抓点。成功标准：闭环仿真稳定抓起并抬升。

## 环境
- 镜像：`openarm-rocm:graspgen`（持久容器 graspgen-dev）
- 双 venv：planner venv 跑闭环仿真；GraspGenX 用其自身环境
- 渲染：osmesa 软渲染
- 硬件：W7900 gfx1100 / ROCm

## 步骤（可复现命令）
```bash
# 1. Scan2Sim：原始扫描 obj -> MuJoCo 资产（容器内，需 imagemagick 做贴图转换）
apt-get install -y imagemagick
cd /workspace/Scan2Sim
python -c "from scan2sim.core.batch_converter import convert_object; \
from pathlib import Path; convert_object(Path('assets/ginger/ginger_0529_1.obj'), unit_scale=0.001)"
# 产物：mjcf/ginger/{ginger.xml, meshes/visual.obj, collision.stl, texture.png}

# 2. GraspGenX 对 ginger 视觉网格推理抓取
cd /workspace/GraspGenX
python scripts/demo_object_mesh.py \
  --mesh_file /workspace/Scan2Sim/mjcf/ginger/meshes/visual.obj --mesh_scale 1.0 \
  --gripper_name openarm --grasp_threshold -1.0 --return_topk --topk_num_grasps 50 \
  --no-visualization --output_file /workspace/output/openarm_ginger_grasps.yml

# 3. 闭环仿真（planner venv），--object 指定扫描物体 MJCF
cd /workspace/openarm_mp_labs
/opt/venv-planner/bin/python -m openarm_mp_labs.demo_pick_place \
  --object /workspace/Scan2Sim/mjcf/ginger/ginger_lite.xml \
  --grasp-file /workspace/output/openarm_ginger_grasps.yml --grasp-mode full \
  --simulate-only

# 4. 录像（用抽取后的轻量网格，render-every 12）
MUJOCO_GL=osmesa /opt/venv-planner/bin/python -m openarm_mp_labs.demo_pick_place \
  --object /workspace/Scan2Sim/mjcf/ginger/ginger_lite.xml \
  --grasp-file /workspace/output/openarm_ginger_grasps.yml --grasp-mode full \
  --render-every 12 --record /workspace/output/pick_place_ginger_full.mp4
```

## 结果
- ginger 尺寸 ~69×48×65mm（最窄 48mm，逼近夹爪极限），mass 0.0575kg
- GraspGenX：50 个抓取，置信度 0.775–0.968
- topdown 模式：选中 conf 0.920，抬升 **120.4mm**
- full 模式：选中 conf 0.968，approach=[+0.92,+0.09,-0.38]（倾斜~67° 对角抓），抬升 **112.4mm**，IK 误差 0.5mm
- 产物：`output/pick_place_ginger_full.mp4`

## 结论
成立。GraspGenX→轨迹的闭环可直接泛化到扫描得到的不规则物体；只需 (a) Scan2Sim 把扫描转 MJCF，(b) 新增 `scene_builder.py` 自动拼场景替换方块，(c) 把抓取的世界接触点从"物体质心"改成 `base + GRASP_DEPTH_M·approach`（对非对称物体才正确，对方块仍≈质心，向后兼容）。

## 踩坑 / 后续（已同步 gotchas）
- Scan2Sim 导出贴图依赖 ImageMagick 的 `convert`，容器缺则导出失败（mesh 处理本身已成功）。
- 全分辨率视觉网格（148k 面）在 osmesa 软渲染下录像极慢（34 分钟未完）；抽取到 ~14k 面（`visual_lite.obj` + `ginger_lite.xml`）后 ~4m50s 完成，几何/抓取结果不变。
- 运动学吸附 `ATTACH_PHASES` 原含 `descend_grasp`，导致夹爪接触前物体就被"粘"住并沿接近方向滑动（大物体更明显）；已改为只在 `close_gripper` 阶段吸附，接触前物体保持静止，抬升不变。
- 待办：可探索基于接触力的真实闭合抓取，替代纯运动学就位。
