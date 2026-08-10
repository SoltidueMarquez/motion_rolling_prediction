---
name: motion-rolling-prediction-repro
description: 面向 facebookresearch/motion_rolling_prediction（RPM）的可验证复现规范。用于在 Windows 或 Linux 上审计并部署 RPM、配置独立 Conda/PyTorch/CUDA 环境、补齐 human_body_prior/body_visualizer/SMPL-X/官方 checkpoint 等外部资产、把 AMASS Protocol 1/2 转换为 RPM 数据、运行 pretrained 或自行训练的模型、执行 MC/Hand-Tracking 测试与 MPJRE/MPJPE/MPJVE/Jitter/PJ/AUJ 评估，以及定位 README、数据路径、checkpoint 参数和论文协议之间的差异。
---

# RPM 复现与 AMASS 实验

## 目标

把 RPM 从“代码已经下载”推进到“环境、数据、模型、测试和指标均有证据可核对”的状态。优先完成 AMASS Protocol 1（A-P1）与官方 checkpoint 的闭环验证，再进行短训和正式训练。

## 开始前必须完成

1. 将工作目录切到 RPM 仓库根目录，读取仓库内 `AGENTS.md`（若存在）、`README.md`、`tutorial/README.md`、`environment.yaml` 和当前 `git status`。
2. 涉及部署、AMASS、训练或评估时，完整读取 [references/repository-audit.md](references/repository-audit.md)。若当前 commit 与参考文件记录不同，先重新审计相关入口并更新参考文件。
3. 检查并保护用户已有改动；不回滚、不覆盖、不清理未知产物。
4. 首次汇报使用以下顺序：`复现目标 → 当前仓库与机器 → 缺失文件/依赖 → 拟改文件 → 执行顺序 → 验证方式`。汇报后继续完成安全的环境与 smoke 阶段；不要停留在建议。
5. 建立 `documents/reproduction-log.md`，逐阶段记录 commit、命令、参数、环境、输入/输出路径、SHA256、耗时、指标和结论。不存在 `documents/` 时可以创建。

## 证据优先级

按以下顺序判断真实契约：

1. 当前 commit 的代码、checkpoint 同目录的 `args.json`、生成文件和原始评估输出；
2. 官方仓库、官方 release 和官方数据/模型页面；
3. 论文正文与补充材料；
4. README、教程和二手汇总。

README 命令与代码不一致时，以代码为准并记录差异。不得为了接近论文数字而静默改变数据划分、帧率、人体模型、输入缺失协议、指标单位或后处理。

## 阶段 1：环境与 CUDA 门槛

1. 创建 RPM 专用 Conda 环境，不修改或复用 DiffusionPoser 环境。
2. 先记录操作系统、GPU、驱动、显存、Conda、Python、PyTorch、CUDA runtime 和 compute capability。
3. 官方环境基线是 Python 3.10、PyTorch 2.5.1、CUDA 11.8；仓库文件名实际为 `environment.yaml`。不要照抄 README 中不存在的 `environment.yml`。
4. 先判断官方 PyTorch 构建是否支持当前 GPU。若 CUDA kernel smoke 失败，保留 Python 3.10 和其余依赖约束，改用 PyTorch 官方提供且支持当前 GPU compute capability 的 CUDA 构建；把这一项标成“硬件兼容偏差”，不要伪称完全一致环境。
5. 环境验证至少包括：
   - `import torch`、`torch.cuda.is_available()`、GPU 上矩阵乘法与反向传播；
   - `numpy==1.26.x` 与关键库导入；
   - RPM 自身 `train.py --help`、`test.py --help`（补齐外部模块后）；
   - 一个只构建模型、不训练的 CPU/GPU forward smoke。
6. 环境创建失败时保留完整 solver 日志，不在 base 环境逐包试装。

## 阶段 2：外部资产台账

下载或复用任何资产前，记录 `名称、官方来源、版本/commit、许可证、大小、SHA256、目标路径、是否必需`。

必须核对：

- `human_body_prior` Python 包应能从仓库根目录导入为 `human_body_prior.*`；
- `body_visualizer` 应能导入为 `body_visualizer.*`；
- RPM 论文复现需要经授权取得的 SMPL-X neutral NPZ，目标为 `SMPL/smplx/neutral/model.npz`；
- 官方 v0 checkpoint 压缩包必须来自 RPM GitHub release；解压后每个模型 checkpoint 旁必须存在匹配的 `args.json`；
- AMASS 数据受注册与许可约束。优先只读复用用户已有原始 AMASS，不复制、不改写原始 `.npz`。

SMPL-X 和 AMASS 需要用户账号/许可时，明确列出需要用户手工完成的动作和期望路径；不得绕过登录、许可或访问控制。大型下载在用户未明确授权时，先报告大小与落盘位置。

所有外部源码、模型、checkpoint、转换数据和训练输出保持 ignored，不提交到 Git。

## 阶段 3：AMASS 转换

默认先复现 A-P1，再决定是否扩展 A-P2。

1. 校验原始 AMASS 是否为 RPM 教程要求的 `SMPL-X N` / `*_stageii.npz`，并检查 split 文件中抽样路径真实存在。
2. 不凭数据集宣传名猜目录名；严格使用 `prepare_data/amass_p1` 或 `prepare_data/amass_p2` 中的相对路径。
3. `prepare_data.py` 的 `--save_dir` 必须指向：
   - P1：`datasets_processed/amass_p1/new_format_data`
   - P2：`datasets_processed/amass_p2/new_format_data`
   教程少写了 `new_format_data`；按教程原命令生成的数据不会被 `data_loaders/dataloader.py` 找到。
4. 先用原 split 文件的前缀创建小型 pilot split，每个子集只取少量 train/test 文件。必须保留原顺序，使之后用完整 split 继续转换时编号一致并安全复用已生成文件。
5. Pilot 验证：
   - 无缺失路径或不受控跳过；
   - 输出 `.pt` 可加载，帧率与协议一致；
   - 动作是 22 关节 × 6D = 132D，稀疏头手条件是 54D；
   - Head transform、world joints、body params、gender、model type、filename 等字段存在且长度一致；
   - 6D rotation 可还原为有限旋转矩阵，FK 输出有限；
   - 训练 Dataset 能读取一个 batch。
6. Pilot 通过后才运行完整转换。转换结束要报告 split/subset 的期望数、成功数、缺失数、跳过数、总帧数、空间占用和统计文件 SHA256。
7. 仓库已带论文使用的 mean/std；先验证并保留其哈希。除非协议明确要求，不因 pilot 数据较少而覆盖它们。

### Pilot 通过后的强制清理与目录整洁

6 样本 pilot 完成并通过本节全部验证后，必须先清理复现过程中产生的临时调试/测试资产，再进入完整 A-P1 转换。不得让一次性辅助文件长期堆积在项目中。

1. 清理前先用 `git status`、上游基线和文件清单区分原项目文件、用户文件与本次新增资产；只清理由本复现流程创建且已不再需要的精确目标，不删除未知文件。
2. 必须清理的临时资产包括：
   - `prepare_data/amass_p1_pilot` 等一次性 pilot split；
   - 仓库外重复提取的 pilot 原始数据副本；
   - 仅用于定位问题的临时脚本、一次性 smoke/unit test、debug 配置和缓存；
   - 已汇总进正式复现日志后的碎片化临时日志、重复 manifest 和中间检查输出；
   - 失败或不完整且不会被正式转换复用的输出。
3. 应保留的内容包括：
   - 原 RPM 项目代码与官方 split/mean/std/gap 配置；
   - 经真实失败证明为必要的最小兼容修复；
   - 最终可复现环境配置与正式运行配置；
   - 用户授权下载的原始 AMASS、SMPL-X、外部依赖和官方 checkpoint；
   - 已验证且编号与官方 split 一致、可被完整转换安全复用的 pilot `.pt`；
   - 完整转换数据、评估结果、训练 checkpoint，以及一份精简的正式复现日志和必要 SHA256 清单。
4. 不保留第二套数据转换实现。正式转换必须继续使用仓库的 `prepare_data.py`；pilot 通过后只把 `--splits_dir` 从 pilot split 切换到官方 `prepare_data/amass_p1`，不得增加 wrapper、复制版转换脚本或并行的自定义预处理管线。
5. 如果临时测试覆盖了必须保留的兼容修复，先把失败现象、修复内容、验证命令与结果汇总进正式复现日志，再删除临时测试文件。测试结论必须可追溯，但测试脚本不因调试方便而默认长期保留。
6. 清理后执行目录验收：确认原始下载数据、正式/可复用处理数据、必要代码和最终配置仍存在；确认无 pilot split、重复 pilot raw、临时测试脚本、缓存或不完整输出；执行 `git diff --check` 和最终 `git status`，报告删除清单及可恢复性。
7. 此清理授权只适用于能够明确归因于本复现流程的临时资产；原始 AMASS/SMPL-X、用户已有文件、DiffusionPoser、正式转换数据和未知产物仍受保护。目标不确定时必须停止并向用户确认。

## 阶段 4：先验证官方 checkpoint

在自行训练前，先证明数据、模型和评估链能共同工作。

1. 校验 release 压缩包、解压目录、checkpoint 和相邻 `args.json`。
2. 评估时显式传入 `--dataset`、`--dataset_path`、`--support_dir` 和 `--results_dir`。否则 `utils/parser_util.py` 可能从作者的 `args.json` 恢复旧机器路径。
3. 先使用 `--dataset_max_samples 1` 或少量样本跑通；再运行完整 A-P1 MC 和 `--eval_gap_config hand_tracking`。
4. MC 与 Hand-Tracking 应复用同一组 checkpoint、split、seed、batch 和 GPU，仅让 HT 增加 `--eval_gap_config hand_tracking`。当前 `test.py` 会把 HT 输出目录自动加上 `_hand_tracking` 后缀，因此同一 `results_dir` 下的 MC `latest_rolling` 与 HT `latest_rolling_hand_tracking` 可以并存；运行前仍须确认目标目录，禁止覆盖已验收结果。
5. 固定 seed、checkpoint、split、batch、输入模式和 GPU；保存 stdout、CSV、NPZ、曲线图及其 SHA256。
6. 至少报告 MPJRE、MPJPE、MPJVE、pred/GT Jitter。Hand-Tracking 还要报告 S→T/T→S 的 PJ、AUJ，并保留逐序列 CSV。
7. A-P1 Reactive 的论文对照向量为：MC `3.25° / 4.08 cm / 19.21 cm/s / 4.21×10² m/s³`；HT `3.82° / 5.18 cm / 22.83 cm/s / 4.35×10² m/s³`，且 `PJ T→S=15.28`、`AUJ T→S=60.51`、`PJ S→T=18.98`、`AUJ S→T=69.02`。这些是精确协议下的对照目标，不是宽松阈值。
8. 单位必须从 `utils/metrics.py` 与论文共同确认；未经确认不要把代码打印值手工乘除后写成论文指标。

## 阶段 5：训练门槛

按以下顺序逐级扩大，不直接启动教程中的完整训练：

1. 真实 batch forward/backward：loss 与梯度有限，模型可保存/恢复。
2. 1～10 step smoke：检查显存、速度、checkpoint 和 `args.json`。
3. 小样本过拟合：确认 loss 确实下降，测试脚本能加载该 checkpoint。
4. 短训 pilot：固定 seed、数据和配置，评估 MC/HT 两种条件。
5. 用户确认预算后，才执行完整 100k 级训练。

Reactive 与 Smooth 分开记录。不得只改 `exp_name`：必须核对 `input_motion_length`、`rolling_fr_frames`、motion/sparse context、loss 权重及 checkpoint `args.json`。batch size 512 是论文训练配置而非本机必须值；按显存预检选择可运行 batch，并明确记录与论文的差异。

## 阶段 6：复现结论

最终交付至少回答：

- 环境与官方环境哪些一致、哪些因硬件发生偏差；
- 下载了什么、哪些资产由用户授权提供、各自哈希是什么；
- AMASS 使用 P1 还是 P2、具体子集/帧率/split/成功数量；
- 测试使用官方 checkpoint 还是自训 checkpoint；
- MC 与 Hand-Tracking 的完整指标和输出路径；
- 与论文数字的差值，以及差异最可能来自环境、数据、模型还是协议；
- 是否达到“代码跑通”“官方权重复评成功”“训练可收敛”“论文主结果复现”中的哪一级。

只有数据协议、checkpoint、输入缺失方式、指标实现与论文一致时，才能称为论文结果复现；否则使用“工程跑通”或“近似复评”。

## 变更边界

- 优先修复可验证的 Windows/路径/CLI 问题，不重写 RPM 模型。
- 不修改原始 AMASS、SMPL-X 或 DiffusionPoser 项目。
- 不静默修改官方 split、mean/std、gap JSON 或评估公式。
- 代码修复必须配套轻量 smoke test；跨数据/训练/评估契约时运行完整 smoke。
- 长训练、全量转换和多 GB 下载不是环境 smoke 的默认动作。
