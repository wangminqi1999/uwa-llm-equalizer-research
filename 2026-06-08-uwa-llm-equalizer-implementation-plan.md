# 深海水声 LLM 序列检测/信道均衡代码工程实现计划

## 1. 项目背景

本项目面向深海水声通信中的信号检测或信道均衡问题。水声通信信道通常具有多径传播、时延扩展、噪声复杂等特点，传统均衡方法在复杂环境下可能存在性能或泛化能力限制。

当前研究想法是借鉴物理层 LLM 相关工作中“用 LLM 微调处理通信物理层任务”的思路，但不直接照搬多任务无线 MIMO 设置。本项目聚焦单任务：在水声通信场景下，利用 LLM 或类 LLM backbone 完成序列检测/信道均衡。

核心任务可以概括为：

```text
接收信号 + 辅助信道信息 h_aux
-> LLM 检测/均衡模型
-> 估计发送符号序列
-> 判决/解调
-> BER 评价
```

这里的 LLM 不是用于自然语言生成，而是作为可微调的序列建模骨干网络，用于处理连续通信信号。

其中 `h_aux` 第一版优先使用 `h_true` 做理想条件和链路检查，后续再扩展到 `h_hat` 或信道估计结果。

## 2. 项目目标

第一版代码工程的目标不是一次性完成完整论文实验矩阵，而是先建立一个可运行、可诊断、可复现的最小 BER 闭环。

第一版 MVP 目标链路为：

```text
BPSK symbols
-> 基带水声多径信道
-> rx_bb
-> h_aux，其中第一版使用 h_true
-> baseline / debug model / LLM equalizer with LoRA
-> x_hat
-> BER-SNR 曲线
```

第一版要证明：

- 基础通信链路是正确的。
- 数据生成、Dataset、baseline、模型、训练和评估能闭环。
- BER 计算可信。
- LLM 效果异常时，能定位问题是来自通信链路、数据对齐、baseline、BER evaluator、训练循环，还是模型本身。

## 3. 代码工程概述

工程可以理解为三条主线：

```text
通信链路代码
+ 学习模型代码
+ 诊断/验证代码
```

其中 sanity check 不是附属内容，而是工程主线的一部分。通信系统和学习模型耦合后，最容易出现的问题是最后只看到 BER 不对，却不知道是哪一步错了。因此每一层都要有能单独运行的检查。

建议代码结构如下：

```text
src/
  uwa_llm_equalizer/
    config.py
    signals.py          # bit、BPSK或者其他调制方式、复数实虚转换、判决/解调
    channels.py         # 基带水声信道、噪声、卷积、SNR 控制
    datasets.py         # Dataset、train/val/test split
    baselines.py        # MMSE/FDE、LS+MMSE/FDE
    metrics.py          # BER、MSE、NMSE、BER-SNR 聚合
    models/
      llm_equalizer.py  # 主 LLM 检测/均衡模型
      debug_models.py   # 小 MLP/CNN/Transformer 调试模型
      blocks.py         # 必要时放 adapter、projection、小模块
    training.py         # train_one_epoch / validate
    evaluation.py       # evaluate_model / evaluate_baseline
    plotting.py         # BER-SNR 曲线

scripts/
  run_sanity_check.py # 串联运行基础 sanity check
  train_llm.py        # 训练 LLM equalizer
  evaluate_llm.py     # 评估 LLM equalizer
  run_baseline.py     # 跑传统 baseline
  plot_ber.py         # 绘制 BER-SNR 曲线

tests/
  test_signal_shapes.py       # sample/batch shape 检查
  test_mod_demod.py           # 调制/解调闭环检查
  test_channel_high_snr.py    # 信道与高 SNR 检查
  test_baseline_high_snr.py   # baseline 高 SNR 检查
  test_model_forward.py       # 模型 forward/backward 检查
  test_ber_metric.py          # BER/MSE/NMSE 指标检查

examples/
  quick_mvp.py        # 最小端到端示例
```

其中 `debug_models.py` 中的小 MLP/CNN/Transformer 是工程诊断工具，不是论文主算法。它的作用是先证明数据、loss、训练循环和 BER evaluator 没问题，再进入 LLM 调参。

设计原则：

- 所有模型、baseline、评估函数共享同一套数据入口。
- BER 计算只在统一 metrics/evaluation 中完成，不在各脚本里重复写。
- baseline 输出估计符号，再交给统一 BER evaluator。
- debug model 和 LLM model 使用同一训练循环。
- 每一层完成后先跑对应 sanity check，再进入下一层。

## 4. 实现原则

第一版先跑通：

```text
BPSK
-> 复基带水声信道
-> h_aux，其中第一版使用 h_true
-> LLM + LoRA 检测/均衡
-> MMSE/FDE baseline
-> BER-SNR 曲线
```

暂时不急着展开：

- 多调制方式。
- 信道编码。
- OFDM/S-NOFDM。
- 真实或准真实水声数据。
- 大量深度学习 baseline。
- LoRA 与非 LoRA、不同 LoRA rank、量化和不同 LLM backbone 的系统对比。
- 论文级完整消融实验矩阵。

注意：LoRA 本身属于第一版 LLM 微调方案；暂时不做的是 LoRA 相关系统消融和量化对比。

关键门槛：

```text
baseline 高 SNR 不对 -> 不训练模型
debug model 不能过拟合 -> 不调 LLM
BER evaluator 不可信 -> 不画论文曲线
```

## 5. 分阶段实现计划

### 阶段 0：工程骨架与配置

涉及：

```text
src/uwa_llm_equalizer/config.py
scripts/
tests/
examples/
```

要完成：

- 配置入口：SNR、信道长度、序列长度、batch size、模型参数、随机种子。
- 输出目录：checkpoint、日志、BER 结果、曲线数据。
- 统一 device、dtype、seed 控制。

Sanity check：

- 固定 seed 后，两次生成的 bit、信道、噪声完全一致。
- 修改 SNR 或 seq_len 后，下游 shape 不崩。

完成标准：

- 有一个统一配置入口。
- 后续脚本不需要各自硬编码核心实验参数。
- seed 控制函数可以被数据生成、训练和评估共用。

### 阶段 1：基础信号处理

涉及：

```text
src/uwa_llm_equalizer/signals.py
src/uwa_llm_equalizer/metrics.py
tests/test_mod_demod.py
tests/test_signal_shapes.py
tests/test_ber_metric.py
```

要完成：

- bit 生成。
- BPSK 调制/解调。
- 复数和 `[real, imag]` 表示转换。
- AWGN 加噪相关基础工具。
- BER、MSE、NMSE 计算。

Sanity check：

- `bit -> BPSK -> demod -> bit`，BER 必须为 0。
- 复数转双通道再转回，误差必须接近 0。
- 全对预测 BER 为 0。
- 全反预测 BER 为 1。
- 随机预测的 BPSK BER 大约接近 0.5。

完成标准：

- 基础调制、解调、复数转换和指标函数通过单元测试。
- 后续 baseline、模型和绘图不重复实现 BER 逻辑。

### 阶段 2：水声信道/接收信号生成

涉及：

```text
src/uwa_llm_equalizer/channels.py
tests/test_channel_high_snr.py
```

要完成：

- 生成或加载基带水声信道。
- 对发送符号做多径卷积。
- 加噪生成 `rx_bb`。
- 控制目标 SNR。
- 输出后续 Dataset 可直接使用的样本内容。

Sanity check：

- identity channel 下，`rx_bb` 应等于 `tx_symbols`。
- 无噪声下，理想均衡后 BER 应为 0 或接近 0。
- 高 SNR 下，接收信号与无噪声信号误差很小。
- `rx_bb`、信道辅助信息、`tx_symbols` 长度必须严格对齐。

完成标准：

- 能独立生成一批可用于 Dataset 的基带水声样本。
- SNR 控制、卷积和裁剪规则稳定。
- 生成结果可被后续 baseline 和模型直接使用。

### 阶段 3：Dataset 与数据划分

涉及：

```text
src/uwa_llm_equalizer/datasets.py
tests/test_signal_shapes.py
```

要完成：

- 构建 train/val/test 数据。
- 支持按 SNR 聚合或采样。
- 每个 batch 输出固定字段，供 baseline、debug model 和 LLM 共用。

Sanity check：

- 单个 sample shape 检查。
- batch shape 检查。
- 随机 seed 固定后，数据可复现。
- 同一个样本的 `rx_bb` 和 `tx_symbols` 不能错位。

完成标准：

- train/val/test 可以稳定构建。
- baseline、debug model 和 LLM 使用同一种 batch 输入。
- 数据划分和 SNR 采样逻辑集中在 Dataset 层。

### 阶段 4：传统 baseline

涉及：

```text
src/uwa_llm_equalizer/baselines.py
src/uwa_llm_equalizer/evaluation.py
tests/test_baseline_high_snr.py
scripts/run_baseline.py
```

要完成：

- 先实现理想条件 baseline：`h_true + MMSE/FDE`。
- 后续再接 `h_hat`、LS、ZF 等。
- baseline 输出估计符号，而不是各自单独计算 BER。
- baseline 和 LLM 使用同一个 BER evaluator。

Sanity check：

- identity channel 下 baseline BER 为 0。
- 无噪声或极高 SNR 下 baseline BER 接近 0。
- BER-SNR 曲线大体应随 SNR 增大而下降。

完成标准：

- 能运行 `h_true + MMSE/FDE` baseline。
- baseline 输出估计符号，再由统一 evaluator 计算 BER。
- 得到一条可作为 LLM 对照的 BER-SNR 曲线。

### 阶段 5：集中 sanity check 脚本

涉及：

```text
scripts/run_sanity_check.py
```

要完成：

- 串联运行基础信号、信道、Dataset、baseline 检查。
- 打印清晰分层结果，例如：

```text
[signals] pass
[channel] pass
[dataset] pass
[baseline] pass
```

Sanity check：

- 任一层失败时，明确指出失败模块和关键数值。
- 不需要训练 LLM 也能验证通信链路基本正确。

完成标准：

- 一条命令能完成 signals、channel、dataset、baseline 的基础检查。
- 后续改动通信链路或 baseline 时，可以先用该脚本定位问题。

### 阶段 6：debug model

涉及：

```text
src/uwa_llm_equalizer/models/debug_models.py
tests/test_model_forward.py
```

要完成：

- 加一个小 MLP/CNN/Transformer。
- 输入与 LLM equalizer 保持一致。
- 输出估计符号序列。
- 只用于检查数据、loss、训练循环和 BER evaluator，不作为论文主算法。

Sanity check：

- forward shape 正确。
- 输出无 NaN/Inf。
- 极小数据集上能过拟合。
- loss 能明显下降。

完成标准：

- 至少一个轻量 debug model 可以跑通 forward、loss、backward。
- debug model 能在小数据集上过拟合，用来证明训练闭环可信。

### 阶段 7：LLM equalizer 模型

涉及：

```text
src/uwa_llm_equalizer/models/llm_equalizer.py
src/uwa_llm_equalizer/models/blocks.py
tests/test_model_forward.py
```

要完成：

- 建立主模型类。
- 输入接收信号和辅助信道信息。
- 用 encoder/projection 转成 LLM embedding。
- LLM backbone 使用 LoRA 微调，后接 output projection。
- 输出估计符号序列。
- 第一版不引入复杂 registry，不做多任务。

Sanity check：

- forward 后输出 shape 与目标符号 shape 一致。
- 小 batch forward/backward 能跑通。
- 冻结/微调参数数量可打印。
- 输入、hidden、输出无 NaN/Inf。
- 极小 batch 至少能出现 loss 下降趋势。

完成标准：

- LLM equalizer 可以替换 debug model 进入同一训练流程。
- LoRA 可训练参数和冻结参数能被清楚打印。
- 输出估计符号可以被统一 evaluator 计算 BER。

### 阶段 8：训练流程

涉及：

```text
src/uwa_llm_equalizer/training.py
scripts/train_llm.py
```

要完成：

- `train_one_epoch`。
- `validate`。
- MSE/NMSE loss。
- checkpoint 保存。
- train/val loss 记录。
- val BER 记录。
- 支持选择 `debug_model` 或 `llm_equalizer`。

Sanity check：

- debug model 小数据过拟合成功。
- LLM 小数据 loss 不发散。
- checkpoint 加载后评估结果一致。
- 同 seed 重复运行结果接近。

完成标准：

- 能完成一次小规模 debug model 训练。
- 能完成一次小规模 LLM + LoRA 训练。
- checkpoint、loss、验证 BER 都能保存。

### 阶段 9：统一评估

涉及：

```text
src/uwa_llm_equalizer/evaluation.py
scripts/evaluate_llm.py
```

要完成：

- `evaluate_model`。
- `evaluate_baseline`。
- 按 SNR 聚合 BER。
- 输出统一结果表，例如：

```text
snr_db, ber_model, ber_baseline, mse, nmse
```

Sanity check：

- 相同输入时，evaluation 结果完全可复现。
- BER 聚合前后的 bit/error 总数对得上。
- baseline 和 LLM 结果字段格式一致。

完成标准：

- `evaluate_model` 和 `evaluate_baseline` 输出统一格式。
- BER 可以按 SNR 聚合。
- 绘图脚本可以直接读取评估结果，不重新实现 BER 计算。

### 阶段 10：绘图与结果保存

涉及：

```text
src/uwa_llm_equalizer/plotting.py
scripts/plot_ber.py
```

要完成：

- 读取评估 CSV/JSON。
- 绘制 BER-SNR 曲线。
- 保存图片和原始曲线数据。
- 支持多条曲线：baseline、debug model、LLM。

Sanity check：

- 图中点数等于 SNR 数量。
- 图片和原始数据对应。
- y 轴使用 log scale。
- BER 值不出现负数或大于 1。

完成标准：

- 能生成 BER-SNR 图片。
- 同时保存可复查的原始曲线数据。
- baseline、debug model 和 LLM 曲线可以放在同一张图里比较。

### 阶段 11：MVP 示例入口

涉及：

```text
examples/quick_mvp.py
```

要完成：

- 一键跑小规模流程：

```text
生成数据
-> 跑 sanity check
-> 跑 baseline
-> 训练 debug model
-> 训练/评估 LLM 小样本
-> 输出 BER-SNR
```

Sanity check：

- 在小规模参数下可快速运行。
- 任一步失败时能定位到模块。
- 结果文件路径清晰。

完成标准：

- `quick_mvp.py` 可以作为第一版工程的最小演示入口。
- 它不追求最优 BER，只证明从数据到曲线的闭环成立。

### 阶段 12：后续扩展接口

暂不深入实现，但第一版接口不要堵死：

```text
h_hat 输入
LS + MMSE/FDE
QPSK
OFDM/S-NOFDM
真实/准真实水声数据
更强深度学习 baseline
论文级消融实验
```

完成标准：

- 第一版接口不把数据源、信道辅助信息或模型结构写死。
- 后续可以逐步加入 `h_hat`、LS baseline、QPSK、OFDM/S-NOFDM 和真实水声数据。

## 6. 推荐执行顺序

```text
1. config
2. signals + metrics
3. channels
4. datasets
5. baseline
6. run_sanity_check
7. debug model
8. training
9. llm_equalizer
10. evaluation
11. plotting
12. quick_mvp
```

## 7. 第一版完成标准

第一版代码工程完成后，应能做到：

1. 运行 sanity check，明确每层通过或失败。
2. 生成固定 seed 的可复现实验数据。
3. 跑出可信的 `h_true + MMSE/FDE` baseline BER-SNR 曲线。
4. debug model 能在小数据集上过拟合。
5. LLM equalizer 能跑通 forward、training、evaluation。
6. LLM 和 baseline 使用同一套 BER 评估逻辑。
7. 保存 checkpoint、loss、BER 原始数据和 BER-SNR 曲线图片。

如果 LLM 最终 BER 不理想，也应能区分问题来源：

- 通信链路错误。
- 信道生成或 SNR 控制错误。
- 样本对齐错误。
- baseline 实现错误。
- BER evaluator 错误。
- 训练循环错误。
- 模型能力或结构问题。

## 8. 后续讨论点

MVP 跑通后，再进一步讨论：

- `h_hat` 如何生成或估计。
- LS 信道估计如何并入数据链路和 baseline。
- 是否加入 QPSK。
- 是否引入 OFDM/S-NOFDM。
- 使用哪些真实或准真实水声数据。
- 加入哪些更强 baseline。
- 正式论文实验矩阵如何设计。
- LLM backbone、LoRA、量化、adapter 和 decoder 如何做消融。
