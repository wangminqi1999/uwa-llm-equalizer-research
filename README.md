# UWA LLM Equalizer Research Workspace

这是一个面向水声通信与 LLM/深度学习物理层算法研究的资料和代码工作区。仓库主要用于整理已有参考代码、论文资料、实验数据和后续实现计划，方便持续开发，也方便 AI 助手快速理解项目背景。

## AI 阅读入口

后续让 AI 接手开发时，建议按这个顺序阅读：

1. `PROJECT_HANDOFF.md`：项目目标、已确认决策、资源定位和后续方向。
2. `2026-06-08-uwa-llm-equalizer-implementation-plan.md`：水声 LLM 序列检测/信道均衡工程实现计划。
3. `AI_RESEARCH_COLLABORATION_GUIDELINES.md`：AI 协作时应遵守的研究和工程原则。
4. 本文件下方的目录说明：快速定位参考代码、论文和数据。

## 仓库内容概览

```text
.
├── PROJECT_HANDOFF.md
├── AI_RESEARCH_COLLABORATION_GUIDELINES.md
├── 2026-06-08-uwa-llm-equalizer-implementation-plan.md
├── BER_predicting-main/
├── DDQN_adaptive_modulation_coding-main/
├── estimate-main/
├── LLM4CP/
├── LLM4WM-main/
├── Multi-task PHY LLM/
├── matlab-main/
├── python-main-Underwater Acoustic Channels Group-lizhengnan/
├── watermark-D/
└── 参考文献/
```

## 主要目录

- `Multi-task PHY LLM/`：物理层 LLM 多任务参考代码，是后续 LLM 检测/均衡工程的重要参考框架。
- `DDQN_adaptive_modulation_coding-main/`：自适应调制编码和水声通信链路相关参考代码，包含 MATLAB/Python 代码和数据。
- `BER_predicting-main/`：BER 预测相关 DNN 示例。
- `LLM4CP/`：LLM4CP 相关代码、论文和 QuaDRiGa 数据生成工具。
- `LLM4WM-main/`：LLM 物理层/无线通信相关参考代码。
- `matlab-main/`：水声信道相关 MATLAB 工具与示例。
- `python-main-Underwater Acoustic Channels Group-lizhengnan/`：水声信道 Python 工具与示例。
- `estimate-main/`：信道估计相关 MATLAB 示例。
- `watermark-D/`：水声/信号数据与 MATLAB 脚本，包括 NOF1 数据。
- `参考文献/`：论文 PDF 和实验设计参考资料。

## 当前工程目标

当前研究方向聚焦于：

- 深海/水声通信场景；
- 序列检测或信道均衡；
- 使用 LLM 或类 LLM backbone 处理连续通信信号；
- 先建立可运行、可诊断、可复现的最小 BER 闭环，再逐步扩展实验。

更具体的接口、模型、baseline、数据生成和验证计划请以 `PROJECT_HANDOFF.md` 和实现计划文档为准。

## 协作约定

- 新增代码前先阅读顶层交接文档和协作指南。
- 涉及调制方式、信道模型、SNR 定义、baseline、模型输入输出等研究性选择时，需要明确说明依据。
- 优先复用现有参考代码中可靠的通信链路、数据生成、评估和模型结构。
- 不提交本地缓存、虚拟环境、训练日志、临时 checkpoint 和 IDE 配置。
