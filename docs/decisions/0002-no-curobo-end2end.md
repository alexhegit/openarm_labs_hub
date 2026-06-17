# ADR 0002 — 不用 GraspGenX 自带的 end2end（cuRobo/Newton），改用 ROCm/CPU 管线

- 日期：2026-06-16
- 状态：已采纳
- 领域：架构 / 抓取流程

## 背景
GraspGenX 提供 end2end pick-and-place：GraspGenX → cuRobo 运动规划 → Newton/MuJoCo 回放。

## 问题
end2end 依赖 `nvidia-curobo / newton / warp / mujoco-warp`，**全是 CUDA-only**，在 AMD W7900 / ROCm 上不可用。

## 决策
只用 GraspGenX 的**抓取位姿生成**部分（torch-rocm，可在 W7900 跑），运动规划与回放改用自有 ROCm/CPU 栈：
- IK：mink（现有）/ PyRoki（JAX-rocm，可选升级）
- 回放+录像：`openarm_mp_labs` 的 MuJoCo 管线（CPU）

## 影响
安装 GraspGenX 时**不装 `end2end` extra**。轨迹生成责任在 `openarm_mp_labs`，GraspGenX 只产出 6-DOF 抓取位姿（isaac_grasp yml）。
