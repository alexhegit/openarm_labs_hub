# Playbook — 容器内跑 OpenArm pick-and-place + 录像

目标：在 `openarm-rocm:unified` 容器里跑通 `openarm_mp_labs` 的 pick-and-place 并录制双机位视频。

```bash
docker run --rm --device=/dev/kfd --device=/dev/dri \
  --group-add video --group-add 110 --security-opt seccomp=unconfined \
  -v /DATA/AMD-Sim/OpenArm_Labs:/workspace \
  -w /workspace/openarm_mp_labs openarm-rocm:unified bash -lc '
  /opt/venv-planner/bin/pip install -q --no-deps -e ../openarm_control -e ../openarm_mujoco -e .
  MUJOCO_GL=osmesa /opt/venv-planner/bin/python -m openarm_mp_labs.demo_pick_place \
    --record output/pick_place_container.mp4'
```

预期：`Recorded ~2150 frames`，`Peak cube lift ~112 mm`，产物 `output/pick_place_container.mp4`。

检查结果：看 `output/pick_place_container.mp4`，确认夹爪尖抓住 cube 并抬升、未脱落。

注意：
- 用 `venv-planner`（含 mujoco/mink），不是系统 python。
- `--no-deps` 装本地三包，避免动到环境其它依赖。
- 软渲染（osmesa），别用 egl（见 ADR 0003）。
