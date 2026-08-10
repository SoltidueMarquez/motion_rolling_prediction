# RPM 可验证复现日志

## 基线与环境

- 日期：2026-08-10；仓库：`facebookresearch/motion_rolling_prediction`；分支：`main`。
- 本轮开始时 HEAD：`67d6536aac43c1efaa9dde3c185fe001d3c4f8e9`。
- Windows 11；NVIDIA GeForce RTX 5070 Ti 16 GB；driver 576.52；compute capability 12.0。
- 独立环境：`D:\Programme\Python\Anaconda3lenvs\rpm`；Python 3.10.16。
- PyTorch：`2.7.0+cu128`；TorchVision `0.22.0+cu128`；TorchAudio `2.7.0+cu128`；CUDA runtime 12.8。
- 环境创建基线：
  ```powershell
  conda create -n rpm python=3.10.16 pip=24.2 setuptools=75.1.0 wheel=0.44.0 -y
  D:\Programme\Python\Anaconda3lenvs\rpm\python.exe -m pip install torch==2.7.0 torchvision==0.22.0 torchaudio==2.7.0 --index-url https://download.pytorch.org/whl/cu128
  ```
- 最终重建配置：`environment.rtx5070ti.yaml`。原 `environment.yaml` 未修改。
- 硬件兼容偏差：官方环境是 PyTorch 2.5.1/CUDA 11.8；RTX 5070 Ti 使用 PyTorch 2.7.0/CUDA 12.8。真实 GPU forward、loss、backward、SMPL-X FK 和 RPM checkpoint forward/backward 均已通过，不能称为与论文软件环境完全一致。

## 外部资产

- `human_body_prior`：commit `78c86eae5ed518ae22bf197fd74211bbfa45551a`，已部署到仓库根并可导入。
- `body_visualizer`：commit `293bb54d1bdb026fe6f89c2def4a030c8f2c2ec3`，已部署到仓库根并可导入。
- SMPL-X locked-head Neutral：用户经官网登录和许可证取得；部署为 `SMPL/smplx/neutral/model.npz`。
- RPM 官方 v0 checkpoint：解压到 `checkpoints/`；A-P1 Reactive 可加载。A-P1 Smooth 官方 `model_latest.pt` 为 0 bytes，属于上游发布资产缺陷，不得伪称已复评。
- 用户提供的原压缩包均保留在项目外，未移动或改写。外部压缩包、已部署模型和关键配置的大小/SHA256 见 `documents/manifests/rpm-reproduction.sha256`；checkpoint 完整清单见 `documents/manifests/rpm-checkpoints-v0.sha256`。

## AMASS Protocol 1 原始数据

- 数据类型：用户经 AMASS 登录和许可证取得的 `SMPL-X N` / `*_stageii.npz`。
- 项目内只读原始数据根：`datasets_raw/amass_p1`，已被 `.gitignore` 忽略。
- 归档目录映射：`BMLrub → BioMotionLab_NTroje`、`CMU → CMU`、`HDM05 → MPI_HDM05`。
- 解压结果：BioMotionLab_NTroje 3061、CMU 1983、MPI_HDM05 215 个 stageii 文件；共 22,029,178,159 bytes。
- 官方 P1 split 覆盖 5251/5251、缺失 0：train 4725，test 526。CMU 归档另含 8 个未进入协议 split 的文件，不擅自加入实验。
- 抽样元数据：`surface_model_type=smplx`、`gender=neutral`、pose 165D、betas 16D、120 FPS。

## 必要代码与路径修复

1. README 把 `--save_dir` 写成 `datasets_processed/amass_p1`，但 Dataset 实际读取 `datasets_processed/amass_p1/new_format_data`；正式配置使用后者。
2. 官方 AMASS 的 `gender` 是 0 维 NumPy ndarray。原 `prepare_data.py` 直接调用 `.upper()` 会失败；当前代码将 ndarray/NumPy scalar/bytes 严格规范成字符串，并保留官方转换算法和 split。
3. PyTorch 2.6+ 默认 `torch.load(weights_only=True)` 无法加载 RPM 转换文件内的 NumPy 元数据与 `SMPLModelType`；`data_loaders/dataloader.py` 仅对可信本地转换产物显式使用 `weights_only=False`，mean/std 仍使用 `weights_only=True`。
4. 不存在第二套转换实现；AMASS 始终由原仓库 `prepare_data.py` 处理。

## 6 样本 pilot 转换与验收

- 用户从 VS Code 运行原 `prepare_data.py`；参数：
  ```text
  --save_dir datasets_processed/amass_p1/new_format_data
  --root_dir datasets_raw/amass_p1
  --splits_dir prepare_data/amass_p1_pilot
  --support_dir SMPL
  --out_fps 60
  ```
- 第一次运行在 `gender.upper()` 失败，未生成序列；修复 NumPy 标量读取后重新运行成功。
- 产物：6/6 个 `.pt`，8392 帧，18,122,102 bytes；每个数据集各一个 train/test。
- 数据契约：local/global motion `(T,132)`、sparse `(T,54)`、world joints `(T,22,3)`、head transform `(T,4,4)`；body parameters 为 `T+1`，首帧用于速度计算；60 FPS；gender/model type/path 均正确。
- 全部张量有限；最大局部旋转正交误差 `4.470348e-06`，最大 determinant 误差 `4.470348e-06`。
- 六个样本用 SMPL-X Neutral 重新 FK；与保存 world joints 的最大绝对误差 `2.384186e-07`。
- 真实训练 batch 成功：motion `(1,70,132)`、motion context `(1,70,132)`、sparse `(1,71,54)`，全部有限。
- 结论：`AMASS_P1_PILOT_VALIDATION=PASS`。六个 `.pt` 编号与完整官方 split 的第一条一致，将由全量转换安全跳过并复用。

## Pilot 后清理

- 已清理一次性 `prepare_data/amass_p1_pilot`、临时 `tests/`、分散的 `documents/logs/`、冗余 Conda export/freeze 快照、Python 缓存及无必要的 `.vscode/settings.json`。
- 已清理项目外重复的 6 文件 pilot raw 副本（44,795,058 bytes）；项目内完整 raw 和三个原始 AMASS 压缩包均保留。
- 两份过时 manifest 已合并为 `documents/manifests/rpm-reproduction.sha256`；正式 checkpoint 完整 manifest 单独保留。
- 上述目录均移入 Windows 回收站；Git 已跟踪文件也可从当前 Git 历史恢复，重复 pilot raw 可从保留的原压缩包重新提取。
- 保留：原项目、必要兼容修复、RPM SKILL、最终环境配置、正式 launch、原始下载数据、完整 raw、6 个有效 `.pt`、官方 checkpoint、精简日志和必要 SHA256。

## 完整 A-P1 转换执行与 CPU 续跑

- 首次 Full 使用 GPU 运行原 `prepare_data.py`，`--splits_dir` 指向官方 `prepare_data/amass_p1`。BioMotionLab_NTroje train/test 全部完成，CMU train 完成前 6 个；连同此前其它 pilot 产物，当前共有 3070 个有效序列 `.pt`。
- GPU Full 在第一个缺失输出 `CMU/train/7.pt`（源 `CMU/01/01_09_stageii.npz`，120 FPS 原始 4242 帧、60 FPS 输出 2121 帧）附近触发 `CUDA out of memory`。异常在 `.cpu()` 报告是 CUDA 异步执行的同步点，实际内存不足发生在前面的整段 SMPL-X forward。
- 该行为符合原 README 对长序列 OOM 的提示；正式恢复策略是不修改转换算法，按 README 加 `--cpu`。`.vscode/launch.json` 现只保留 `RPM: Prepare AMASS P1 Full (CPU Resume)`。
- 转换脚本会按既有编号跳过 3070 个已完成文件，从首个缺失文件继续；不得删除或覆盖已有有效产物。
- 预期总序列 5251；当前已有 3070 个将按编号跳过，剩余 2181 个。
- 根据 pilot 的 raw→processed 比例估计，完整转换数据约新增 8.3 GiB；建议预留 10–12 GiB。记录时 D 盘可用约 1058 GiB。
- GPU 阶段实际用时约 6 分钟后触发 OOM。CPU Resume 会明显更慢，预计剩余转换约 1–4 小时；以实际前 10 个新文件速度重新校准，不改变数据协议。
- 本预算只涉及数据转换，不涉及训练预算。完整转换尚未启动；启动前由用户确认。
- 完整转换通过后，按既定顺序先复评官方 A-P1 Reactive checkpoint，再进行训练 smoke、短训与正式训练预算评估。

## 完整 A-P1 转换验收与可视化

- 用户于 2026-08-10 使用原仓库 `prepare_data.py` 完成 CPU Resume，参数为：
  ```text
  --save_dir datasets_processed/amass_p1/new_format_data
  --root_dir datasets_raw/amass_p1
  --splits_dir prepare_data/amass_p1
  --support_dir SMPL
  --out_fps 60
  --cpu
  ```
- 输出与官方 split 完全一致，共 5251/5251 个 `.pt`、4,347,096 帧、10,368,914,075 bytes（9.657 GiB）：
  - BioMotionLab_NTroje：train 2754 / test 307，1,877,860 帧；
  - CMU：train 1778 / test 197，1,949,175 帧；
  - MPI_HDM05：train 193 / test 22，520,061 帧。
- 对全部 5251 个文件逐一执行深度读取与数据契约检查：路径/编号连续，必需字段齐全，local/global motion `(T,132)`、sparse `(T,54)`、world joints `(T,22,3)`、head transform `(T,4,4)`，body parameters 帧关系正确，60 FPS，SMPL-X/Neutral，所有张量均为有限值，head transform 齐次末行正确。最短 3 帧，最长 11,473 帧。
- 按排序后的相对路径与逐文件内容哈希计算的完整 processed tree SHA256：`5A9392BEBBE5E6273C14EDCF2C8B69A56474558FC7752D986230CE7EA61797C5`。结论：`AMASS_P1_FULL_VALIDATION=PASS`。
- 使用固定随机种子 `20260810`，从三个数据集 test split 各取一条序列、每条取三个时刻，直接绘制已保存的 `position_global_full_gt_world` 骨架；未发现 NaN、爆点、肢体断裂或异常尺度。输出：`results/amass_p1_data_sanity/amass_p1_random_gt_skeletons.png`。
- 项目原生可视化入口已确认：`test.py --vis_gt` 生成 GT SMPL-X MP4，`test.py --vis` 生成 checkpoint 预测 MP4，`--vis_export` 导出 OBJ/JSON；内部调用 `evaluation.visualization.VisualizerWrapper` 和 `utils.utils_visualize.save_animation`。
- Windows 原生 GT 视频 smoke 使用官方可视化子集中的 `BioMotionLab_NTroje-13`：159 帧、60 FPS、800×800、2.65 秒，输出 `results/amass_p1_data_sanity/official_gt/BioMotionLab_NTroje-13_gt.mp4`，并抽取首/中/末帧生成 contact sheet。视频可解码且人体、地面、头/双手标记与坐标轴稳定。
- 为使原生可视化在 Windows + SMPL-X 下真正产出有效帧，保留三项必要兼容修复：Windows 不强制 EGL，face-color 棋盘网格关闭 Pyrender smooth，顶点颜色数量由硬编码 SMPL 6890 改为当前网格实际顶点数（SMPL-X 为 10475）。这些仅影响渲染后端和材质，不改变 AMASS 转换、模型输入或评估数值。
- 可视化产物 SHA256：骨架 PNG `D6D38E3EE1C016B44613C9BFC89D792B9B931BFFAABA22587DE6B6A9D331CFA9`；GT MP4 `A31D588D056E960AFA1981A815C3FA5AD64C8FEB8BA4B19EE4625A7D3A994858`；contact sheet `F10F049176C0662C57C64621F43E077DB5E1548581D8815163C3BB89425D8D0C`。
- 下一步严格保持协议顺序：先复评官方 A-P1 Reactive checkpoint，再进行训练 smoke 与短训；A-P1 Smooth 官方模型文件仍为 0 bytes，不得伪造复评结果。

## 官方 A-P1 Reactive checkpoint 推理 smoke

- 运行日期：2026-08-10；入口：`test.py`；设置：A-P1 test、Motion Controllers（无 gap）、官方 Reactive `model_latest.pt`、seed 10、CUDA device 0、`dataset_max_samples=1`、`eval_batch_size=1`。
- 命令参数显式覆盖本地 `dataset_path`、`support_dir` 与 `results_dir`，模型结构和 rolling 参数从 checkpoint 相邻的官方 `args.json` 恢复：prediction window 10、motion context 10、sparse context 10、free-running 60。
- checkpoint 加载通过，`unexpected keys: []`；模型推理、SMPL-X FK、指标计算与 CSV 写出均成功。评估主体耗时约 1.89 秒。
- 实际样本：`BioMotionLab_NTroje-1`，235 帧。单样本结果：MPJRE `5.472794°`、MPJPE `6.498073 cm`、MPJVE `4.2385435 cm/s`、pred jitter `106.671364 m/s³`、GT jitter `178.31349 m/s³`。
- 输出：`results/official_recheck/results/reactive/latest_rolling/results_amass_p1.csv`，137 bytes，SHA256 `2F0DA8848FB00816547EC662D8707A83B7502420A5D63B53F7D7195BE73633C3`。
- Transformer nested-tensor 与 `torch.cross` 信息均为当前依赖版本的性能/弃用提醒，不影响本次数值链；没有为消除 warning 修改模型或指标。
- 论文 A-P1 MC 的 RPM-Reactive 全测试集参考值为 MPJRE `3.25°`、MPJPE `4.08 cm`、MPJVE `19.21 cm/s`、Jitter `4.21 × 10² m/s³`。当前只有一个样本，不能与论文聚合值直接比较，结论仅为“官方权重推理 smoke 通过”，尚不是论文结果复现。
- 下一步：移除 `dataset_max_samples=1`，先做完整 A-P1 MC 官方权重复评；通过后再做 `eval_gap_config=hand_tracking` 的完整 HT 复评。

## 完整 A-P1 MC 首次尝试 OOM 与重试配置

- 完整 MC 首次使用官方教程建议的 `eval_batch_size=16`；数据加载为 526 条，评估 padding 过滤 3 条，计划评估 523 条（33 batches）。运行至 21/33 完成、进入第 22 batch 的 SMPL-X FK 时触发 CUDA OOM；CSV 只在完整评估结束后写出，因此本次没有不完整 CSV，结果目录为空。
- 第 22 batch 的序列长度范围为 593–696 帧，并非数据集最长序列；这证明报错与 batch=16 时的显存组合峰值有关，不能通过过滤长序列伪装成完整复评。
- 使用官方 checkpoint、相同 A-P1 MC 协议和 CUDA device 0 做无输出隔离诊断：`CMU-80`（696 帧）在 batch=1 时峰值 allocated/reserved 为 `1164.0/1216.0 MiB`；全 test 最长的 `BioMotionLab_NTroje-219`（6746 帧）为 `10187.2/10584.0 MiB`，两者均成功。
- `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 在当前 Windows PyTorch 构建上明确报告不支持，未写入正式配置。
- 正式 Full MC Launch 仅将 `eval_batch_size` 从 16 降为 1；不修改 checkpoint、数据、split、padding、模型、FK 或指标。预计运行时间约 15–30 分钟，输出仍为 `results/official_recheck_full/results/reactive/latest_rolling/results_amass_p1.csv`。

### Batch 1 第二次 OOM 与 CUDA cache 修复

- Batch 1 正式重跑在 339/523、约 3 分 40 秒处再次于 `get_body_poses()` 的 SMPL-X LBS 触发 OOM。其失败位置与 batch 16 的 `21×16≈336` 高度一致，说明降 batch 只改变进度显示，没有消除连续评估中的累积显存问题。
- 排序位置附近的 `CMU-46`（600 帧）、`BioMotionLab_NTroje-133`（620 帧）和 `BioMotionLab_NTroje-106`（632 帧）均在新进程中隔离通过，峰值 allocated 约 1.02–1.07 GiB，排除坏样本和单条序列固有 OOM。
- 原 `evaluate_all()` 连续 20 条的显存采样显示：每条返回后的 live allocated 恒定为 `121.9 MiB`，但 CUDA reserved 随不同序列尺寸从 `254 MiB` 累积到 `678 MiB`。这是未使用缓存块碎片累积，不是仍被引用的模型或 mesh 张量泄漏。
- 必要兼容修复：在 `evaluation/evaluation.py` 的每个 DataLoader batch 完成后，仅当 device 为 CUDA 时调用 `torch.cuda.empty_cache()`。它只归还未使用的缓存块，不改变模型输出、FK、样本、padding 或指标。
- 修复后连续 50 条 smoke 通过：五项指标全部有限，最终 allocated/reserved 为 `121.9/130.0 MiB`，峰值 allocated `298.5 MiB`。结合最长 6746 帧序列的隔离通过，正式 Full MC 继续使用 batch 1，无需过滤任何序列。

## 官方 A-P1 Reactive 完整 MC 复评完成

- 运行日期：2026-08-10；VS Code 配置：`RPM: Step 1 - Official A-P1 Reactive Full MC Eval`。
- 固定协议：官方 A-P1 Reactive `model_latest.pt`、A-P1 `test`、Motion Controllers（无 gap）、完整数据 `dataset_max_samples=-1`、CUDA device 0、`eval_batch_size=1`、seed 10；本地数据、SMPL-X 和结果路径均由命令行显式覆盖。
- 加载 test 526 条；`SortedSampler` 按 A-P1 的 60 帧评估 padding 加 jerk 所需 3 帧门槛过滤 3 条，完整评估 523 条。评估主体耗时 13 分 41 秒。
- 逐序列 CSV 为 523 行、7 列；523 个 filename 全部唯一、无重复行，MPJRE/MPJPE/MPJVE/pred jitter/GT jitter 均为 523/523 有限值。
- 独立重算未四舍五入均值：MPJRE `3.253503732256°`、MPJPE `4.080012594302 cm`、MPJVE `19.209780292352 cm/s`、pred jitter `421.201718317400 m/s³`、GT jitter `371.263651902486 m/s³`。
- 按论文单位与两位小数报告：MPJRE `3.25°`、MPJPE `4.08 cm`、MPJVE `19.21 cm/s`、Jitter `4.21 × 10² m/s³`；四项均与论文 Table 1 的 A-P1 MC / RPM-Reactive 对齐到公布精度。
- 输出：`results/official_recheck_full/results/reactive/latest_rolling/results_amass_p1.csv`，37,075 bytes，SHA256 `3CCCADDF6C3142406F915D46172C81D5609BA6ED0660C10F9B9A9481EA55FDBA`。
- 结论：`A_P1_REACTIVE_FULL_MC_OFFICIAL_RECHECK=PASS`。CUDA cache 兼容修复只释放未使用缓存，没有改变 checkpoint、样本、模型输出、SMPL-X FK 或指标公式。

## 官方 A-P1 Reactive 完整 HT 待运行配置

- 新增 VS Code 配置：`RPM: Step 2 - Official A-P1 Reactive Full HT Eval`。
- 与已验收 MC 严格保持同一 checkpoint、A-P1 test split、完整样本、CUDA device 0、batch 1 和 seed 10；唯一任务协议变化是增加 `--eval_gap_config hand_tracking`。
- `test.py` 会自动给 gap 评估输出追加 `_hand_tracking`，因此与 MC 共用 `results/official_recheck_full` 根目录也不会覆盖 MC。预期输出目录：`results/official_recheck_full/results/reactive/latest_rolling_hand_tracking/`。
- 验收内容：523 条逐序列 CSV 的完整性与有限值；MPJRE、MPJPE、MPJVE、pred/GT Jitter；T→S/S→T jerk 数组和图；两向 PJ/AUJ；所有正式产物 SHA256。
- 论文 A-P1 HT / RPM-Reactive 对照：MPJRE `3.82°`、MPJPE `5.18 cm`、MPJVE `22.83 cm/s`、Jitter `4.35 × 10² m/s³`、PJ T→S `15.28`、AUJ T→S `60.51`、PJ S→T `18.98`、AUJ S→T `69.02`。
- 本节只记录可重复执行配置，尚未记录 HT 运行结果；用户完成 Launch 后再据真实 stdout 和输出文件更新结论。
