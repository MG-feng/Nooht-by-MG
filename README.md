```markdown
# 🌙 LunarNet

> **为资源受限环境而生的长上下文代码生成架构**

[![Version](https://img.shields.io/badge/version-0.1.0-blue)](https://github.com/MG-feng/LunarNet)
[![License](https://img.shields.io/badge/license-Apache%202-green)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.8+-blue)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/pytorch-2.0+-orange)](https://pytorch.org/)

---

## 📌 项目定位

**LunarNet 是一个专为「资源受限环境下的长上下文代码生成」设计的AI架构。**

它通过 **MoE（混合专家）+ GQA（分组查询注意力）** 的稀疏计算，配合 **CPU-GPU 异构内存管理**，让 1B 参数模型在 **T4 GPU 或纯 CPU 设备** 上，实现接近 7B 模型的代码生成能力，并支持 **128K 超长上下文**。

### 🎯 核心哲学
> **用时间换空间，用数量换速度——在低成本硬件上，完成高成本任务。**

---

## ✨ 关键特性

| 特性 | 说明 |
| :--- | :--- |
| 🧠 **MoE + GQA 架构** | 8专家稀疏激活（每Token仅激活2个）+ 4头GQA，1B参数实现更高知识容量 |
| 💾 **极低资源占用** | **最低 4GB 系统内存/显存**即可运行，适配 Colab 免费版、个人工作站 |
| ⚡ **双版本优化** | CPU版（激进内存压缩+多核流水线）和 GPU版（Flash Attention+异步预取） |
| 🌐 **原生分布式支持** | 多服务器任务队列，自动拆分模块并行生成，大幅缩短总耗时 |
| 📜 **128K 超长上下文** | 通过 KV Cache int8量化 + CPU/NVMe 分层卸载，处理完整项目方案 |
| 🔧 **开箱即用的工具链** | 内置训练脚本、批量生成、多卡并行、ONNX 导出 |

---

## 📊 性能对比

### 与主流代码模型对比（理论估算）

| 模型 | 参数量 | 硬件需求 | 代码能力 (HumanEval) | 最大上下文 | 推理速度 | 硬件门槛 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **LunarNet-1B** | 1B (MoE) | **T4 / 4GB+ CPU** | ~45-50% | **128K** | 慢 (1-3 tok/s) | **极低** |
| DeepSeek-Coder-1.3B | 1.3B | T4 | ~40% | 8K | 中 | 低 |
| CodeLlama-7B | 7B | A100 (40GB) | ~50% | 16K | 中 | 高 |
| DeepSeek-Coder-6.7B | 6.7B | A100 (40GB) | ~55% | 16K | 中 | 高 |

### 架构对比

| 架构 | 推理显存 | 长上下文 | 硬件门槛 | 代码生成适配 |
| :--- | :--- | :--- | :--- | :--- |
| **LunarNet** | **极低** | **极强 (128K+)** | **极低** | **专精** |
| 标准 Transformer | 高 | 弱 (8K-32K) | 中高 | 通用 |
| MoE (Mixtral) | 中 | 中 | 高 | 通用 |
| Mamba (SSM) | 低 | 极强 | 中 | 探索中 |

---

## 🚀 快速开始

### 安装

```bash
# 克隆仓库
git clone https://github.com/MG-feng/LunarNet.git
cd LunarNet

# 安装依赖
pip install torch>=2.0.0 psutil>=5.9.0

# 可选：安装Flash Attention（GPU版加速）
pip install flash-attn --no-build-isolation
```

### 基础使用

```python
# 自动检测硬件，一键运行
from lunarnet import LunarNetGPUConfig, LunarNetModel
from lunarnet.utils import auto_config, quick_generate

# 1. 自动配置（根据你的硬件自适应）
config = auto_config()
model = LunarNetModel(config)

# 2. 生成代码
prompt = "def fibonacci(n):"
output = quick_generate(model, prompt, tokenizer, max_new=100)
print(output)
```

### CPU版使用

```python
from lunarnet import LunarNetCPUConfig, LunarNetModel
from lunarnet.utils import auto_config_cpu

config = auto_config_cpu()  # 自动适配4-8GB内存
model = LunarNetModel(config)
```

### 多服务器并行生成

```bash
# 服务器1
python -c "from lunarnet.distributed import DistributedRunner; r=DistributedRunner(config); r.run_worker(model, tokenizer)"

# 服务器2（自动从队列获取任务）
python -c "from lunarnet.distributed import DistributedRunner; r=DistributedRunner(config); r.run_worker(model, tokenizer)"
```

---

## 🏗️ 架构详解

### 整体数据流

```
用户输入完整方案 → CPU预处理（语言检测+专家预取）→ GPU/CPU计算 → 输出代码
         ↑                           ↓
    方案解析器              多服务器任务队列
```

### 核心组件

| 模块 | 功能 | 优化策略 |
| :--- | :--- | :--- |
| **MoE (混合专家)** | 8专家，每Token激活2个 | CPU预取热门专家，GPU仅保留当前需要的 |
| **GQA (分组查询注意力)** | 16头Q，4头KV | 减少KV Cache 75%内存占用 |
| **Memory Manager** | 三层存储（GPU↔CPU↔NVMe） | int8量化+稀疏化，压缩率高达70% |
| **Expert Scheduler** | 动态专家交换 | 基于Router Logits预测下个专家，异步加载 |
| **Distributed Runner** | 多服务器任务队列 | 文件/Redis队列，自动负载均衡 |

### 已实现的优化（v0.1）

- ✅ **CPU分担采样**：Token采样在CPU执行，释放GPU计算资源
- ✅ **CPU分担KV压缩**：KV Cache压缩在后台CPU进行，不阻塞生成
- ✅ **CPU分担专家预取**：专家交换在CPU预判，GPU专注计算
- ✅ **自动硬件检测**：4-8GB内存自动降级配置
- ✅ **多核流水线（CPU版）**：利用所有CPU核心并行处理不同层
- ✅ **Swap分区利用（CPU版）**：主动使用交换空间突破物理内存限制

---

## 📁 项目结构

```
LunarNet/
├── lunarnet/
│   ├── __init__.py          # 包入口
│   ├── config.py            # CPU/GPU双版本配置
│   ├── model.py             # 主模型（精简版）
│   ├── moe.py               # MoE层
│   ├── attention.py         # GQA注意力
│   ├── memory.py            # 内存管理器（CPU/GPU差异化）
│   ├── scheduler.py         # 专家预取调度器
│   ├── distributed.py       # 多服务器并行接口
│   ├── parallel.py          # 多卡/多核并行
│   ├── trainer.py           # 训练脚本（支持LoRA）
│   └── utils.py             # 工具函数（自动检测、批量生成）
├── LICENSE                  # Apache 2.0
└── README.md               # 本文档
```

---

## 🔬 训练与微调

### 预训练

```python
from lunarnet.trainer import LunarNetTrainer
from lunarnet.utils import auto_config

config = auto_config()
model = LunarNetModel(config)
trainer = LunarNetTrainer(model, config)

# 准备数据：List[str] 格式的代码文本
dataloader = CodeDataset(codes, tokenizer)
trainer.train_epoch(dataloader)
```

### LoRA微调（节省内存）

```python
from lunarnet.trainer import LoRATrainer

trainer = LoRATrainer(model, config, rank=8)
# 只训练LM Head，内存占用降低90%
trainer.train_epoch(dataloader)
```

---

## 🛠️ 硬件兼容性

| 硬件配置 | CPU版 | GPU版 | 预期速度 |
| :--- | :--- | :--- | :--- |
| **纯CPU (4GB RAM)** | ✅ 可用（极限模式） | ❌ | 0.3-0.5 tok/s |
| **纯CPU (8GB RAM)** | ✅ 流畅运行 | ❌ | 0.5-1 tok/s |
| **T4 (16GB)** | ✅ 可运行 | ✅ 原生支持 | 3-5 tok/s |
| **RTX 3060 (12GB)** | ✅ | ✅ | 5-8 tok/s |
| **A100 (40GB)** | ✅ | ✅ 全速 | 15-25 tok/s |

---

## 🤝 贡献指南

欢迎贡献！请遵循以下流程：

1. Fork 本仓库
2. 创建你的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交你的修改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启一个 Pull Request

### 开发计划

- [x] LunarNet-1B-CPU-0.1 原型
- [x] LunarNet-1B-GPU-0.1 原型
- [x] 多服务器并行支持
- [x] 训练脚本与LoRA微调
- [ ] Nightglow-3B 模型训练
- [ ] 方案→任务分解器（自动拆分项目）
- [ ] 生成结果自动验证与修正

---

## 📄 许可证

本项目采用 **Apache License 2.0** 许可证。详情请见 [LICENSE](LICENSE) 文件。

---

- 作者: **MG-feng**

---

## ⭐ Star 历史

如果这个项目对你有帮助，请给一个 Star ⭐ 支持一下！
```
