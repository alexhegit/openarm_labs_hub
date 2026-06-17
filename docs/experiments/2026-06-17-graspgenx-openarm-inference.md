# 实验：GraspGenX + OpenArm 夹爪 mesh 抓取推理（ROCm/W7900）

- 日期：2026-06-17
- 领域：抓取生成 / 环境
- 相关仓库与分支：`GraspGenX@visual-mesh-demo`（6f2e1b9）、`openarm_mp_labs@venv`（95fd743）
- 状态：成功

## 目标
在 W7900/ROCm 上跑通 GraspGenX 对 OpenArm 夹爪的 mesh 抓取推理，产出可用的 6-DOF 抓取位姿，且**不破坏**已验证的 torch 2.10-rocm 环境。成功标准：list_grippers 含 openarm + headless 推理输出有效 isaac_grasp yml。

## 环境
- 镜像：`openarm-rocm:unified`，常驻容器 `graspgen-dev`
- torch 2.10.0+rocm7.2.4 / torchvision 0.25.0+rocm7.2.4 / numpy 1.26.4（被 GraspGenX 降级）
- scene-synthesizer 1.15 / usd-core 26.5 / torch-geometric 2.8 / diffusers 0.11.1（均 py3.12 装上）
- 硬件：AMD Radeon Pro W7900 gfx1100，宿主 ROCm 6.3.2

## 步骤（可复现命令）
```bash
# 1. 常驻容器
docker run -d --name graspgen-dev --device=/dev/kfd --device=/dev/dri \
  --group-add video --group-add 110 --security-opt seccomp=unconfined \
  -v /DATA/AMD-Sim/OpenArm_Labs:/workspace -w /workspace/GraspGenX \
  openarm-rocm:unified sleep infinity

# 2. 安装（保护 torch：不加 rocm extra、不用 constraints 钉 torch）
docker exec graspgen-dev bash -lc '
  git config --global --add safe.directory /workspace/GraspGenX
  pip install -e .'

# 3. 拉 checkpoint 真权重（git-LFS）
docker exec graspgen-dev bash -lc '
  apt-get update -qq && apt-get install -y -qq git-lfs
  cd ext/graspgenx_checkpoints && git config --global --add safe.directory "$(pwd)"
  git lfs install && git lfs pull'

# 4. 验证夹爪 + headless 推理
docker exec graspgen-dev bash -lc 'python scripts/list_grippers.py | grep -i openarm'
docker exec graspgen-dev bash -lc '
  python scripts/demo_object_mesh.py \
    --mesh_file assets/sample_data/object_mesh/box.obj --mesh_scale 1.0 \
    --gripper_name openarm --grasp_threshold -1.0 --return_topk --topk_num_grasps 50 \
    --no-visualization --output_file /workspace/output/openarm_box_grasps.yml'
```

## 结果
- `list_grippers` → openarm 在列（#1）✓
- headless 推理 rc=0，GPU 推理 ~1.5–4.5s
- **50 个 6-DOF 抓取，置信度 0.42–0.63**
- 产物：`output/openarm_box_grasps.yml`（isaac_grasp 格式，position + quaternion + confidence）
- torch 2.10 GPU 在 numpy 降级后仍正常（gpu_sum 4096）✓

## 结论
GraspGenX 抓取生成在 ROCm/W7900 上可用，openarm 夹爪 onboard 成功。pip（不加 rocm extra、不用 constraints）能在保留 torch 2.10 的前提下装齐 py3.12 依赖。

## 踩坑 / 后续
- 坑已记入 `docs/gotchas.md`：constraints 钉本地版 torch、git safe.directory、checkpoint LFS、numpy 降级、夹爪缺缓存用 dummy、容器可写层。
- 待办：
  - 为 openarm 生成 `points.json/tsdf.npy` 缓存以提升分数到标称 0.70–0.99
  - 阶段 8：把 yml 抓取位姿接入 openarm_mp_labs 轨迹层做端到端闭环
