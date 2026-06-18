# 02-MLDL 环境搭建

> 读完本文你将了解：conda/uv 的用法对比、PyTorch/TensorFlow 版本匹配规则、CUDA/cuDNN 的配置和 GPU 验证，以及常见环境报错的定位和解决。

---

## 你大概率遇到过的问题

环境搭建是 ML 研究里最磨人的环节之一：

- **PyTorch 装上了但 GPU 检测不到**：`torch.cuda.is_available()` 返回 False，明明 nvidia-smi 能看到 GPU
- **两个项目需要不同版本的 PyTorch**：一个要 1.13 一个要 2.0，conda 环境切来切去
- **TensorFlow 要求 CUDA 11.8，你装的是 12.1**，TensorFlow 直接罢工
- **uv 和 conda 到底选哪个**，新项目用什么来管理环境
- **装了 cuDNN 但还是报 cuDNN 错误**，版本号和 PyTorch 编译用的 cuDNN 不匹配

这些问题本质上是 **Python 包、C++ 库、CUDA 驱动、硬件** 四层之间的不兼容问题。

---

## 常见的误解

| 你以为的 | 实际的 |
|---------|--------|
| pip install torch 会自动装上 GPU 支持 | pip install torch 默认装 CPU 版，GPU 版需要指定 index URL |
| nvidia-smi 能看到 GPU 就说明 PyTorch 能用 GPU | nvidia-smi 显示的是驱动支持的 CUDA 最高版本，不一定和 PyTorch 编译的 CUDA 版本匹配 |
| 装最新版 CUDA Toolkit 就对了 | PyTorch/TensorFlow 可能只支持某个 CUDA 范围，最新版反而不能用 |
| Conda 和 uv 只能二选一 | 可以组合使用：uv 做包管理，conda 做 Python 版本和 CUDA 管理 |

---

## Conda 和 uv 怎么选

### Conda（推荐 ML 项目）

优势：

- 管理 Python 版本（conda create -n xxx python=3.9）
- 管理非 Python 的二进制依赖（cudatoolkit、cuDNN、ffmpeg）
- 环境迁移方便（environment.yml）
- 社区生态成熟，ML 项目基本都有 conda 方案

劣势：依赖解析慢（mamba 是很好的替代，用法和 conda 一样但快很多）

### uv（推荐纯 Python 项目）

优势：

- **极快**，比 pip 快 10-100 倍
- 支持 lockfile（uv.lock），保证可复现
- 原生支持 pyproject.toml

劣势：

- 不管理 Python 版本（要配合 pyenv / rye）
- 不管理 CUDA 等系统库

### 实际组合方案

```
ML 项目：conda/mamba 管理环境 + pip/uv 装包
纯 Python 项目：uv 一把梭
```

```bash
# 用 mamba 替代 conda（更快）
mamba create -n myenv python=3.10
mamba activate myenv

# 用 uv 替代 pip（加速包安装）
uv pip install torch torchvision torchaudio
```

---

## PyTorch 版本匹配

### 核心规则

PyTorch 版本 + CUDA 版本 + cuDNN 版本需要三层对齐。

**查匹配表：** https://pytorch.org/get-started/previous-versions/

```bash
# 安装特定版本 PyTorch + CUDA 组合
pip install torch==2.1.0+cu121 --index-url https://download.pytorch.org/whl/cu121

# 验证
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

### CUDA 版本怎么选

先看驱动支持的 CUDA 版本：

```bash
nvidia-smi  # 看右上角 CUDA Version
```

然后装 PyTorch 时选择 **不超过该版本的 CUDA**。比如驱动支持 CUDA 12.4，可以装 cu118、cu121 等。

### cuDNN 冲突处理

大多数情况下你不用手动装 cuDNN——**PyTorch 的 pip wheel 自带 cuDNN**。

```bash
# 确认 PyTorch 编译用的 cuDNN 版本
python -c "import torch; print(torch.backends.cudnn.version())"
```

如果你非要自己装（比如为了性能），确保 cuDNN 和你 PyTorch 编译的版本兼容。否则要么用 conda 装：

```bash
conda install cudnn=8.9.2
```

---

## TensorFlow 版本匹配

TensorFlow 2.x 对 CUDA 版本要求非常严格：

| TensorFlow 版本 | CUDA | cuDNN |
|-----------------|------|-------|
| 2.15+ | 12.x | 8.9 |
| 2.10-2.14 | 11.8 | 8.6 |
| 2.5-2.9 | 11.2 | 8.1 |

**特别注意：** TensorFlow 2.10 之后 Windows 不再提供 GPU 支持。Windows 用户如果要 GPU 训练，要么用 WSL2，要么用 Docker，要么换 PyTorch。

```bash
# Linux 上安装 TF 2.15
pip install tensorflow==2.15.0

# 验证 GPU
python -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
```

---

## GPU 验证 checklist

装完环境强制走一遍：

```bash
# 1. 确认 GPU 可见
nvidia-smi

# 2. PyTorch GPU 检测
python -c "
import torch
print(f'PyTorch: {torch.__version__}')
print(f'CUDA: {torch.version.cuda}')
print(f'cuDNN: {torch.backends.cudnn.version()}')
print(f'GPU count: {torch.cuda.device_count()}')
print(f'Current GPU: {torch.cuda.get_device_name(0)}')
print(f'cuda available: {torch.cuda.is_available()}')
"

# 3. TensorFlow GPU 检测
python -c "
import tensorflow as tf
print(f'TensorFlow: {tf.__version__}')
gpus = tf.config.list_physical_devices('GPU')
print(f'GPU list: {gpus}')
"

# 4. 实际跑一个小张量操作
python -c "
import torch
x = torch.randn(1000, 1000).cuda()
y = torch.mm(x, x)
print(f'Matrix multiplication on GPU: {y.shape}')
"
```

每个步骤都能通过，才说明环境装好了。

---

## 常见报错速查

| 报错 | 原因 | 解决 |
|------|------|------|
| `RuntimeError: CUDA out of memory` | 显存不足 | 减小 batch size / sequence length / 用 fp16 |
| `ImportError: libcudart.so.x.x` | CUDA runtime 版本不对 | 检查 PyTorch 对应的 CUDA 版本，重新安装 |
| `AssertionError: Torch not compiled with CUDA enabled` | 装了 CPU 版 PyTorch | pip uninstall torch，然后装 CUDA 版 |
| `Could not find cudart` | cuDNN 路径问题 | conda install cudnn 或检查 LD_LIBRARY_PATH |
| `RuntimeError: NCCL error` | 多卡通信问题 | 设 `export NCCL_DEBUG=INFO` 查具体原因 |
| `pip install 速度极慢` | 国内网络 | 加 `-i https://pypi.tuna.tsinghua.edu.cn/simple` |

---

## 要点总结

1. **ML 环境搭建是四层对齐问题**：Python 包 -> C++ 库 -> CUDA 驱动 -> 硬件，缺一不可
2. **Conda + pip 是最稳的组合**，新项目可以尝试 mamba + uv 加速
3. **PyTorch 的 pip wheel 自带 cuDNN**，大多数情况下不需要手动装
4. **TensorFlow 2.10 以上 Windows 无 GPU 支持**，要用 WSL2 或 Docker
5. **GPU 验证不能只看 nvidia-smi**，要跑 torch.cuda.is_available() + 实际张量运算
6. **安装前先查 PyTorch 官网的版本兼容表**，确认 CUDA 版本后一步到位
