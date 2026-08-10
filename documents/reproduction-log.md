# RPM 可验证复现日志

## 基线

- 日期：2026-08-10
- 仓库：`facebookresearch/motion_rolling_prediction`
- 分支：`main`
- Commit：`64fe8f18101713814d4a552202f187c5604303ef`
- 操作系统：Windows 11 专业版 64 位（Build 22631）
- GPU：NVIDIA GeForce RTX 5070 Ti，16303 MiB
- NVIDIA driver：576.52
- Compute capability：12.0
- Conda：25.5.1
- 环境状态：独立 `rpm` 环境已创建并通过 CUDA、SMPL-X 和 RPM checkpoint smoke。

## 外部资产

### human_body_prior

- 状态：文件级部署完成；`rpm` 环境中 `BodyModel` 导入与 CPU/GPU smoke 已通过。
- 官方来源：`https://github.com/nghorbani/human_body_prior`
- 固定 commit：`78c86eae5ed518ae22bf197fd74211bbfa45551a`
- 原始下载文件：`D:\Desktop\动画项目\human_body_prior-78c86eae5ed518ae22bf197fd74211bbfa45551a.zip`
- 压缩包大小：8,565,276 bytes
- 压缩包 SHA256：`B92B3EFA4AA1AF0EC84D2634FE2CD6B28D26B7AE579CEB2D72B5182931222F3F`
- 许可证：上游自定义非商业科学研究许可证；许可证副本保存为 `human_body_prior/LICENSE`。
- 目标路径：`D:\Desktop\动画项目\motion_rolling_prediction\human_body_prior`
- 部署方式：仅复制压缩包中的 `human_body_prior/` Python 包，并补入上游 `LICENSE`；没有把上游仓库根目录整体嵌套进目标目录。
- 目标文件：34 个，共 1,022,662 bytes。
- 结构验证：`__init__.py`、`body_model/body_model.py`、`tools/rotation_tools.py` 和 `LICENSE` 均存在。
- 安全验证：压缩包 81 个条目，单一根目录，未发现绝对路径或 `..` 路径穿越条目。
- Git 状态：目标目录命中仓库现有 `.gitignore` 的 `human_body_prior` 规则。
- 环境验证：`human_body_prior.body_model.body_model.BodyModel` 导入成功；SMPL-X CPU/GPU FK 的形状、有限值与梯度验证通过。

### body_visualizer

- 状态：文件级部署完成；`rpm` 环境中可视化模块导入验证已通过。
- 官方来源：`https://github.com/nghorbani/body_visualizer`
- 固定 commit：`293bb54d1bdb026fe6f89c2def4a030c8f2c2ec3`
- 原始下载文件：`D:\Desktop\动画项目\body_visualizer-293bb54d1bdb026fe6f89c2def4a030c8f2c2ec3.zip`
- 压缩包大小：152,847 bytes
- 压缩包 SHA256：`761DB0E8BEDF7B3F114220EAE7B8050466D138189CB5537E307B01B65F4BFE35`
- 许可证：上游自定义非商业科学研究许可证；许可证副本保存为 `body_visualizer/LICENSE`。
- 目标路径：`D:\Desktop\动画项目\motion_rolling_prediction\body_visualizer`
- 部署方式：仅复制压缩包中的 `body_visualizer/` Python 包，并补入上游 `LICENSE`；没有把上游仓库根目录整体嵌套进目标目录。
- 目标文件：13 个，共 41,349 bytes。
- 结构验证：`__init__.py`、`mesh/mesh_viewer.py`、`tools/vis_tools.py` 和 `LICENSE` 均存在。
- 安全验证：压缩包 36 个条目，单一根目录，未发现绝对路径或 `..` 路径穿越条目。
- Git 状态：目标目录命中仓库现有 `.gitignore` 的 `body_visualizer` 规则。
- 环境验证：`body_visualizer.mesh.mesh_viewer.MeshViewer` 与 Pyrender/Trimesh 导入成功；本阶段不生成可视化产物。

### SMPL-X Neutral（locked head / NPZ）

- 状态：文件级部署完成；`BodyModel` CPU/GPU FK 数值验证已通过。
- 官方来源：`https://smpl-x.is.tue.mpg.de/download.php`
- 授权来源：由用户从需要登录及接受许可证的 SMPL-X 官方下载页提供；不复制、公开或提交模型文件。
- 上游压缩包：`smplx_lockedhead_20230207.zip`
- 原始下载文件：`D:\Desktop\动画项目\smplx_lockedhead_20230207.zip`
- 压缩包大小：411,448,739 bytes
- 压缩包 SHA256：`88D35123FC97151BC258DC6F28F3967AE4C16117E4B452445F20598C43C43099`
- 压缩包内容：Female、Male、Neutral 三个 NPZ；RPM 仅提取 `models_lockedhead/smplx/SMPLX_NEUTRAL.npz`。
- 目标路径：`D:\Desktop\动画项目\motion_rolling_prediction\SMPL\smplx\neutral\model.npz`
- 目标文件大小：137,106,406 bytes
- 目标文件 SHA256：`43D8F3A1375D7C5BAAE207870A5D51DEF0F7E6B507DF709B4937598B5E7D965D`
- 安全验证：源 ZIP 共 6 个条目，未发现绝对路径或 `..` 路径穿越条目；原始 ZIP 未移动或修改。
- NPZ 结构验证：28 个数组；`v_template`、`shapedirs`、`posedirs`、`J_regressor`、`weights`、`kintree_table`、`f` 均存在。
- Git 状态：目标文件命中仓库现有 `.gitignore` 的 `SMPL` 规则。
- 数值验证：CPU/GPU 零姿态输出 vertices `(1, 10475, 3)`、joints `(1, 55, 3)`，全部有限；GPU pose backward 梯度有限。

### RPM 官方 v0 checkpoints

- 状态：文件级部署完成；A-P1 Reactive 加载与合成输入 forward/backward smoke 已通过，真实数据复评待 AMASS P1 pilot；A-P1 Smooth 存在官方资产缺陷。
- 官方来源：`https://github.com/facebookresearch/motion_rolling_prediction/releases/tag/v0`
- 原始下载文件：`D:\Desktop\动画项目\RPM相关材料\rpm_checkpoints.zip`
- 压缩包大小：882,648,883 bytes
- 压缩包 SHA256：`A0848B33BD3162EAABECE5C8D8EF79EA8C3499ADE9688C681941B91BFD672837`
- 官方元数据核对：GitHub release API 返回的资产大小为 882,648,883 bytes，digest 为同一 SHA256；本地下载完整。
- 目标路径：`D:\Desktop\动画项目\motion_rolling_prediction\checkpoints`
- 解压结果：24 个文件，共 965,276,231 bytes；8 个配置目录均存在，每个目录均包含相邻 `args.json` 和 optimizer state。
- 完整文件清单：`documents/manifests/rpm-checkpoints-v0.sha256`
- A-P1 Reactive：`model_latest.pt` 为 49,822,233 bytes，SHA256 `CFBF2A23C0BDE3D1B30AAD587144DA1B6BA3FE26D90767950AA54248BAFA88D7`；相邻 `args.json` 存在，协议参数为 input motion 10、free-running 60、motion/sparse context 10/10。
- A-P1 Smooth 官方缺陷：ZIP 内及解压后的 `checkpoints/amass_p1/smooth/model_latest.pt` 均为 0 bytes，SHA256 为标准空文件哈希 `E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855`。ZIP 本身与 GitHub 官方 digest 一致，因此不是本地下载或解压损坏。该目录虽有 `args.json` 和 `opt_latest.pt`，但不能作为可加载模型 checkpoint。
- 协议观察：官方 A-P2 Reactive checkpoint 的 `args.json` 使用 `input_motion_length=5`，与教程训练命令文字存在差异；若后续复评 A-P2，以 checkpoint 相邻参数为证据并记录差异。
- 安全验证：ZIP 共 37 个条目，单一 `checkpoints` 根目录，未发现绝对路径或 `..` 路径穿越条目；原始 ZIP 未移动或修改。
- Git 状态：目标目录命中仓库现有 `.gitignore` 的 `checkpoints` 规则。
- 复评顺序：环境与 A-P1 pilot 数据就绪后，先加载并复评 A-P1 Reactive；A-P1 Smooth 暂记为“官方发布资产不可用”，不得伪称已复评。

## RPM 独立环境部署

### 环境与安装来源

- 环境名：`rpm`
- 环境路径：`D:\Programme\Python\Anaconda3lenvs\rpm`
- Python：`3.10.16`；pip：`24.2`；setuptools：`75.1.0`；wheel：`0.44.0`。
- 创建命令：`conda create -n rpm python=3.10.16 pip=24.2 setuptools=75.1.0 wheel=0.44.0 -y`。
- 首次创建失败：用户 `.condarc` 的清华 Conda channels 在当时网络下触发 `SSL: WRONG_VERSION_NUMBER`，且唯一配置的 package cache `D:\Programme\Python\Anaconda\pkgs` 不可写；失败日志保留为 `documents/logs/01-conda-create-rpm.txt`。
- 成功重试：仅在当前进程设置 `CONDA_PKGS_DIRS=C:\Users\WINDOWS\.conda\pkgs`，并使用 `--override-channels -c defaults`；未修改用户 `.condarc`，未写入 base 环境。
- Conda 创建成功耗时：257.98 秒；日志：`documents/logs/01b-conda-create-rpm-official-defaults.txt`。
- PyTorch 安装命令：`python -m pip install torch==2.7.0 torchvision==0.22.0 torchaudio==2.7.0 --index-url https://download.pytorch.org/whl/cu128`。
- PyTorch CUDA wheels 只来自官方 `download.pytorch.org/whl/cu128`；3.338 GB 主 wheel 实测约 19.1 MB/s，总耗时 579.69 秒；日志：`documents/logs/02-pytorch-cu128-install.txt`。
- RPM 通用依赖最终使用 `https://pypi.tuna.tsinghua.edu.cn/simple`，并加 `--no-cache-dir`；安装耗时 112.93 秒。
- 镜像实测：官方 PyPI/Files 路线约 45–65 KB/s；阿里源约 70 KB/s；腾讯与清华初测曾出现 TLS 错误。用户网络状态变化后，清华源 `grpcio` 达 88.3 MB/s，`numpy` 达 90.8 MB/s。
- 镜像完整性：清华源 `numpy==1.26.0` wheel SHA256 为 `09AAEE96C2CBDEA95DE76ECB8A586CB687D281C881F5F17BFC0FB7F5890F6B91`，与 PyPI 声明一致。华为源返回的同一 wheel 哈希为 `C8090E5413ECD245C31D19FC96127F7BAB7AFD9107F862E38FCAD37598399BFC`，pip 因与期望哈希不一致而拒绝安装；没有绕过校验，后续安装不再使用华为源。
- 依赖偏差：保持官方 pip 列表的版本，但用 `sympy==1.13.3` 代替官方 `1.13.1`，因为 PyTorch 2.7.0 声明要求 `sympy>=1.13.3`。`pyglet==2.1.1` 按仓库基线保留，安装时记录了 PyPI yanked 警告。
- 环境大小：6,692,973,284 bytes（6.233 GiB），43,077 个文件。测量时 D 盘可用约 1,086.24 GiB。

### 验证结果

- CUDA 基础门槛：`torch==2.7.0+cu128`、CUDA runtime `12.8`、cuDNN `90701`、RTX 5070 Ti、compute capability `(12, 0)`，arch list 包含 `sm_120`。1024x1024 GPU 矩阵前向、loss 和 backward 成功，输出与梯度全部有限。
- 调用记录：第一次 CUDA 门槛使用 PowerShell `python -c` 时引号被 native argument parsing 剥离，在执行 CUDA 之前即以 `SyntaxError` 退出；改为通过 stdin 传入代码后通过，成功记录为 `documents/logs/03-cuda-gate.txt`。
- 依赖：`numpy==1.26.0`；Human Body Prior、Body Visualizer、SciPy、Pandas、OpenCV、Trimesh、Pyrender 和 TensorBoard 导入成功。`pip check` 输出 `No broken requirements found.`。
- 入口：`train.py --help` 与 `test.py --help` 均以 exit code 0 退出。
- SMPL-X CPU/GPU：零姿态 vertices `(1, 10475, 3)`、joints `(1, 55, 3)`，值全部有限；GPU pose backward 梯度有限。
- RPM checkpoint：A-P1 Reactive 官方 checkpoint 为 49,822,233 bytes，加载 63-key state dict。合成输入 motion `(1, 10, 132)`、motion context `(1, 10, 132)`、sparse `(1, 20, 54)`；输出 `(1, 10, 132)`，loss/backward 成功，60 个参数梯度张量全部有限。
- A-P1 Smooth：不测试；官方 `model_latest.pt` 为 0 bytes，继续记为官方资产缺陷。
- 可重复 smoke：`D:\Programme\Python\Anaconda3lenvs\rpm\python.exe scripts\smoke_rpm_environment.py`；总耗时 16.54 秒，输出 `RPM_ENVIRONMENT_SMOKE=PASS`。

### 固化与边界

- 推荐重建文件：`environment.rtx5070ti.yaml`。它保留 Python 3.10.16 与 RPM 固定依赖，并明确指向 PyTorch 官方 cu128 index；原 `environment.yaml` 未修改。
- 精确状态：`documents/environments/rpm-conda-export.yaml`、`rpm-conda-explicit.txt` 和 `rpm-pip-freeze.txt`。其中 Conda export 会展示用户 `.condarc` 的 channel 列表，不代表本次成功创建使用这些 channel；重建应以经审计的 `environment.rtx5070ti.yaml` 为主。
- SHA256：环境配置、smoke 脚本、导出文件、日志与本阶段关键模型记录在 `documents/manifests/rpm-environment.sha256`。
- 论文环境偏差：为支持 RTX 5070 Ti，使用 PyTorch 2.7.0/CUDA 12.8，而不是官方环境的 PyTorch 2.5.1/CUDA 11.8；该部署不可描述为与论文软件环境完全一致。
- 本阶段边界：未读写或移动原始 AMASS；未做 AMASS P1 转换；未做真实数据官方 checkpoint 复评、训练或测试。下一阶段仍需先定位已授权的 AMASS/SMPL-H 文件，仅生成少量 A-P1 pilot，并确认 Dataset 实际读取路径后才复评官方 checkpoint。
