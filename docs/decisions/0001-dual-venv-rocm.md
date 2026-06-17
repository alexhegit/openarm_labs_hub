# ADR 0001 — ROCm 统一镜像采用「双 venv + 进程分离」

- 日期：2026-06-16
- 状态：已采纳
- 领域：环境 / ROCm

## 背景
通用抓取流程需要 GraspGenX（PyTorch-ROCm）和规划侧 PyRoki（JAX-ROCm）+ MuJoCo 同时可用。

## 问题
torch-rocm 与 jax-rocm 装进同一 site-packages 会因 ROCm 运行库 / numpy ABI 冲突而崩溃，连单独 import torch 都段错误。

## 决策
**一个 Docker 镜像，两个隔离 venv**：
- 系统 python（`/opt/venv`）：torch 2.10-rocm → GraspGenX
- `/opt/venv-planner`：jax + mujoco + mink → PyRoki / 规划与回放

集成时**进程分离**：GraspGenX 走自带 ZMQ server，或通过 grasp yml 文件解耦。

## 备选方案
- **单 venv 全装**：已验证必崩，否决。
- **双容器**：更干净但更重，GPU 依赖隔离收益不足以抵消编排成本；保留为未来选项。

## 影响
镜像 `openarm-rocm:unified`。两栈不能在同一进程内互调，必须经文件/IPC。
