# 锆石 Ti 含量温度计（Zircon Ti-in-zircon thermometry）

一个可以直接使用的计算程序：读入锆石 Ti 含量（ppm）与 TiO₂、SiO₂ 活度（及压力），
一次算出 5 种主流标定下的结晶温度、1σ 不确定度、温度分布直方图，并导出 Excel。

本程序内置的公式，均已与本文件夹里各篇文献附带的表格逐点核对：
**15 组校验、共约 900 个计算点，最大偏差 0.0000 K**（见 `validation_report.md`）。

---

## 1. 文件清单

| 文件 | 说明 |
|---|---|
| `锆石Ti温度计.html` | **网页版：双击即用，无需安装任何东西**（离线单文件） |
| `启动GUI.bat` | 双击打开 Python 图形界面（功能相同，需 Python 环境） |
| `ti_gui.py` | 图形界面：表格输入、粘贴 Excel、单点速算、直方图、导出 |
| `zircon_ti.py` | 计算内核 + 命令行，可被其它脚本 `import` |
| `validate_models.py` | 自动校验脚本：把本程序结果与原始表格逐点比对 |
| `validation_report.md` | 校验报告（15 组，全部通过） |
| `example_input.csv` | 输入样例（10 个测点） |
| `example_output.xlsx` | 输出样例（结果 / 按模型统计 / 说明 三个工作表） |
| `screenshot.png` / `screenshot-html.png` | 两个版本的界面截图 |

网页版与 Python 版共用同一套公式与算法，两者在 25 组随机用例上逐项比对：
温差 0.000 K、1σ 差 <1e-9 K、系数边界差 <1e-12 K。

运行环境：网页版只需浏览器（Chrome/Edge/Firefox，离线可用）；
Python 版需 Python 3.9+（numpy、pandas、openpyxl、xlsxwriter、xlrd；界面用标准库 tkinter）。
不需要安装 Excel，也不需要联网。

---

## 2. 快速开始

### 2.1 网页版（推荐）

双击 `锆石Ti温度计.html`，用浏览器打开即可，全部计算在本地完成，不联网、不上传数据。

1. 左侧「参数」填 aSiO₂、aTiO₂、P 的缺省值——表格里留空的单元格会用它（灰色提示即当前生效值）；
2. 「读入 CSV/TXT」选文件，或直接从 Excel 复制整块数据、点表格任一单元格后 Ctrl+V 粘贴（自动识别表头）；
3. 勾选模型、选不确定度方式，点「计算」（或按 Ctrl+Enter）；
4. 左下直方图可切换模型；右下为各模型统计；
5. 「导出 CSV」或「导出 Excel」（结果 + 按模型统计 + 公式说明 三张表）。

注意：网页版直接读 CSV/TXT；`.xlsx` 请在 Excel 里选中区域复制，再粘贴进表格。

### 2.2 Python 图形界面

双击 `启动GUI.bat`（或 `python ti_gui.py`）。

1. 左上「参数」填 aSiO₂、aTiO₂、P 的缺省值——表格里留空的单元格会用它；
2. 「读入文件」选 CSV/Excel，或直接「粘贴 Ctrl+V」从 Excel 复制整块数据，也可「添加行」手工输入；
3. 勾选要用的模型，选不确定度方式，点「计算」；
4. 右下角直方图可切换模型，看温度分布与 1σ；
5. 「保存结果」导出 xlsx（结果表 + 各模型统计 + 公式说明）。

单元格可双击就地编辑；「单点速算」用于试算单个 Ti 值。

### 2.3 命令行

```bat
python zircon_ti.py example_input.csv -o 结果.xlsx

:: 单点试算
python zircon_ti.py --ti 25 --atio2 0.45 --p 0.5

:: 指定模型与不确定度方法
python zircon_ti.py example_input.csv --models FW2007,CRISP2023 --uncertainty mc -o out.xlsx

:: 查看模型列表
python zircon_ti.py --list-models
```

### 2.3 在别的脚本里调用

```python
import zircon_ti as zt

inp = zt.Inputs(Ti=25.0, aTiO2=0.45, aSiO2=1.0, P=0.5,
                Ti_sigma=2.5, aTiO2_sigma=0.15, P_sigma=0.1)
res = zt.compute(inp, ["WH2005", "FW2007", "CRISP2023"], uncertainty="analytic")
for k, r in res.items():
    print(k, f"{r.T_C:.1f} ± {r.sigma_K:.1f} °C")
```

---

## 3. 内置模型

| 键名 | 短名 | 公式（T 单位 K） |
|---|---|---|
| `WH2005` | WH05 | T = 5080 / (6.01 − lg(Ti/aTiO₂)) |
| `FW2007` | FW07 | T = 4800 / (5.711 − lg Ti + lg aTiO₂ − lg aSiO₂) |
| `FW2007_P50` | FW07+50 | 上式 + 50·(P − 1) |
| `FW2007_P100` | FW07+100 | 上式 + 100·(P − 1) |
| `CRISP2023` | Crisp23 | T = 4800 / (5.84 − 0.12P − 0.0056P³ − f·lg aSiO₂ + lg aTiO₂ − lg(Ti·f))，f = 1/(1 + 10^(0.77P − 3.37)) |

- Ti 为 ppm，P 为 GPa，a 为活度（0–1，缺省 1）。
- `WH2005` 只用 aTiO₂，不含 aSiO₂ 与压力项；`FW2007` 是 1 atm 标定，压力高时应用带 P 校正的版本或 `CRISP2023`。
- Crisp (2023) Eq.13 是实验标定的压力相关式，内含 f[4]Ti（四配位 Ti 比例）项，适用压力范围最宽。

各模型系数及其 1σ：

| 模型 | 系数 | 取值 ± 1σ |
|---|---|---|
| WH2005 | A, B | 5080 ± 30, 6.01 ± 0.03 |
| FW2007 | A, B | 4800 ± 86, 5.711 ± 0.072 |
| FW07+50 / +100 | A, B, K | 4800 ± 86, 5.711 ± 0.072, K = 50 / 100 |
| CRISP2023 | A, B, c1, c3, t0, t1 | 4800 ± 86, 5.84 ± 0.07, 0.12 ± 0.01, 0.0056 ± 0.0015, 3.37 ± 0.13, 0.77 ± 0.05 |

---

## 4. 输入数据格式

支持 `.csv` / `.txt`（逗号或制表符分隔，UTF-8 或 GBK）/ `.xlsx` / `.xls`。
程序会在前 40 行内自动找表头，并按列名识别（中英文均可，大小写、括号、空格不敏感）：

| 列 | 可用的列名（部分） | 必需 |
|---|---|---|
| 样品 | 样品、样品号、编号、sample、spot、grain | 否 |
| Ti | Ti、Ti (ppm)、Ti含量、ti ppm | **是** |
| Ti 1σ | Ti 1σ、Ti误差、Ti_sigma | 否 |
| aTiO₂ | aTiO2、TiO2活度、a(TiO2) | 否（缺省 1） |
| aSiO₂ | aSiO2、SiO2活度、a(SiO2) | 否（缺省 1） |
| P | P、P (GPa)、压力 | 否（缺省 0） |
| 各 1σ | aTiO2 1σ、aSiO2 1σ、P 1σ | 否 |

`example_input.csv` 就是最小可用样例：

```
样品,Ti (ppm),Ti 1σ,aTiO2,aSiO2,P (GPa)
Zrn-01,12.6,1.3,0.6,1,0.2
...
```

> 如果直接读原始文献表格（表头多层、前面有标题行）识别不了，最简单是只留
> 「样品 / Ti / aTiO2 / aSiO2 / P」五列，或在 GUI 里从 Excel 复制粘贴。

---

## 5. 不确定度

`--uncertainty` 有四种：

- `analytic`（缺省，推荐）一阶误差传播，同时计入 Ti、aTiO₂、aSiO₂、P 的 1σ 与标定系数 1σ；
- `bounds` 系数取 ±1σ 的极端组合，给出温度上下限（最保守）；
- `mc` 蒙特卡洛（2 万次抽样，活度做截断正态，截断在 (0,1]），误差大时分布明显不对称，更可靠；
- `none` 只算温度。

温度对 Ti 是对数关系，所以 1σ 在低温端往往比高温端大；报告温度时建议写「均值 ± 1σ」并给出点数 n。

---

## 6. 活度怎么取

- **aSiO₂**：岩石含石英（或接近石英饱和）时取 1；不含石英必须实测或估算，否则温度会被系统性带偏。
- **aTiO₂**：有金红石时取 1；无金红石时用 Hayden & Watson (2007) 的全岩成分法估算，花岗质熔体常落在 0.5–0.8。一般酸性岩习惯取 1，但更稳妥的是用 0.6–0.8 做敏感性检验。
- **P**：由矿物温压计或相平衡给出；`WH2005`、`FW2007` 本身不含 P 项，只能靠 `+50(P−1)`、`+100(P−1)` 或 `CRISP2023` 引入。

同一个样品建议同时报几个模型的结果——模型间差异（常 20–60 °C）本身就是方法不确定度的一部分。

---

## 7. 校验情况

`python validate_models.py` 会重跑全部校验并重写 `validation_report.md`。结果概览：

| 来源 | 表/列 | 对应模型 | 点数 | 结果 |
|---|---|---|---|---|
| Zou et al. 2021 附录 C | E、H 列 | WH2005 | 逐点 | 通过 |
| Zou et al. 2021 附录 C | I 列 | FW2007 | 逐点 | 通过 |
| Barboni et al. 2016 数据集 S2 | CP 列 | FW2007（aTiO₂=0.323） | 276 | 通过 |
| Volante et al. 2020 | K、N 列 | FW2007（aTiO₂=0.6/0.9） | 115 | 通过 |
| Volante et al. 2020 | L、M 列（上下限） | FW2007 ±86/±0.072 | 115 | 通过 |
| Crisp et al. 2023 表 S15 | R、T、V、X 列 | WH2005 / FW+50 / FW+100 / Eq.13 | 各 114 | 通过 |
| Crisp et al. 2023 表 S14 | H19 | Eq.13（含 P、f[4]Ti） | 单点 | 通过 |
| 刘志超（锆石中的 Ti） | 计算1、计算2 | FW2007、WH2005 | 37 | 通过 |
| Ti温度计 含活度.xls | F 列 | WH2005（aTiO₂=1） | 逐点 | 通过 |

### 已知的两处原始表格问题

1. `Ti温度计 含活度.xls` 里的 Ferry & Watson 列（C/D/E）与 aTiO₂=0.8/0.6 的 Watson 列，
   用任何已发表公式都复现不出来（差 15–30 °C，相当于 lg aTiO₂ 偏了约 0.066）。
   推测是当年的内部标定或手工调整，本程序不采用，只作为对照数据。
2. `Barboni 2016` 表里的 1σ 列，温度用 FW2007 算、误差项却用了 WH2005 的系数（5080/6.01），
   内部不自洽；本程序统一用各模型自己的系数传播误差。
   Crisp 表 S14 的 I19 单元格在活度项里漏了 ln10，会把 1σ 放大近 2 倍（107.9 K vs 正确的约 60 K）；
   表 S15 的 S11 则是正确的，本程序按正确写法实现。

---

## 8. 参考文献

- Watson, E.B. & Harrison, T.M. (2005) Zircon thermometer reveals minimum melting conditions on earliest Earth. *Geology* 33, 357–360.
- Ferry, J.M. & Watson, E.B. (2007) New thermodynamic models and revised calibrations for the Ti-in-zircon and Zr-in-rutile thermometers. *Contributions to Mineralogy and Petrology* 154, 529–540.
- Hayden, L.A. & Watson, E.B. (2007) Rutile saturation in hydrous siliceous melts and its bearing on Ti-thermometry of quartz and zircon. *Earth and Planetary Science Letters* 258, 561–568.
- Barboni, M., Boehnke, P., Schmitt, A.K. et al. (2016) Warm storage for arc magmas. *PNAS* 113, 13279–13284.
- Volante, S. et al. (2020) *Contributions to Mineralogy and Petrology* 175, 52.
- Zou, H. et al. (2021) *Geoscience Frontiers*（附录 C 锆石 Ti 温度计汇总）.
- Crisp, L.J., Berry, A.J., Huang, et al. (2023) *Geochimica et Cosmochimica Acta*（Ti-in-zircon 新标定，表 S14–S16）.
