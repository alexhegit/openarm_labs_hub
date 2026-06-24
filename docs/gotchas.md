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

## 集成 / 坐标系

### GraspGenX 抓取帧 ↔ MuJoCo EE 帧方向相反（2026-06-22）
- GraspGenX `isaac_grasp` 的 `position`=夹爪基座、`+z`=接近方向（指尖在 +z 0.068）。
- 我们 MuJoCo 侧 `TCP_OFFSET_LOCAL=[0.005,0,-0.150]`，指尖≈ site **−z**。两者相反，映射时要按各自约定换算，别直接套用。
- 实用结论：每个 grasp 的接近线都过物体质心 → 指尖目标可取 sim 实测物体中心，接近轴/朝向取自 grasp。

### topdown 模式别"强制竖直再重对齐"（2026-06-22）
- **现象**：GraspGenX 驱动闭环 lift=0（抓空），但 IK 误差只有 1mm。
- **根因**：已验证的俯抓直接用 home EE 朝向（已校准好曲面指尖夹取）。我额外「强制 approach=[0,0,-1] 再把 home 朝向对齐过去」，反而相对 home 引入几度倾斜，指尖偏离方块。
- **解法**：`topdown` 模式直接复用 home 朝向、竖直退让；只有 `full/best` 模式才用 grasp 的真实朝向做对齐。修复后 topdown 抬升 112mm、full（侧抓）115mm。

### 非对称物体：接触点 ≠ 质心（2026-06-24）
- **现象**：方块用"指尖目标=物体质心"可行；换成扫描 ginger（不规则）后，质心不是抓取实际接触处。
- **解法**：把抓取的世界接触点改成 `base + GRASP_DEPTH_M·approach`（`GRASP_DEPTH_M≈0.12`，从方块标定，object-agnostic）。对方块仍≈质心，向后兼容。

### 运动学吸附别在接近阶段触发（2026-06-24）
- **现象**：夹爪还没接触物体,物体就沿接近方向跟着滑动（大物体更明显）。
- **根因**：`ATTACH_PHASES` 含 `descend_grasp`，且 `ATTACH_DISTANCE_M=0.08`（8cm）；pinch 点一进 8cm 就把物体粘住,之后随夹爪平移。
- **解法**：`ATTACH_PHASES` 去掉 `descend_grasp`，只在 `close_gripper`（真正闭合）阶段吸附。抬升不变。

## 环境 / 资产转换

### Scan2Sim 导出贴图依赖 ImageMagick（2026-06-24）
- **现象**：`scan2sim convert` 在导出阶段报 `[Errno 2] No such file or directory: 'convert'`（mesh 处理已成功）。
- **解法**：容器内 `apt-get install -y imagemagick`。抓取/物理本身不需要贴图。

### 高模视觉网格在 osmesa 下录像极慢（2026-06-24）
- **现象**：ginger 全分辨率视觉网格 148k 面，osmesa 软渲染录像 34 分钟未完。
- **解法**：`trimesh.simplify_quadric_decimation`（装 `fast-simplification`）抽取到 ~14k 面，做 `*_lite.xml` 专供录像；~4m50s 完成，几何/抓取结果不变。仿真本身用全分辨率不受影响。

## Git / 仓库管理

### output 产物不要入库（2026-06-16）
- 视频/checkpoint/grasp yml 体积大，统一放 `output/` 并 gitignore；在实验记录里登记路径即可。
