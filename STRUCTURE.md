# Infinite-ISP 项目结构说明

> 一句话：Python 实现的完整 ISP 流水线参考模型。输入 RAW，输出处理后的 RGB/YUV 图像。

---

## 1. 快速运行

```powershell
cd D:\ISP-total\Infinite-ISP
python isp_pipeline.py
```

- 输入：`config/configs.yml` 中 `platform.filename` 指定的 RAW（默认 `in_frames/normal/ColorChecker_2592x1536_12bits_RGGB.raw`）
- 输出：`out_frames/Out_<文件名>_<时间戳>.png` + 同名 `.yaml`（本次运行配置快照）
- 批量：`python isp_pipeline_mulitple_images.py`

实测：Python 3.12.10，2592×1536 单帧约 **21.7 s**。

---

## 2. 目录结构

```
Infinite-ISP/
├── isp_pipeline.py                  ← 单张入口
├── isp_pipeline_mulitple_images.py  ← 批量入口
├── infinite_isp.py                  ← 编排器（核心）
├── requirements.txt                 ← 6 个依赖
├── config/
│   └── configs.yml                  ← 唯一配置文件
├── modules/                         ← 算法库，20 个模块
│   ├── crop/  dead_pixel_correction/  black_level_correction/  oecf/
│   ├── digital_gain/  lens_shading_correction/  bayer_noise_reduction/
│   ├── auto_white_balance/  white_balance/  demosaic/
│   ├── color_correction_matrix/  gamma_correction/  auto_exposure/
│   ├── color_space_conversion/  ldci/  sharpen/
│   └── noise_reduction_2d/  rgb_conversion/  scale/  yuv_conv_format/
├── util/                            ← 工具层（读写图、保存、统计）
├── in_frames/                       ← 输入 RAW（normal/ 三张，hdr_mode/ 空）
├── out_frames/                      ← 输出图像 + 配置快照
├── docs/                            ← 算法原理 PDF
├── assets/                          ← README 配图
├── .github/                         ← CI（lint）+ PR 模板
└── blc_test/                        ← 本地单模块测试（未被 git 跟踪）
```

---

## 3. 四层架构

| 层 | 位置 | 职责 |
|---|---|---|
| 入口层 | 根目录 3 个 `.py` | 指定输入图与配置文件 |
| 编排层 | `infinite_isp.py` | 唯一知道处理顺序的地方 |
| 算法层 | `modules/` | 20 个互不依赖的类，只做“数组进、数组出” |
| 工具层 | `util/` | 读图、存图、CFA 辅助、3A 统计 |

模块之间**零耦合**：`modules/` 内没有任何跨模块 import，全部由 `infinite_isp.py` 串联。

---

## 4. 流水线顺序

```
Crop → DPC → BLC → OECF → DigitalGain → LSC → BNR
     → AWB → WB → Demosaic → CCM → Gamma → AE
     → CSC → LDCI → Sharpen → NR2D → RGBConv → Scale → YUV
```

| 阶段 | 模块 | 作用 |
|---|---|---|
| RAW 前端 | Crop, DPC, BLC, OECF, DigitalGain, LSC | 裁剪、坏点、黑电平、非线性校正、增益、暗角 |
| Bayer 域 | BNR, AWB, WB | 拜耳域降噪、白平衡估增益与应用 |
| 重建 | Demosaic | 单通道 CFA → 三通道 RGB |
| 颜色 | CCM, Gamma, AE, CSC | 色彩矩阵、gamma、曝光统计、色彩空间 |
| 后处理 | LDCI, Sharpen, NR2D, RGBConv, Scale, YUV | 对比度、锐化、降噪、格式/尺寸/编码 |

---

## 5. 关键文件

| 文件 | 说明 |
|---|---|
| `infinite_isp.py` | `InfiniteISP` 类：`load_config()` → `load_raw()` → `run_pipeline()` |
| `config/configs.yml` | 23 个段落，含输入文件名、位深、Bayer 排列、各模块参数 |
| `util/utils.py` | `save_pipeline_output()`、`apply_cfa()` 等公共函数 |
| `requirements.txt` | numpy / scipy / matplotlib / rawpy / PyYAML / tqdm |

---

## 6. 配置机制

- 唯一配置源：`config/configs.yml`，`yaml.safe_load()` 解析成嵌套字典后传给各模块。
- 模块开关：段内 `is_enable: true/false`。
- **3 个模块没有开关、永远执行**：`digital_gain`、`demosaic`、`color_space_conversion`。
- `is_save: true` 可把该级中间结果存到 `output_array/`。
- `render_3a: true` 时启用 3A 迭代（AWB/AE 反馈直到曝光收敛）。

---

## 7. 单模块调试

项目**没有** per-module CLI，模块文件均无 `__main__`，直接运行会因 `sys.path` 问题报错。

要在隔离环境跑单个模块，需自行 import 并构造参数，示例见 `blc_test/run_blc.py`：

```python
from modules.black_level_correction.black_level_correction import BlackLevelCorrection

blc = BlackLevelCorrection(raw, platform, sensor_info, parm_blc)
out = blc.execute()
```

---

## 8. 环境与依赖

- 已验证：Python **3.12.10**（64-bit），numpy 2.5.1 下全流程无报错
- 依赖齐备（含 rawpy 0.27.1，仅 DNG/NEF 输入时才被调用）
- `.raw` 文件为裸 12-bit 打包数据，由 `np.fromfile` 读取

---

## 9. 本地附加内容（非上游）

| 路径 | 说明 |
|---|---|
| `blc_test/` | BLC 单模块测试脚本与产物，已加入 `.git/info/exclude` |
| `out_frames/*.png/.yaml` | 本地运行产生的输出，上游仅保留 `.gitkeep` |
