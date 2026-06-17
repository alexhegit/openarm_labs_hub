# 踩坑速查（Gotchas）

> 回报最高的文件。任何子项目（仿真/RL/IL/ROS）碰到环境问题先查这里。
> 格式：**现象 → 根因 → 解法**。新坑往上加，标日期。

---

## ROCm / GPU

### W7900 上 torch GPU 段错误，但 `torch.cuda.is_available()` 返回 True（2026-06-16）
- **现象**：`rocm/pytorch:...release_2.7.1` 镜像里 `is_available()` 为 True，一做真实 GPU 计算就段错误。
- **根因**：该镜像 torch 的 arch list 只含 `gfx942/gfx1201/gfx950`，**不含 W7900 的 gfx1100**。`is_available()` 不校验当前卡是否在编译目标里，极具迷惑性。
- **解法**：改用 `rocm/pytorch:latest`（torch 2.10.0+rocm7.2.x，arch list 含 gfx1100）。验证要做**真实计算**：`float(torch.ones(4096,device="cuda").sum())` 应得 4096，而不是只看 `is_available()`。

### torch-rocm 与 jax-rocm 装进同一环境必崩（2026-06-16）
- **现象**：同一 site-packages 同时装 torch-rocm + jax-rocm 后，连单独 import torch 都段错误。
- **根因**：两者各自捆绑的 ROCm 运行库 / numpy ABI 冲突。
- **解法**：**一个镜像、两个隔离 venv** —— 系统 python 放 torch（GraspGenX），`/opt/venv-planner` 放 jax+mujoco+mink。集成时进程分离。详见 ADR 0001。

### ROCm 7.2 容器跑在 ROCm 6.3 宿主上（2026-06-16）
- **结论（好消息）**：ROCm 7.2 容器在本机 ROCm 6.3.2 宿主上正常，容器内 `rocminfo` 能看到 W7900 gfx1100，**无需升级宿主驱动**。
- **注意**：传 GPU 设备要用数字 GID：`--device=/dev/kfd --device=/dev/dri --group-add video --group-add 110`（render 组名解析在某些镜像里失败，用数字 110 稳）。

### jax-rocm wheel 不在公共 PyPI（2026-06-16）
- **根因**：`pip install jax[rocm]` 在公共 PyPI 找不到对应 wheel。
- **解法**：用 AMD 官方源 `https://repo.radeon.com/rocm/manylinux/rocm-rel-7.2/`，或直接用 `rocm/jax` 镜像自带的 jax。

---

## 渲染 / 录像

### 容器内 EGL 初始化失败（2026-06-16）
- **现象**：W7900 容器内 MuJoCo 用 `MUJOCO_GL=egl` 初始化失败，无法离屏渲染。
- **解法**：改用软件渲染 `MUJOCO_GL=osmesa`，镜像里装 `libosmesa6`。录像还需 `ffmpeg`。性能够用，画面与宿主机一致。

---

## GraspGenX 安装（py3.12 / ROCm）

### 别用 constraints 钉本地版 torch（2026-06-17）
- **现象**：`pip install -e . -c constraints.txt`（constraints 里写 `torch==2.10.0+rocm7.2.4.git...`）→ `ResolutionImpossible`，pip 报 torch 无匹配分发。
- **根因**：本地版后缀 `+rocm7.2.4.git...` 在任何远程索引都找不到，constraints 逼 pip 去远程解析。
- **解法**：**不要 constraints**。已装的 torch 2.10 满足 `torch>=2.1`，pip 默认就保留不动。装 GraspGenX 用 `pip install -e .`、**不加 `--extra rocm`**（rocm extra 会把 torch 路由到 rocm 索引重装）。

### git "dubious ownership" 导致 setuptools-scm 构建失败（2026-06-17）
- **现象**：`pip install -e .` 报 `fatal: detected dubious ownership in repository`。
- **根因**：宿主用户拥有目录、容器内 root 运行，git 拒绝读版本。
- **解法**：`git config --global --add safe.directory /workspace/GraspGenX`（checkpoint 子仓同理）。

### checkpoint 是 git-LFS 指针，不是真权重（2026-06-17）
- **现象**：`import graspgenx` 自动 clone `ext/graspgenx_checkpoints`，但 `.pth` 只有 ~135 字节。
- **根因**：git-LFS 指针文件，未拉取真权重。
- **解法**：`apt install git-lfs && cd ext/graspgenx_checkpoints && git lfs install && git lfs pull` → gen 1.2GB / dis 484MB。

### numpy 被 GraspGenX 钉降到 1.26.4（2026-06-17）
- **现象**：装 GraspGenX 后 numpy 2.4.6 → 1.26.4（pyproject 硬钉 `numpy==1.26.4`）。
- **结论**：**实测 torch 2.10 GPU 仍正常**（gpu_sum 4096），可接受。若后续别的包要 numpy 2.x，再评估。

### 新夹爪缺缓存文件，推理用 dummy 值（2026-06-17）
- **现象**：openarm 推理日志 `points.json / proc_gripper_only_pointnet_vae_repr.json / tsdf.npy not found, using dummy values`。
- **影响**：能跑出抓取（conf 0.42–0.63），但可能低于标称 0.70–0.99。
- **解法（待办）**：按 GraspGenX README「Integrating a New Gripper」为 openarm 生成这些缓存。

### 容器可写层 vs 挂载盘（2026-06-17）
- **注意**：`pip install` 的依赖装在**容器可写层**，不在挂载盘。容器删掉即丢。
- **解法**：跨容器复用就 `docker commit graspgen-dev openarm-rocm:graspgen` 或写进 Dockerfile；源码/checkpoint 因在挂载盘 `/workspace` 而保留。

---

## Git / 仓库管理

### output 产物不要入库（2026-06-16）
- 视频/checkpoint/grasp yml 体积大，统一放 `output/` 并 gitignore；在实验记录里登记路径即可。
