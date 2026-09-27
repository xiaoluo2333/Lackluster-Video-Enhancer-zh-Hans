# REAL 视频增强器

> 🌐 **语言 / Language:** [English](README.md) | **简体中文 (zh-Hans)**

<h1>公告</h1>
<h3>这是 REAL Video Enhancer 的一个积极维护分支：在已归档的上游版本之上，应用了缺陷修复、稳定性改进与性能优化。</h3>

## 本分支的改动

**最新新增**
- 新增 **bfloat16** 作为 PyTorch 后端的精度选项。`Auto` 现在会在 RTX 30 系列 / Ampere 及以上 GPU 上使用 bfloat16（速度与 fp16 相同，动态范围更好 —— 明亮 / HDR 内容不会溢出），其他情况使用 fp16。TensorRT 后端始终使用 fp16/fp32 引擎（TensorRT 引擎不支持 bf16）。
- 新增 **PyTorch 2.13**（CUDA 13 / Blackwell / RTX 50 系列）与 **TensorRT 11** 作为可选安装项。旧版本（2.9 / 2.8 / 2.6）保留在下拉菜单中，以兼容较旧的 GPU —— 详见应用内的提示信息。
- **2.13 / TRT 11 组合下的 TensorRT 引擎修复**：不再向 TRT 引擎构建传入 bf16；此前在 torch-tensorrt 2.13 上导致重影/闪烁伪影的 RIFE `grid_sampler` 拆解实现，已替换为原生 grid_sampler 路径（引擎会自动以 `_gsnative` 后缀重新构建）。
- 修复了 PyInstaller 构建（Python 3.11 上的 `backports` 运行时错误），并让 `build.py` 不再强制重建已存在的 venv。
- 修复了本分支的 GitHub Actions CI（使用本仓库的分支与发行说明、Node 24 运行时、macOS Homebrew tap 信任 / llvm 修复）；`build-prerelease` 会构建 Windows/Linux/macOS 并发布 release。
- 应用更新检查、主页选项卡的更新日志以及后端下载现在都指向**本分支**，而不是上游。
- Windows 安装程序（NSIS）版本同步至 2.4.2，品牌标记为 **LacklusterOpsec**，开始菜单快捷方式直接安装（无公司子文件夹）。

**缺陷修复**
- 修复了 FFmpeg 读取在渲染中途随机停止的问题（读取进程使用 `-nostdin`）
- 修复了旋转 / iPhone 竖屏视频无法渲染的问题（感知旋转的尺寸、安全的裁剪钳制、边界检测中的 `-noautorotate`）
- 修复了启用修复模型时 PySceneDetect 行为异常的问题
- 修复了渲染线程崩溃导致整个应用卡死的问题（带 traceback 的全局异常处理器）
- 修复了启动时与共享内存暂停块相关的暂停/恢复竞争问题
- 修复了模型名称不含缩放倍数时 NCNN 缩放倍数检测崩溃的问题
- 修复了分段场景检测只评估部分图像的问题
- 修复了 NCNN 放大帧报告缩放前尺寸的问题
- 修复了小数（非整数）插帧的帧时序与缓存步长
- 修复了暂停后进度 / 预计剩余时间少算以及 FPS 错误的问题
- 简单安装失败时现在会干净退出，而不是启动一个损坏的界面

**性能 / 稳定性**
- 移除了每帧一次的全局 CUDA 同步与冗余张量克隆
- 场景检测现在在 GPU 上降采样（不再每帧把全分辨率帧复制到 CPU）
- 输出覆盖缩放的重置移出了 CPU 路径（可用时使用 GPU 缩放）
- NCNN 回退实现复用其提取器，不再每帧分配
- TensorRT 引擎构建完成后现在会清理 VRAM
- 调整了 AV1 / AV1 NVENC 质量预设
- 版本号提升至 2.4.2

## 兼容性

### PyTorch 版本（可在“下载”选项卡中选择）

| 版本 | CUDA wheel | GPU 支持 | 说明 |
|---|---|---|---|
| **2.13.0**（默认） | `+cu130` | RTX 20/30/40/50（Turing / sm_75 及更新） | 最新版本。在 RTX 50 系列 / Blackwell 上表现最佳。搭配 TensorRT 11 使用。需要支持 CUDA 13 的驱动。 |
| **2.9.0** | `+cu130` | RTX 20/30/40（Turing+） | 稳定的全能版本。 |
| **2.8.0** | `+cu129` | RTX 20/30/40（Turing+） | |
| **2.6.0** | `+cu118` | GTX 10/16、V100 及更旧（Pascal/Volta/Turing+） | 传统回退版本。如果较新版本在你的 GPU 上加载失败，请使用此版本。 |

说明：
- **CUDA 13（`+cu130`）需要 Turing 或更新架构（sm_7.5+）。** Maxwell/Pascal/Volta GPU（GTX 900/10、V100）必须使用 CUDA 11.8 构建（**2.6.0**）。
- Volta（V100）支持已从较新的 CUDA 12.8+ / 13 wheel 集合中移除；这些显卡请使用 2.6.0。
- **ROCm（AMD，仅 Linux）：** 2.13 → ROCm 7.2，2.9/2.8 → ROCm 6.4，2.6 → ROCm 6.2.4。
- **XPU（Intel）：** 上述所有版本均可用。
- **MPS（Apple Silicon）：** 为支持 uint16（HDR）而锁定在 PyTorch 2.9.0。

### 精度选项

| 精度 | GPU 支持 | 说明 |
|---|---|---|
| **auto**（默认） | — | PyTorch 后端：在 RTX 30 系列 / Ampere 及以上（sm_80+）使用 **bfloat16**，其他情况使用 **float16**，不支持半精度时使用 **float32**。TensorRT 后端：始终使用 fp16/fp32 引擎。 |
| **float16** | 所有 CUDA/ROCm/XPU/MPS | 最快，显存占用最低。 |
| **bfloat16** | RTX 30 系列及以后（Ampere，sm_80+） | 仅 PyTorch 后端。速度与 float16 相同，但具有 8 位指数 —— 在明亮 / HDR 内容上不会溢出。如果模型不支持，会自动回退到 float32。TensorRT 则改用 fp16。 |
| **float32** | 全部 | 最慢，兼容性最好。 |

### TensorRT

| PyTorch | TensorRT | torch-tensorrt | 说明 |
|---|---|---|---|
| **2.13.0** | **11.0.0.114** | 2.13.0 | 需要 CUDA 13。需要 Turing 及以上（RTX 20 系列 / sm_75+）。 |
| 2.9.0 / 2.8.0 / 2.6.0 | 10.12.0.36 | 与 torch 版本匹配 | 较旧的 TRT 10 系列。 |

说明：
- TensorRT 需要带专用张量核心的 NVIDIA GPU —— **RTX 20 系列（Turing）或更新**。GTX 10/16 系列及更旧的显卡不可用。
- TensorRT 11 需要支持 CUDA 13 的驱动（所有使用较新驱动的 RTX 20+）。
- TensorRT 引擎仅以 fp16/fp32 构建（不支持 bf16 引擎）。
- 在 torch-tensorrt 2.13 / TRT 11 上，RIFE 引擎使用原生 `grid_sampler` 路径（旧的手动拆解实现会在该组合上导致重影/闪烁伪影）。

### NCNN（Vulkan）

任何具备 Vulkan 驱动的 GPU 均可运行（NVIDIA、AMD、Intel、Apple Silicon）。无需 CUDA。是较旧硬件或 CUDA 不可用时的绝佳回退方案。

![license](https://img.shields.io/github/license/LacklusterOpsec/Lackluster-Video-Enhancer)
![Version](https://img.shields.io/badge/Version-2.4.2-blue)

<p align=center>
  <img src="https://raw.githubusercontent.com/LacklusterOpsec/Lackluster-Video-Enhancer/v2-main/icons/logo-v2.svg" width = "25%">
</p>

# 目录
  
* **[简介](#简介)**
* **[功能](#功能)**
* **[兼容性](#兼容性)**
* **[硬件与软件要求](#硬件与软件要求)**
* **[模型](#模型)**
  * [插帧模型](#插帧模型)
  * [放大模型](#放大模型)
* **[后端](#后端)**
* **[常见问题](#常见问题)**
  * [常规应用使用](#常规应用使用)
  * [TensorRT](#tensorrt-相关问题)
  * [ROCm](#rocm-相关问题)
  * [NCNN](#ncnn-相关问题)
* **[克隆](#克隆)**
* **[构建](#构建)**
* **[Colab 笔记本](#colab-笔记本)**
* **[致谢](#致谢)**

# 简介

<strong>REAL 视频增强器</strong> 是原 Rife ESRGAN App for Linux 的重新设计与增强版本。本程序在 Windows、Linux 和 macOS 上提供便捷的插帧与放大功能，是 <a rel="noopener noreferrer" href="https://nmkd.itch.io/flowframes" target="_blank" >Flowframes</a> 或 <a rel="noopener noreferrer" href="https://github.com/mafiosnik777/enhancr" target="_blank">enhancr</a> 等过时软件的替代品。

<p align=center>
  <img src="https://raw.githubusercontent.com/LacklusterOpsec/Lackluster-Video-Enhancer/v2-main/screenshots/demo.png" width = "100%">
</p>
# 功能
<ul>
  <li> 支持 Windows。<strong>!!! 注意 !!!</strong> 该可执行文件可能被检测为木马。这是 pyinstaller 导致的误报。</li>
  <li> 支持 Ubuntu 22.04+ 的可执行文件与 Flatpak。（20.04 可用，但已属传统支持） </li>
  <li> 支持 macOS 15+ arm/x86 </li>
  <li> 支持 Discord 系统包与 Discord flatpak 的 Discord RPC。 </li>
  <li> 场景变化检测，以保留锐利的转场。 </li>
  <li> 预览窗口显示最新已渲染的帧。 </li>
  <li> 使用 TensorRT 与 NCNN 在多种 GPU 上高效推理。 </li>
</ul>

# 硬件与软件要求
|  | 最低配置 | 推荐配置 | 
 |--|--|--|
| CPU | 双核 x64 位 | 四核 x64 位
| GPU | 支持 Vulkan 1.3 的设备 | Nvidia RTX GPU（20 系列及以上）
| 显存 | 4 GB - NCNN | 8 GB - TensorRT（Nvidia，为什么还在做 8GB 显存的卡？）
| 内存 | 16 GB | 32 GB
| 存储 | 1 GB 可用空间 - NCNN | 16 GB 可用空间 - TensorRT
| 操作系统 | Windows 10/11 64 位 / macOS 14+ | 任何现代 Linux 发行版（Ubuntu 22.04+）

# 模型
### 插帧模型
| 模型 | 作者 | 链接 |
|--|--|--|
| RIFE 4.6,4.7,4.15,4.18,4.22,4.22-lite,4.25 | Hzwer | [Practical-RIFE](https://github.com/hzwer/Practical-RIFE) 
| GMFSS | 98mxr | [GMFSS_Fortuna](https://github.com/98mxr/GMFSS_Fortuna) 
| IFRNet | ltkong218 | [IFRnet](https://github.com/ltkong218/IFRNet)

### 放大模型
| 模型 | 作者 | 链接 |
|--|--|--|
| 4x-SPANkendata | Crustaceous D | [4x-SPANkendata](https://openmodeldb.info/models/4x-SPANkendata) 
| 4x-Nomos8k-SPAN series | Helaman | [4x-Nomos8k-SPAN series](https://openmodeldb.info/models/4x-Nomos8k-span-otf-strong) 
| 2x-OpenProteus | SiroSky | [OpenProteus](https://github.com/Sirosky/Upscale-Hub/releases/tag/OpenProteus) 
| 2x-AnimeJaNai V2 and V3 Sharp | The Database | [AnimeJanai](https://github.com/the-database/mpv-upscale-2x_animejanai)
| 2x-AniSD | SiroSky | [AniSD](https://github.com/Sirosky/Upscale-Hub/releases/tag/AniSD)
| AnimeSR | Tencent ARC | [AnimeSR](https://github.com/TencentARC/AnimeSR)

### 去除压缩模型
| 模型 | 作者 | 链接 |
|--|--|--|
| DeH264 | Helaman | [1xDeH264_realplksr](https://github.com/Phhofm/models/releases/tag/1xDeH264_realplksr) 

### 降噪模型
| 模型 | 作者 | 链接 |
|--|--|--|
| DRUnet | cszn | [DRUnet](https://github.com/cszn/DPIR)
| DnCNN | czsn | [DnCNN](https://github.com/cszn/DPIR)

# 后端
  | 后端 | 硬件 | 
  |--|--|
  | TensorRT | NVIDIA RTX GPUs
  | PyTorch  | CUDA 12.6 and ROCm 6.2 capable GPUs
  | NCNN | Vulkan 1.3 capable GPUs
 
# 常见问题
### 常规应用使用
  | 问题 | 回答 | 
  |--|--|
  | 本程序试图实现什么？ | 快速、高效且易于使用的视频插帧（例如 24->48FPS）与视频放大（例如 1920->3840）
  | 为什么无法识别已安装的后端？ | REAL Video Enhancer 使用 PIP 和便携式 Python 进行推理，安装过程有时会出问题。在提交 issue 之前，请先尝试重新安装本应用。

### TensorRT 相关问题
  |||
  |--|--|
  | 为什么开始推理要花很长时间？ | TensorRT 会在推理开始时根据你的设备进行高级优化，每种输入视频分辨率只需执行一次。
  | 为什么优化和推理会失败？ | 优化失败最常见的原因是**显存不足**。除了改用 CUDA 或 NCNN，没有别的解决办法。
 
### ROCm 相关问题
  |||
  |--|--|
  | 为什么我会遇到（在此填入错误信息）？ | ROCm 存在不少缺陷，请查看 <a href="https://github.com/TNTwise/REAL-Video-Enhancer/wiki/ROCm-Help">ROCm Help</a>。

### NCNN 相关问题
  |||
  |--|--|
  | 为什么我会遇到（在此填入 Vulkan 错误信息）？ | 这通常是 OOM（内存不足）错误，可能说明 iGPU 性能较弱或 GPU 过于老旧，我建议改用 <a href="https://github.com/TNTwise/REAL-Video-Enhancer-Colab">Colab 笔记本</a>。


# 克隆
```
# 主分支
git clone --recurse-submodules https://github.com/LacklusterOpsec/Lackluster-Video-Enhancer --branch v2-main
```
# 构建

<p>支持 3 种构建方式：</p>
<p> - pyinstaller（推荐用于 Windows/Mac）<br/>
    - cx_freeze（推荐用于 Linux）<br/>
    - nuitka（实验性）
</p>
<p>支持的 Python 版本：</p>
<p> - 3.10 3.11, 3.12 <br/>
</p>

```
python3 build.py --build BUILD_OPTION --copy_backend
```

**构建产物（便携版）：** `dist\REAL-Video-Enhancer\REAL-Video-Enhancer.exe`

整个 `dist\REAL-Video-Enhancer\` 目录即为完整可运行的便携版本，可直接拷贝分发；双击 exe 即可启动，无需安装。

# Colab 笔记本
 <a href="https://github.com/tntwise/REAL-Video-Enhancer-Colab">Colab 笔记本</a>

# 致谢
### 个人
| 人员 | 贡献 | 链接 |
|--|--|--|
| NevermindNilas | 部分后端与参考代码，并与我在多个项目上合作 | https://github.com/NevermindNilas/ 
| Styler00dollar | RIFE ncnn 模型（4.1-4.5、4.7-4.12-lite）、Sudo Shuffle Span 以及场景检测模型 | https://github.com/styler00dollar 
| HolyWu | TensorRT 引擎生成代码、推理优化，以及 RIFE 锯齿线修复 | https://github.com/HolyWu/ 
| Rick Astley | 出色的音乐 | https://www.youtube.com/watch?v=dQw4w9WgXcQ 

### 软件
| 使用的软件 | 用途 | 链接|
|--|--|--|
| FFmpeg | 用于处理视频、音频及其他媒体文件的多媒体框架 | https://ffmpeg.org/ 
| QT | GUI 框架 | https://qt.io/
| FFMpeg Builds | FFMpeg 的预编译构建。 | Windows/Linux:  https://github.com/BtbN/FFmpeg-Builds, MacOS: https://github.com/eko5624/mpv-mac
| PyTorch | 神经网络推理（CUDA/ROCm/TensorRT） | https://pytorch.org/ 
| NCNN | 神经网络推理（Vulkan） | https://github.com/tencent/ncnn 
| RIFE | 用于视频插帧的实时中间流估计 | https://github.com/hzwer/Practical-RIFE 
| rife-ncnn-vulkan | 使用 NCNN 和 Vulkan 的视频插帧实现 | https://github.com/nihui/rife-ncnn-vulkan 
| rife ncnn vulkan python | RIFE NCNN Vulkan 实现的 Python 绑定 | https://github.com/media2x/rife-ncnn-vulkan-python 
| GMFSS | 基于 GMFlow 的动漫 VFI | https://github.com/98mxr/GMFSS_Fortuna
| GIMM | 运动建模的真实感 VFI | https://github.com/GSeanCDAT/GIMM-VFI 
| ncnn python | NCNN Vulkan 框架的 Python 绑定 | https://pypi.org/project/ncnn 
| Real-ESRGAN | 放大 | https://github.com/xinntao/Real-ESRGAN 
| SPAN | 放大 | https://github.com/hongyuanyu/SPAN 
| Spandrel | CUDA 放大模型架构支持 | https://github.com/chaiNNer-org/spandrel 
| ChaiNNer | 模型缩放倍数检测 | https://github.com/chaiNNer-org/chainner
| cx_Freeze | 从 Python 脚本创建独立可执行文件的工具（Linux 构建） | https://github.com/marcelotduarte/cx_Freeze 
| PyInstaller | 从 Python 脚本创建独立可执行文件的工具（Windows/Mac 构建） | https://github.com/pyinstaller/pyinstaller
| Feather Icons | 开源图标库 | https://github.com/feathericons/feather 
| PySceneDetect | Python 转场检测库 | https://github.com/Breakthrough/PySceneDetect/
| Python Standalone Builds | 使用便携式 Python 进行后端推理，便于移植到不同平台。 | https://github.com/indygreg/python-build-standalone |


# Star 历史
[![Star History Chart](https://api.star-history.com/svg?repos=LacklusterOpsec/Lackluster-Video-Enhancer&type=Date)](https://star-history.com/#LacklusterOpsec/Lackluster-Video-Enhancer&Date)
