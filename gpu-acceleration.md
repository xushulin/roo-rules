## GPU 加速环境检测与配置规则（全局）

你在执行涉及模型训练、深度学习推理等 GPU 优势明显的任务时，**必须**按以下流程检测和配置 GPU 环境，无需再次请求批准。本规则适用于所有项目、所有工作区。

### 一、适用范围

以下任务类型在启动前**必须**执行 GPU 检测流程：

| 任务类别 | 示例 |
|---------|------|
| 深度学习模型训练 | PyTorch/TensorFlow/Keras 训练、微调、迁移学习 |
| 大规模推理 | 批量图片处理、视频分析、NLP 批量推理 |
| 数据增强处理 | 大规模图像变换、GAN 数据生成、Embedding 计算 |
| GPU 加速的算法 | CUDA 加速的传统计算机视觉、科学计算（CuPy 等） |
| 性能基准测试 | 涉及深度学习模型的 benchmark |

以下任务**不适用**本规则：

| 任务类别 | 原因 |
|---------|------|
| 轻量级推理（单张图片） | CPU 足以胜任 |
| 传统模板匹配/特征工程 | 无 GPU 加速库依赖 |
| 小规模数据处理 | 开销小于收益 |
| 项目编译/构建 | 与 GPU 无关 |

### 二、GPU 检测流程

执行 GPU 优势任务前，**必须**按以下顺序检测。

#### 2.1 硬件层检测

```
nvidia-smi
```

**判断标准**：

| nvidia-smi 输出 | 结论 | 后续操作 |
|-----------------|------|---------|
| 显示 GPU 信息和 CUDA 版本 | GPU 硬件可用 | 进入 2.2 软件层检测 |
| `command not found` / `不是内部或外部命令` | 无 NVIDIA 驱动或未安装 | 跳至 2.4 告知用户 |
| `NVIDIA-SMI has failed` | 驱动异常 | 跳至 2.4 告知用户 |

#### 2.2 Python/CUDA 软件层检测

```bash
python -c "import torch; print(f'PyTorch {torch.__version__}'); print(f'CUDA available: {torch.cuda.is_available()}'); print(f'CUDA version: {torch.version.cuda}'); print(f'GPU count: {torch.cuda.device_count()}')"
```

**判断标准**：

| 检测结果 | 结论 | 后续操作 |
|---------|------|---------|
| `CUDA available: True` | GPU 环境就绪 | 直接使用 GPU 执行任务 |
| `CUDA available: False` | PyTorch 未附带 CUDA 支持 | 进入 2.3 配置 GPU 环境 |
| `torch.version.cuda` 为 `None` 或报错 `Torch not compiled with CUDA enabled` | 当前 PyTorch 为 CPU 版 | 进入 2.3 配置 GPU 环境 |
| `ModuleNotFoundError: No module named 'torch'` | PyTorch 未安装 | 进入 2.3 配置 GPU 环境 |

#### 2.3 配置 GPU 环境

若软件层检测未通过，**必须**执行以下配置步骤。

##### 步骤 1：确认 Python 环境管理器

检查当前使用的 Python 环境：

```bash
python -c "import sys; print(sys.executable)"
```

| 环境类型 | 识别方式 | 操作方式 |
|---------|---------|---------|
| Conda | 路径包含 `anaconda3` 或 `conda` | 使用 Conda 管理 |
| venv / virtualenv | 路径包含 `.venv` 或 `venv` | 使用 pip 管理 |
| 系统 Python | 路径在系统目录 | 使用 pip 管理（可能需要 `--user`） |

##### 步骤 2：安装 PyTorch CUDA 版

**Conda 环境**（推荐）：

先确认 CUDA 版本（从 `nvidia-smi` 右上角获取，如 `CUDA Version: 12.9`），再安装对应版本：

```
conda install pytorch torchvision torchaudio pytorch-cuda=12.4 -c pytorch -c nvidia
```

**pip 环境**：

查询最新可用 GPU 版并安装。若不确定版本，先检查 PyTorch 官方推荐命令：

```
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

##### 步骤 3：验证安装

安装完成后**必须**重新执行 2.2 的检测命令，确认 `CUDA available: True`。

##### 步骤 4：回退策略

若安装 GPU 版失败（网络、权限等），**必须**告知用户具体错误信息，并**继续使用 CPU 执行**，不得无限阻塞。

#### 2.4 告知用户

若硬件层检测未通过（无 GPU 或无 NVIDIA 驱动）：

- **明确告知**用户本机无可用 GPU 硬件
- **使用 CPU 继续执行**任务
- 告知用户 CPU 执行预计耗时会更长

### 三、任务执行规范

#### 3.1 运行时强制使用 GPU

当 GPU 环境就绪时，训练/推理脚本**必须**使用以下确保 GPU 利用的方式。

**PyTorch**：

- 模型和数据必须 `.to(device)` 到 GPU
- 使用 `torch.cuda.amp` 混合精度加速（适用时）
- 使用 `torch.compile()` 等优化（PyTorch 2.0+）

**与设备无关的代码模式**：

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
```

#### 3.2 监控 GPU 使用

对于长时间运行的任务（如训练），启动后**应**简要检查一次 GPU 利用率：

```bash
nvidia-smi --query-gpu=utilization.gpu,memory.used,temperature.gpu --format=csv
```

若 `utilization.gpu` 持续为 0%，说明代码未正确使用 GPU，**必须**排查并修复。

#### 3.3 多 GPU 处理

若 `torch.cuda.device_count() >= 2`：

| 场景 | 策略 |
|------|------|
| 单次训练 | 使用 `DataParallel` 或单卡训练（通常足够） |
| 超参数搜索 / K-Fold | 使用多 GPU 并行运行不同参数组合 |
| 训练任务明确要求 | 配置文件指定 `CUDA_VISIBLE_DEVICES` |

### 四、检查清单

每次开始涉及 GPU 优势任务的**第一条消息**中，自问：

- [ ] 这个任务是否属于 GPU 优势任务（模型训练/大规模推理等）？
- [ ] 是否已执行 `nvidia-smi` 确认硬件可用？
- [ ] 是否已执行 PyTorch CUDA 可用性检测？
- [ ] 若环境未就绪，是否已安装 CUDA 版 PyTorch 并验证？
- [ ] 若硬件不可用，是否已告知用户并回退到 CPU？
- [ ] 任务代码中是否确保模型/数据已正确迁移到 GPU？
- [ ] 模型训练/数据处理开始后，是否确认 GPU 利用率 > 0%？

### 五、常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| `Torch not compiled with CUDA enabled` | PyTorch 为 CPU 版 | 按 2.3 安装 CUDA 版 PyTorch |
| `CUDA out of memory` | 显存不足 | 减小 batch_size、使用 gradient_accumulation、降低输入分辨率 |
| `nvidia-smi` 显示 GPU 但 `torch.cuda.is_available()` 为 False | PyTorch CUDA 版本与驱动不兼容 | 检查 CUDA 版本兼容矩阵，安装匹配版本 |
| Conda 安装 GPU 版失败 | 网络或镜像问题 | 用 pip 作为备选安装方案 |
| `LIBRARY_PATH` / `LD_LIBRARY_PATH` 错误 | CUDA Toolkit 环境变量缺失 | Conda 环境会自动管理；系统 Python 需手动配置 |

### 六、禁止行为

- ❌ 检测到可用 GPU 后仍然默认使用 CPU
- ❌ GPU 环境配置失败后无限循环重试而不回退到 CPU
- ❌ 不检测 GPU 就直接在 CPU 上训练模型
- ❌ GPU 环境配置完成后不验证就用
- ❌ 训练任务运行时 GPU 利用率为 0% 但不排查

### 七、与其他规则的关系

- 本规则与 [`environment-config.md`](.roo/rules/environment-config.md) 互补：环境配置规则定义基础运行时（Python/JDK/Android SDK），本规则定义 GPU 加速环境的检测与配置
- 本规则与 [`testing-requirements.md`](.roo/rules/testing-requirements.md) 协同：若测试脚本涉及深度学习模型，必须遵循本规则的 GPU 检测流程
- 本规则与 [`cleanup-temp-scripts.md`](cleanup-temp-scripts.md) 一致：GPU 配置脚本若为一次性使用，完成后需清理
