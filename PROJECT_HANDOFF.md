# 深海水声 LLM 序列检测/信道均衡项目交接说明

> 更新日期：2026-06-07  
> 用途：让没有参与此前讨论的 AI、开发者或初学者能够理解项目目标、已有决策、资源定位和后续工作。  
> 当前状态：研究设计与最小可行试跑范围已确认；尚未开始正式工程实现，也尚未编写文件级 implementation plan。

## 1. 阅读约定

本文使用以下状态标记，避免把建议误认为已确定结论：

- **已确认**：此前讨论中已经明确认可，后续不应随意修改。
- **暂定/推荐**：已有推荐方案，但仍可在实施前调整。
- **后续讨论**：当前不影响最小可行试跑，留到后续阶段再确认。
- **未开始**：尚未实施或验证。

如果本文与较早讨论中的方案冲突，以本文记录的较新结论为准。

## 2. 项目要做什么

### 2.1 最终目标

构建一个面向深海水声通信的、基于 LLM 微调的单任务序列检测/信道均衡研究代码工程，并形成可用于论文实验的完整闭环。

核心流程是：

```text
发送 bit
-> 调制与发送帧生成
-> 水声信道与噪声
-> 接收信号 rx
-> LLM 序列检测/均衡算法
-> 估计发送符号序列 x_hat
-> 判决/解调
-> BER 评估与 BER-SNR 曲线
```

### 2.2 核心研究任务

**已确认：核心算法做序列检测/信道均衡，不做信道估计。**

核心算法接口语义为：

```text
接收信号 rx + 辅助信道信息 h_aux
-> 发送符号序列估计 x_hat
-> BER
```

`h_aux` 可以来自：

- `h_true`：真实信道，用于理想条件、upper bound 和 sanity check。
- `h_hat`：估计信道，用于更实际的主实验。

信道估计可以出现在系统中，但其结果只是核心算法的辅助输入、传统 baseline 的中间步骤或实验对照条件，不是 LLM 的预测目标。

### 2.3 整个项目不是一个函数

完整工程预计包含：

1. bit、调制符号和发送帧生成；
2. 水声信道、噪声和接收信号生成；
3. 数据集与数据划分；
4. LLM 检测/均衡核心算法；
5. 传统 baseline；
6. 训练、验证和测试流程；
7. 判决、解调、BER 统计和绘图；
8. 后续真实/准真实水声实验与消融实验。

### 2.4 第一版基本架构还需要的工程支撑

**已确认：第一版目标是跑通一个可复现的最小 BER 闭环，而不是一次性完成完整论文实验。**

除主链路功能外，第一版还应包含以下支撑骨架：

1. 统一数据格式约定；
2. 统一配置入口；
3. sanity check 脚本或检查函数；
4. 训练、评估、baseline 和绘图的清晰入口；
5. checkpoint、loss、BER 和曲线数据保存；
6. 随机种子与可复现性控制；
7. 一个轻量 debug model，用于排查数据链路和训练循环问题。

这些内容不是额外论文贡献点，而是保证后续可以稳定替换数据、模型、baseline 和实验设置的工程基础。第一版不应只做一个能运行一次的脚本。

## 3. 总体技术路线

### 3.1 基础代码框架

**已确认：`Multi-task PHY LLM` 是主要基础框架。**

保留或参考：

- GPT-2 / LoRA 微调结构；
- 训练与评估骨架；
- “信道 + 接收数据 -> 连续发送符号估计”的检测链路；
- 检测任务的数据组织和输出语义。

不需要继承：

- 多任务 router；
- 自然语言 instruction 选择任务；
- 多任务 loss 和多个任务输出头；
- 原代码中写死的无线 MIMO、`Nr=128`、`Nt=8`、16-QAM 和固定 token 数等设定。

### 3.2 信号域分层

**已确认：采用“通带物理系统描述 + 等效复基带算法输入”。**

```text
论文/数据生成层：
  可描述载频、带宽、采样率、Doppler 和通带传播/replay

适配层：
  通带数据做下变频、同步、重采样和裁剪

核心算法层：
  输入等效复基带 rx_bb 和复基带信道辅助 h_aux
```

这样既不会把项目限制为抽象的纯基带实验，也能让核心模型接口稳定。以后加入通带或实测数据时，只需扩展数据与适配层，不推翻核心算法。

### 3.3 通信体制

**已确认：主线先采用单载波 SC-FDE，代码结构预留 OFDM/S-NOFDM 扩展。**

选择 SC-FDE 的原因：

- 与现有 `DDQN` 水声通信链路最接近；
- 已有调制、CP/PN、LS 信道估计、MMSE/FDE 和 BER 参考；
- 适合先稳定完成序列检测/均衡闭环。

OFDM/S-NOFDM 属于后续扩展，不是当前 MVP 的阻塞项。

### 3.4 调制与编码

**已确认：不把不同调制阶数和信道编码作为当前研究变量。**

- 固定一种基础调制方式；
- 第一阶段不加入信道编码；
- 核心关注水声信道下的序列检测/均衡性能。

**默认实现选择：MVP 使用 BPSK。**  
历史讨论没有把 BPSK 与 QPSK 作为需要用户立即拍板的阻塞问题。接手者在没有收到新要求时应直接使用 BPSK，不要重新发起多调制方案讨论，也不要扩展成多调制对比实验。

## 4. 核心接口与数据约定

### 4.1 核心算法输入

**已确认：当前不把 `rx only` 作为必要设定，保留以下两种模式。**

```text
模式 A：rx_bb + h_true -> x_hat
模式 B：rx_bb + h_hat  -> x_hat
```

- 模式 A 用于理想条件、数据链路检查和 upper bound。
- 模式 B 用于更实际的主实验。

### 4.2 核心算法输出

**已确认：输出连续复符号序列估计 `x_hat`。**

建议接口形状：

```text
x_hat: [batch, seq_len, 2]
```

最后一维表示实部和虚部。即使 MVP 使用 BPSK，也建议保留复数双通道接口，便于后续替换调制方式或处理复基带信号。

BER 不由模型直接输出，而是在 evaluator 中完成：

```text
x_hat
-> 符号判决/解调
-> pred_bits
-> 与 true_bits 比较
-> BER
```

### 4.3 最小样本字段

**已确认：样本遵循最小必要原则，不保存复杂 label 和大量中间信号。**

统一 Dataset 样本至少提供：

```text
rx_bb
h_aux
tx_symbols
```

数据集或评估流程还需要知道 `snr`，用于 BER-SNR 聚合和绘图，但它不一定作为模型输入。

`tx_bits`、`tx_indices` 和 BER 可以通过统一后处理函数派生，不要求全部存入每个样本：

```text
tx_symbols -> true_bits
x_hat -> decision/demod -> pred_bits
BER = compare(pred_bits, true_bits)
```

### 4.4 发送帧与模型预测范围

**已确认：发送信号结构参考 DDQN 风格。**

发送帧可以包含：

```text
Pilot / training part
+ data block(s)
+ CP / PN / guard structure
```

**已确认：模型只学习有效 data symbols 的检测/均衡。**  
Pilot、CP 和 PN 用于数据生成、同步、信道估计或辅助输入，不作为模型预测目标。

以下细节曾被讨论，但后来明确推迟，不应在当前文档中视为定案：

- 一个信道样本对应一个 frame 还是一个 data block；
- 一个 frame 包含多少 block；
- `h_hat` 每帧估计一次还是每 block 更新；
- `Len_block`、`nBlock`、`N_cp` 的具体数值；
- BER 首先按 block 还是 frame 统计。

这些应在数据生成模块的 implementation plan 中再确认。

## 5. 数据生成与数据源设计

### 5.1 统一发送结构，双生成路径

**已确认：上层统一生成 DDQN 风格发送帧，下层支持两条接收信号生成路径。**

```text
generate_tx_frame(...)
  -> bits / symbols / pilot / CP / frame

路径 A：DDQN-style 等效复基带
  frame_bb
  -> complex channel filter
  -> complex noise
  -> rx_bb

路径 B：replay-style 通带回放
  frame_bb
  -> 上变频
  -> 水声信道 replay
  -> 真实/准真实噪声
  -> 下变频、同步、裁剪
  -> rx_bb
```

两条路径最终整理成统一样本：

```text
rx_bb + h_aux + tx_symbols
```

### 5.2 数据源优先级

**已确认：数据源必须可替换，模型和训练流程不能依赖某一个具体数据源。**

阶段安排：

1. 当前最小可行试跑：优先使用现有数据或 DDQN-style 基带生成路径，验证核心算法和 BER 闭环。
2. 正式主数据：后续替换为用户自己仿真的基带水声信道数据集。
3. 扩展实验：使用 replay、Watermark 或其他真实/准真实水声数据进行泛化和补充实验。

当前不需要提前实现所有 Dataset 类，但设计上应保证不同数据源都转换为相同 sample 格式。

### 5.3 数据划分

**已确认：当前试跑可以使用简单随机 train/val/test 划分。**

代码要求：

- 数据源和划分方式集中在一个清楚的入口处；
- 不写死在模型内部；
- 不散落在很多文件中；
- 不引入初学者难以理解的高级配置或插件系统；
- 后续能够方便切换到按信道、SNR、文件、场景或数据源划分。

正式实验的划分方式尚未确定，属于后续讨论内容。

## 6. 模型结构设计

### 6.1 总体数据流

**已确认的数据流：**

```text
rx_bb + h_aux
-> 输入特征处理 / encoder
-> token embeddings
-> GPT-2 / LoRA backbone
-> decoder / output projection
-> x_hat
-> evaluator
-> BER
```

### 6.2 代码组织方式

**已确认：第一版采用一个主模型类，内部用清楚的方法分段，不做过度模块化。**


目标是：

- `forward` 中能直观看懂数据如何流动；
- 后续替换 encoder 时，不需要修改 Dataset、训练循环和 BER evaluator；
- 后续替换 decoder 时，不需要修改 Dataset、训练循环和 BER evaluator；
- 不引入复杂的基类、注册器、插件系统或过度分文件设计。

`LLM4CP` 和 `LLM4WM-main` 可以作为实现参考，但不改变上述已确认的简单组织原则：

- `LLM4CP`：复数信号实/虚特征、连续物理数据转 token embedding、output projection 等思路；
- `LLM4WM-main`：preprocess、adapter、LLM forward、output projection、postprocess 的功能分段；
- `Multi-task PHY LLM`：GPT-2/LoRA 训练框架和连续符号检测输出。

## 7. 训练、评估与 baseline

### 7.1 训练目标

**已确认：训练使用符号级 MSE/NMSE，BER 只作为评估指标。**

```text
loss = MSE(x_hat, tx_symbols)
```

如果后续效果不足，再考虑 constellation-aware loss、分类 loss 或 BER surrogate；这些不属于第一阶段必要内容。

### 7.2 结果与指标

**已确认：**

- 主指标：BER；
- 主要结果图：BER-SNR 曲线；
- 辅助记录：训练 MSE/NMSE；
- baseline 与 LLM 使用同一批数据和同一 BER 计算函数进行比较；
- 可比较 `h_true` 与 `h_hat` 输入条件。

### 7.3 第一阶段 baseline

**已确认：**

1. 主 baseline：`LS + MMSE/FDE`；
2. 理想对照：`h_true + MMSE/FDE`；
3. 复杂 baseline 后续扩展。

baseline 不应写死在训练脚本中。代码应保留简单、直观的独立入口，例如独立函数或清楚的 baseline 选择位置，方便后续添加 ZF、TR、VTRM、CNN equalizer 等方法，但不要建立复杂插件系统。

## 8. 最小可行试跑（MVP）

### 8.1 已确认范围

最小可行试跑的目标不是产出最终论文结果，而是验证整个工程和关键逻辑正确。

核心范围：

```text
DDQN-style/已有基带数据
-> 统一 Dataset
-> rx_bb + h_true 输入
-> LLM 检测/均衡模型
-> x_hat
-> MSE 训练
-> BER 评估
-> h_true + MMSE/FDE baseline
```

跑通后再接：

```text
h_hat 输入
LS + MMSE/FDE baseline
```

### 8.2 MVP 不能只验证“脚本能跑”

**已确认：MVP 成功标准是分层 sanity check + 小规模端到端训练。**

1. 数据链路正确性
   - 检查 `rx_bb`、`h_aux`、`tx_symbols` 对齐；
   - 无噪声或高 SNR 下，`h_true + MMSE/FDE` 的 BER 应接近 0；
   - 若传统理想 baseline 都无法恢复，优先检查生成、裁剪和标签。

2. 模块接口正确性
   - `x_hat.shape` 与 `tx_symbols.shape` 对齐；
   - encoder、LLM、decoder 的输入输出语义明确；
   - encoder/decoder 的局部替换不应迫使 Dataset、训练循环或 evaluator 一起改。

3. 训练正确性
   - 小数据集上 loss 能下降；
   - 极小 batch 能过拟合，用于发现标签、loss、输出或数据流错误。

4. 评估正确性
   - BER 由统一 decision/demod 函数计算；
   - LLM 和 baseline 使用同一批数据与同一评估逻辑；
   - 能输出可检查的 BER 和基础结果。

5. 可复现性和结果留存
   - 固定随机种子，覆盖 bit、信道、噪声、数据划分和模型初始化；
   - 保存 checkpoint、训练 loss、验证/测试 BER、baseline BER 和 BER-SNR 曲线数据；
   - 保存绘图所需的原始数值，不只保存图片。

6. debug model 检查
   - 保留一个轻量 MLP、CNN 或小 Transformer 作为调试模型；
   - debug model 不作为论文主算法，只用于确认数据、loss、BER evaluator 和训练循环是否合理；
   - 当 LLM 效果异常时，先用 debug model 区分“模型能力问题”和“数据链路/评估问题”。

### 8.3 当前是否还有阻塞确认项

**已确认：MVP 范围已经确认完成，目前没有其他需要用户立即拍板的阻塞项。**

实施过程中可能出现具体工程问题，但应优先按上述已有约定和最小目标处理，不重新翻动已经确认的方向。

## 9. 复杂实验扩展

### 9.1 当前定位

**后续讨论：复杂实验扩展仍然存在，但不阻塞 MVP。**

目标包括：

- 加入真实/准真实水声信道；
- 加入更强 baseline；
- 扩展跨信道、跨噪声、跨 Doppler 等实验；
- 增加论文式消融与泛化验证。

### 9.2 后续需要确认的问题

等 MVP 结果出来后，再单独讨论：

1. 使用哪类真实/准真实水声信道、数据源或模型；
2. 强 baseline 具体加入哪些；
3. 实验矩阵扩展到什么规模；
4. 正式 train/val/test 如何按信道、SNR、文件或场景划分；
5. 复杂实验阶段的成功标准、算力和数据规模。

这些问题不应在 MVP 实施前被提前定死。

### 9.3 第一版暂不展开的内容

**已确认：以下内容暂不作为第一版基本架构的必要范围。**

- 多调制方式对比；
- 信道编码；
- 复杂真实海试数据接入；
- 大量深度学习 baseline；
- 大规模消融实验；
- LoRA、量化和不同 LLM backbone 的系统对比；
- 论文级完整实验矩阵。

这些内容后续仍可扩展，但应等最小 BER 闭环、baseline 和 sanity check 跑通后再逐步加入。

## 10. 现有资源及其定位

### 10.1 主要 LLM 通信代码

| 资源 | 在本项目中的定位 |
|---|---|
| `Multi-task PHY LLM/code_final_version` | 主要基础框架；参考 GPT-2/LoRA、训练骨架、检测链路和连续符号输出 |
| `LLM4CP/LLM4CP-master` | 参考复数特征处理、连续序列 embedding、频域/时延域思路和 output projection |
| `LLM4WM-main/LLM4WM-main` | 参考 adapter、LLM forward 和 output projection 的功能分段 |

### 10.2 水声代码与数据

| 资源 | 在本项目中的定位 |
|---|---|
| `DDQN_adaptive_modulation_coding-main` | 复基带 SC-FDE 链路、发送帧、调制/解调、LS、MMSE/FDE、BER 和基础 baseline 参考 |
| `python-main-Underwater Acoustic Channels Group-lizhengnan` | Python 版实测水声信道 replay、真实海洋噪声和信道数据适配 |
| `matlab-main` | MATLAB/Octave 版 replay、noisegen、unpack；用于理解通带回放流程 |
| `estimate-main` | 生成或估计 `h_hat`、`theta_hat`、`phi_hat`；作为信道辅助输入来源或信道估计参考 |
| `watermark-D` | 后期真实/准真实水声 benchmark、packet replay 和泛化测试 |
| `BER_predicting-main` | 可作为 BER 预测相关参考，但不是当前核心算法基础 |

### 10.3 参考论文

`参考文献` 文件夹中的论文主要用于确定或扩展：

- 水声系统和实验参数；
- 通带与等效复基带表述；
- 稀疏多径、时延扩展和 Doppler 场景；
- 传统和深度学习 baseline；
- BER-SNR 曲线、消融和泛化实验。

论文作为实验设计和方法参考。


## 12. 讨论中遇到的主要问题与处理方式

### 问题 1：核心算法是否做信道估计

处理方式：明确将信道估计排除出主任务。`h_true/h_hat` 只作为辅助输入，主输出始终是发送序列估计。

### 问题 2：后续需要换数据、encoder、decoder 和 baseline

处理方式：固定接口语义和集中切换位置，但不提前实现复杂插件系统。替换局部模块时，不应迫使其他流程大改。

### 问题 3：MVP 只要运行成功是否足够

处理方式：不够。必须同时验证数据链路、模块接口、局部替换边界、小 batch 过拟合、loss 下降、BER 和 baseline 正确性。

### 问题 4：是否现在就确定复杂实验全部细节

处理方式：不需要。复杂实验扩展保留为后续阶段，等 MVP 跑通后再依据结果确认真实信道、强 baseline 和实验矩阵。

## 13. 给接手 AI 或开发者的工作原则

1. 不要把主任务改成信道估计、信道预测或多任务学习。
2. 不要直接照搬 `Multi-task PHY LLM` 的 MIMO 和多任务设定。
3. 不要为了“可扩展”建立复杂插件、注册器或配置框架。
4. 优先复用现有代码思路，但统一到本项目已确认的接口。
5. 每个实现阶段都应先做可验证的 sanity check。

