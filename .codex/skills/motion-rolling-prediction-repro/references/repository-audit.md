# RPM 仓库审计参考

## 审计快照

- 审计日期：2026-08-10
- 仓库：`facebookresearch/motion_rolling_prediction`
- 本地 commit：`67d6536aac43c1efaa9dde3c185fe001d3c4f8e9`（2026-08-10 AMASS 放置前的 HEAD）
- 分支：`main`
- 工作树：创建本 SKILL 前为 clean
- 当前机器：Windows、NVIDIA GeForce RTX 5070 Ti、16 GB 显存、driver 576.52、compute capability 12.0

若 commit、机器或关键文件发生变化，重新核对本参考，不把该快照当成永久事实。

## 当前仓库已有与缺失内容

已有：

- 训练、测试、评估和 AMASS/GORP 转换代码；
- A-P1/A-P2 split 文件；
- A-P1/A-P2 的 Hand-Tracking gap JSON；
- A-P1/A-P2/GORP 的论文 mean/std 文件。
- 独立 `rpm` Conda 环境（Python 3.10.16、PyTorch 2.7.0+cu128），RTX 5070 Ti 真实 forward/backward 已通过；
- `human_body_prior/`、`body_visualizer/`、`SMPL/smplx/neutral/model.npz` 与官方 v0 checkpoints；
- A-P1 所需三套已授权 AMASS `SMPL-X N` 原始数据，位于 `datasets_raw/amass_p1`，官方 split 覆盖 5251/5251；
- 完整 A-P1 转换与逐文件验收已完成：5251/5251 个序列、4,347,096 帧、9.657 GiB；processed tree SHA256 已记录。
- 首次 pilot 暴露的 AMASS `gender` 0 维 ndarray 兼容问题已修复；临时测试在结论写入正式日志后按清理规则移除。
- Full GPU 转换曾在 3070/5251 后因整段 SMPL-X forward 显存不足中止，随后按 README 使用 CPU Resume 完成；原始数据与转换算法未改。
- 官方 A-P1 Reactive checkpoint 的 MC 单样本 smoke 与完整 MC 复评均已通过。完整评估加载 526 条、按官方最短长度规则评估 523 条；四项论文指标与 Table 1 对齐到公布精度，CSV、耗时与 SHA256 已记录。

缺失或尚未执行：

- 完整 A-P2 train/test `.pt`；
- A-P1 Reactive 官方 checkpoint 的完整 Hand-Tracking 复评；完整 MC 已完成；
- 训练 smoke、短训、正式训练和完整测试；
- 官方 A-P1 Smooth `model_latest.pt` 为 0 bytes，属于上游发布资产缺陷。

顶层 `.gitignore` 已忽略上述外部源码目录、SMPL、checkpoints、results、`datasets_raw/`，以及 `datasets_processed/*/new_format_data/*/` 下的转换数据。

## 官方外部资产

| 资产 | 官方来源 | 当前已知信息 | 目标位置 |
|---|---|---|---|
| RPM checkpoint | `https://github.com/facebookresearch/motion_rolling_prediction/releases/tag/v0` | release v0；`rpm_checkpoints.zip`；882,648,883 bytes；发布于 2025-06-05 | 解压到仓库根的 `checkpoints/` |
| human_body_prior | `https://github.com/nghorbani/human_body_prior` | 需要其 Python 包内容能作为 `human_body_prior.*` 导入 | `human_body_prior/` |
| body_visualizer | `https://github.com/nghorbani/body_visualizer` | 需要其 Python 包内容能作为 `body_visualizer.*` 导入 | `body_visualizer/` |
| SMPL-X neutral | `https://smpl-x.is.tue.mpg.de/` | 注册下载 “SMPL-X with removed head bun (NPZ)”；受许可证约束 | `SMPL/smplx/neutral/model.npz` |
| AMASS | `https://amass.is.tue.mpg.de/download.php` | 注册下载 `SMPL-X N` 数据；受许可证约束 | 可位于仓库外，转换时以只读 `--root_dir` 引用 |

下载后补录具体 commit、SHA256 和解压内容。不要把依赖仓库的整个 `src/` 目录机械改名；最终必须满足代码中的实际 import，并避免得到 `human_body_prior/src/human_body_prior` 这种多一层的路径。

## 环境事实与兼容风险

- 实际文件是 `environment.yaml`，README 命令写成了 `environment.yml`。
- 官方锁定 Python 3.10.16、PyTorch 2.5.1、`pytorch-cuda=11.8`，并包含较多 Windows build string。
- 当前 GPU compute capability 为 12.0。创建环境前必须核对 PyTorch 构建是否支持它，并用真实 CUDA forward/backward 验证；仅 `torch.cuda.is_available()==True` 不足以证明可运行。
- 若需要换用较新的 PyTorch/CUDA 构建，其他 Python 包优先保持官方版本，并在复现日志中单列这项偏差。

## AMASS 协议与路径

### Protocol 1

- 输出帧率：60 FPS
- split 子集及行数：
  - `BioMotionLab_NTroje`：train 2754，test 307
  - `CMU`：train 1778，test 197
  - `MPI_HDM05`：train 193，test 22
- 论文/README 常写 BMLrub、CMU、HDM05；实际目录与 split 名必须按上表和 split 文件。

### Protocol 2

- 输出帧率：30 FPS
- train：ACCAD、BioMotionLab_NTroje、BMLmovi、CMU、EKUT、Eyes_Japan_Dataset、KIT、MPI_HDM05、MPI_Limits、MPI_mosh、SFU、TotalCapture
- test：HumanEva、Transitions_mocap

### 关键保存路径修正

`prepare_data.py` 直接写入 `<save_dir>/<subset>/<train|test>/<index>.pt`，而 Dataset 固定读取：

```text
<dataset_path>/<dataset>/new_format_data/<subset>/<train|test>/*.pt
```

因此正确命令模板是：

```powershell
conda run -n rpm python prepare_data.py `
  --save_dir .\datasets_processed\amass_p1\new_format_data `
  --root_dir <AMASS_SMPLX_N_ROOT> `
  --splits_dir .\prepare_data\amass_p1 `
  --support_dir .\SMPL `
  --out_fps 60
```

官方教程把 `--save_dir` 写成 `./datasets_processed/amass_p1`，这与当前 dataloader 不兼容。

转换脚本已存在目标文件时会跳过。Pilot split 只能使用原 split 的有序前缀，这样完整转换时相同编号仍指向相同原始序列。

## 数据张量契约

- 模型动作输出：22 关节局部 6D rotation，132D。
- 三点稀疏输入：Head、Left Hand、Right Hand；每点包含 rotation/rotational velocity/position/position velocity，总计 54D。
- 处理文件还应包含 world joints、head world transform、body parameters、framerate、gender、filepath 和 surface model type。
- Dataloader 读取根目录是 `--dataset_path`，通常传 `./datasets_processed`，而不是传到 `new_format_data`。
- A-P1 test 前 60 帧作为评估左侧 padding；A-P2 是 30 帧。

## Checkpoint 与测试 CLI 陷阱

`test.py` 会从 checkpoint 相邻的 `args.json` 恢复模型参数。若命令行没有显式给 `--dataset`，还会恢复作者机器上的 `support_dir`、`dataset_path` 和 `results_dir`。

官方 checkpoint 的安全评估模板应显式覆盖本地路径：

```powershell
conda run -n rpm python test.py `
  --model_path .\checkpoints\amass_p1\reactive\model_latest.pt `
  --dataset amass_p1 `
  --dataset_path .\datasets_processed `
  --support_dir .\SMPL `
  --results_dir .\results\official_recheck `
  --dataset_max_samples 1 `
  --eval --eval_batch_size 1
```

Hand-Tracking 在上述命令后增加：

```text
--eval_gap_config hand_tracking
```

Pilot 成功后移除 `--dataset_max_samples 1` 并按显存提高 batch。当前 Windows/RTX 5070 Ti 的完整评估固定使用 `eval_batch_size=1`；MC 与 HT 使用同一个 `results_dir` 时，当前 `test.py` 会分别写入 `latest_rolling` 与 `latest_rolling_hand_tracking`，不会互相覆盖。

## 训练事实

- `train.py` 默认 `num_steps=100000`，教程命令不显式传该参数，因此属于长训练。
- 论文 A-P1 Reactive 关键参数：prediction window 10、free-running 60、motion context 10、sparse context 10、velocity/FK/FK-velocity loss 均为 1。
- A-P1 Smooth 将 prediction window 改为 20。
- 教程 batch 512 不适合作为当前 16 GB GPU 的无条件默认值；先预检再决定。
- `args.json` 位于 `<results_dir>/checkpoints/<exp_name>/`，必须与 checkpoint 一起保留。

## 评估输出与论文对照

代码默认基础指标：

- `mpjre`
- `mpjpe`
- `mpjve`
- `pred_jitter`
- `gt_jitter`

启用 Hand-Tracking gap 后还会生成 S→T/T→S jerk 数组、图、PJ 和 AUJ。逐序列结果保存为 CSV。

相关工作汇总记录的论文 A-P1 Hand-Tracking RPM-Reactive 参考值为：

- MPJRE 3.82°
- MPJPE 5.18 cm
- MPJVE 22.83 cm/s
- Jitter 4.35 × 10² m/s³
- PJ tracking→synthesis 15.28 × 10² m/s³
- AUJ tracking→synthesis 60.51 × 10² m/s²
- PJ synthesis→tracking 18.98 × 10² m/s³
- AUJ synthesis→tracking 69.02 × 10² m/s²

这些是论文目标，不是宽松通过阈值。只有模型、A-P1 数据、Hand-Tracking gap、评估 padding、单位和后处理一致时才可直接比较。

## 推荐执行顺序

1. 建立复现日志并记录机器/commit。
2. 配置独立环境，完成 CPU/CUDA smoke。
3. 补齐两个外部 Python 包，完成 import smoke。
4. 经用户授权放置 SMPL-X neutral，并完成 BodyModel/FK smoke。
5. 下载并校验官方 checkpoint archive。
6. 发现或取得 A-P1 `SMPL-X N` 原始数据。
7. 用原 split 前缀做少量转换和 Dataset smoke。
8. 扩展为完整 A-P1 转换。
9. 先复评官方 Reactive MC，再复评 Hand-Tracking；随后可评 Smooth。
10. 做 1～10 step 训练 smoke、小样本过拟合和短训。
11. 用户确认计算预算后才开始正式 100k 训练与完整评估。

## 下一步待验证

- A-P1 Full CPU Resume 已完成并逐文件验收：5251/5251 个序列、4,347,096 帧、9.657 GiB，processed tree SHA256 为 `5A9392BEBBE5E6273C14EDCF2C8B69A56474558FC7752D986230CE7EA61797C5`。
- 三数据集随机骨架检查与项目原生 SMPL-X GT MP4 smoke 均通过；Windows/Pyrender/SMPL-X 必要兼容差异已记录在 `documents/reproduction-log.md`。
- 官方 A-P1 Reactive checkpoint 的完整 MC 已于 2026-08-10 使用 batch 1 评估完成：526 条输入中按官方 63 帧门槛过滤 3 条，523 条逐序列结果全部有限且文件名唯一；耗时 13 分 41 秒。聚合值为 MPJRE `3.25°`、MPJPE `4.08 cm`、MPJVE `19.21 cm/s`、Jitter `4.21×10² m/s³`，与论文 Table 1 对齐到公布精度。CSV SHA256 为 `3CCCADDF6C3142406F915D46172C81D5609BA6ED0660C10F9B9A9481EA55FDBA`。
- `.vscode/launch.json` 已提供 `RPM: Step 2 - Official A-P1 Reactive Full HT Eval`：保持完整 MC 的 checkpoint、test split、batch 1、device 0 与 seed 10，仅增加 `--eval_gap_config hand_tracking`；预期输出位于 `results/official_recheck_full/results/reactive/latest_rolling_hand_tracking/`。下一步由用户运行该配置，再验收 HT 的 CSV、PJ/AUJ 数组、图和论文指标；随后进行训练 smoke 和短训。
- A-P1 Smooth 因官方 `model_latest.pt` 为 0 bytes 暂不可复评，不得以 optimizer state 替代模型权重。
