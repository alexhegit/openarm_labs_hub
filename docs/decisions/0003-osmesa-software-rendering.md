# ADR 0003 — 容器内 MuJoCo 离屏渲染用 osmesa 软件渲染

- 日期：2026-06-16
- 状态：已采纳
- 领域：渲染 / 录像

## 背景
容器内需要 MuJoCo 离屏渲染来录制 pick-and-place 视频。

## 问题
W7900 容器内 `MUJOCO_GL=egl` 硬件加速离屏渲染初始化失败。

## 决策
改用 `MUJOCO_GL=osmesa` 软件渲染，镜像装 `libosmesa6`（录像另需 `ffmpeg`）。

## 影响
渲染走 CPU、稍慢但稳定，画面与宿主机一致（实测 2150 帧 / cube 抬升 112mm）。若将来需要 GPU 加速渲染，再单独排查容器内 EGL。
