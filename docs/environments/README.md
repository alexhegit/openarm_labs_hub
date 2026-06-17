# 环境知识（Docker / ROCm）

## 宿主机
- Docker 29.3.0
- amdgpu-dkms 6.10.5 / ROCm 用户态 6.3.2
- GPU：AMD Radeon PRO W7900（48GB，gfx1100）
- `/dev/kfd`、`/dev/dri` 存在；用户在 `video` / `render` 组

## 统一镜像 `openarm-rocm:unified`
基于 `rocm/pytorch:latest`（torch 2.10-rocm，含 gfx1100），双 venv（见 ADR 0001）：

| venv | 路径 | 内容 | 服务 |
|---|---|---|---|
| 系统 | `/opt/venv` | torch 2.10-rocm | GraspGenX（抓取生成） |
| planner | `/opt/venv-planner` | jax-rocm + mujoco + mink | PyRoki / 规划 / 回放录像 |

Dockerfile 维护在 `openarm_mp_labs/docker/`（详见该仓 `docker/README.md`）。

## GPU 容器启动模板
```bash
docker run --rm --device=/dev/kfd --device=/dev/dri \
  --group-add video --group-add 110 --security-opt seccomp=unconfined \
  -v /DATA/AMD-Sim/OpenArm_Labs:/workspace \
  openarm-rocm:unified bash -lc '<cmd>'
```
> `--group-add 110` 是 render 组的数字 GID（组名解析在某些镜像里失败）。

## 派生镜像 / 容器
- `openarm-rocm:graspgen`（32.3GB）：已 `docker commit` 固化，= unified + GraspGenX 全套依赖。容器删/主机重启后直接复用，无需重装。
- `graspgen-dev`（常驻容器）：源自上者。checkpoint 真权重在挂载盘 `/workspace/GraspGenX/ext/`，不在镜像里。

复用启动：
```bash
docker run -d --name graspgen-dev --device=/dev/kfd --device=/dev/dri \
  --group-add video --group-add 110 --security-opt seccomp=unconfined \
  -v /DATA/AMD-Sim/OpenArm_Labs:/workspace -w /workspace/GraspGenX \
  openarm-rocm:graspgen sleep infinity
```

## 验证命令（真实 GPU 计算，不只看 is_available）
```bash
# torch
python -c "import torch;print(torch.cuda.get_device_name(0), float(torch.ones(4096,device='cuda').sum()))"
# jax
/opt/venv-planner/bin/python -c "import jax;print(jax.devices())"
# mujoco（osmesa 软渲染）
MUJOCO_GL=osmesa /opt/venv-planner/bin/python -c "import mujoco;print(mujoco.__version__)"
```

## 关键约束
- torch-rocm 必须含 gfx1100（用 `rocm/pytorch:latest`，**别用 release_2.7.1**）
- torch 与 jax 不可同 venv（见 gotchas）
- 容器内渲染用 `MUJOCO_GL=osmesa`（见 ADR 0003）
