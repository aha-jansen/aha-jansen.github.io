---
title: "CentOS 安装 TensorRT 指南"
date: 2025-12-04
draft: false
tags:
  - AI
  - TensorRT
  - 工具
categories:
  - AI Infra
---

## 前置条件

在开始之前，请确保您的系统满足以下条件：

- CentOS 8 系统环境
- 拥有 `sudo` 权限
- 已正确安装 NVIDIA 显卡驱动和与目标 TensorRT 版本兼容的 CUDA Toolkit

**重要提示**：在安装时，需要关注两个版本号对应，一个是 TensorRT 的版本（比如8.4），另一个是 CUDA 的版本（比如 CUDA 11.7）

## 第一步：添加 TensorRT 软件仓库

首先，需要从 NVIDIA 官网下载与系统及 CUDA 版本匹配的 `rpm` 仓库配置文件。

**下载仓库文件**：访问 NVIDIA TensorRT 8.x 下载页面，根据 RHEL/CentOS 版本和 CUDA 版本选择并下载对应的 `rpm` 文件。文件名格式通常为：`nv-tensorrt-repo-rhel8-cudaX.Y-trtA.B.C.D-ga-YYYYMMDD.x86_64.rpm`

**安装仓库**：
```bash
# 将 xxx 替换为您下载的实际文件名
sudo dnf install -y nv-tensorrt-repo-xxx.rpm
```

## 第二步：安装 TensorRT 核心库

添加仓库后，安装 TensorRT 的核心运行时库、开发库和解析器：

```bash
sudo dnf install -y libnvinfer8 libnvinfer-devel libnvinfer-plugin8 libnvparsers8 libnvonnxparsers8
```

### 常见问题：缺少 `libcublas` 依赖

**报错信息**：
```
Problem: conflicting requests - nothing provides libcublas.so.11()(64bit) needed by ...
```

**原因**：TensorRT 依赖于 CUDA 的 `cublas` 库，但系统默认的软件源中可能没有这个包。

**解决方案**：

1. 首先，尝试在已配置的仓库中查找提供该文件的包：
```bash
sudo dnf whatprovides 'libcublas.so.11()(64bit)'
```

2. 如果找不到，需要添加 NVIDIA 官方的 CUDA 仓库：
```bash
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
sudo dnf clean all
sudo dnf makecache
```

3. 添加仓库后，再次查找并安装所需的 `cublas` 包：
```bash
# 注意：下面的版本号仅为示例，请根据 whatprovides 的输出安装正确的版本
sudo dnf install -y libcublas-11-5-11.7.4.6-1.x86_64
```

安装完依赖后，重新执行本步骤开头的核心库安装命令。

## 第三步：安装并验证 C++ 执行工具 (`trtexec`)

`trtexec` 是 TensorRT 自带的命令行工具，用于快速测试模型性能，验证 C++ 环境是否安装成功。

尝试执行：
```bash
trtexec --version
```

### 常见问题：`command not found: trtexec`

**原因**：包含 `trtexec` 的二进制工具包没有被安装，或者其路径未在系统的 `PATH` 环境变量中。

**解决方案**：

1. 使用 `dnf provides` 查找提供 `trtexec` 命令的包：
```bash
sudo dnf provides '*/trtexec'
```

2. 根据输出结果，安装对应的包（通常是 `libnvinfer-bin`）：
```bash
# 版本号仅为示例，请根据上一步的输出进行安装
sudo dnf install -y libnvinfer-bin-8.4.3-1.cuda11.6.x86_64
```

3. 安装后，使用找到的绝对路径进行验证（常见路径为 `/usr/src/tensorrt/bin/trtexec`）：
```bash
/usr/src/tensorrt/bin/trtexec --version
```

如果成功输出版本信息，则 C++ 工具安装成功。您可以选择将此路径添加到 `~/.bashrc` 的 `PATH` 中以便后续使用。

## 第四步：安装并验证 Python API

最后，需要为 Python 环境安装 TensorRT 绑定，以便在 Python 代码中调用它。

尝试在 Python 中导入 `tensorrt`：
```bash
python3 -c "import tensorrt as trt; print(trt.__version__)"
```

### 常见问题：`ModuleNotFoundError: No module named 'tensorrt'`

**原因**：TensorRT 的 Python 绑定包尚未安装，或者安装到了系统路径，而你当前使用的是虚拟环境。

**解决方案**：

1. 首先，查找并安装 Python 绑定包（通常是 `python3-libnvinfer`）：
```bash
sudo dnf provides python3-libnvinfer
# 根据输出结果安装，版本号仅为示例
sudo dnf install -y python3-libnvinfer-8.4.3-1.cuda11.6.x86_64
```

2. **如果仍无法导入（尤其是在虚拟环境中）**，说明它被安装到了系统 Python 路径下。需要将其链接到当前环境。找到 `tensorrt` 模块的实际安装位置：
```bash
rpm -ql python3-libnvinfer-8.4.3-1.cuda11.6.x86_64 | grep site-packages
```

3. 在您的 Python 环境中创建一个软链接指向它：
```bash
# 格式: ln -s <源路径> <目标路径>
ln -s /usr/lib64/python3.8/site-packages/tensorrt /path/to/your/env/lib/python3.8/site-packages/tensorrt
```

4. 最后，再次运行验证命令，此时应能成功打印版本号。

## 总结

至此，您应该已经在 CentOS 8 系统上成功安装了 TensorRT 的 C++ 和 Python 环境。通过遵循本指南的排错步骤，可以解决大部分安装过程中的依赖和路径问题。
