# comsol_3d_hotspot_dense3：dense3 三维 COMSOL 模型 — 只解线上应力，温度用 HotSpot 的值

算例：base3 堆叠、50 µm 轨距（每颗芯片 162 条轨 × 80 段）。

这个包与 `comsol_3d_dense3` 的几何、轨、方程、输出文件和图完全相同，唯一的区别是**不在 COMSOL 里解温度**：线上的温度直接取 `segments.csv` 里的 T_L、T_R（即 StackEM 用的 HotSpot 温度场），温度图把 HotSpot 的场贴在芯片上显示。没有热学研究步，所以 csv 里的两列温度都是 HotSpot 的；成核时间应与参考求解器基本一致（用来确认三维模型里线上的方程是对的）。体网格只为出图而存在，默认很粗。

这个包独立运行，不读写 `C:\stackem_cf`，不影响计时实验。需要 COMSOL 6.2、LiveLink for MATLAB 和 MATLAB；出图另需 Python（numpy、matplotlib；`t3d_emulate.py` 另需 scipy）。

`t3d_run.m` 是在 base3 上跑通（246 条轨、全部图）的脚本的通用化版本：堆叠改成从 `stack.json` 读的层列表，芯片可以是矩形，层内可以有导热系数不同的插入块；脚本本身不含任何算例专有的数字。通用化后的代码没有再上 COMSOL 跑过，**第一次请先做冒烟测试**（见下）。Python 部分用 `t3d_emulate.py` 生成的本算例数据测试过。

## 算例数据（`data\dense3\`，由 `t3d_make_data.py` 从 StackEM 的运行结果和 HotSpot 的输入文件生成）

- 芯片 4 × 4 mm，3 颗；轨 486 条（每颗芯片每个方向 81 条，每条 80 段，轨距 50 µm），共 38880 段、39366 个结点；Blech 筛选后 274 条需要求解（COMSOL 里全部 486 条都解）。
- 封装：R_conv = 0.5 K/W，环境 318.15 K；HotSpot 网格 64 × 64。
- 参考求解器（HotSpot 温度）：最早成核 4.33 年，10 年内成核 10 条。三维热场版的成核时间会因温度差别而不同，这是正常的。

| 芯片 | 轨数 | 每轨段数 | 轨宽 (µm) | 功耗 (W) | HotSpot 温度 (K) | 参考最早成核 / 10 年内条数 |
|---|---|---|---|---|---|---|
| D1 | 162 | 80 | 0.96 | 2.00 | 342.3 – 359.4 | 9.68 年 / 2 |
| D2 | 162 | 80 | 0.65 | 4.00 | 342.2 – 359.6 | 4.33 年 / 8 |
| D3 | 162 | 80 | 8.14 | 20.00 | 342.0 – 359.9 | 22.25 年 / 0 |

堆叠各层（自下而上，`stack.json`）：

| 层 | 类型 | 厚度 (µm) | k (W/m·K) | 面内尺寸 (mm) | 备注 |
|---|---|---|---|---|---|
| D1 | die | 50 | 100 | 4 × 4 | |
| bond1 | bond | 5 | 40 | 4 × 4 | |
| D2 | die | 50 | 100 | 4 × 4 | |
| bond3 | bond | 5 | 40 | 4 × 4 | |
| D3 | die | 50 | 100 | 4 × 4 | |
| TIM | tim | 20 | 4 | 4 × 4 | |
| spreader | spreader | 1000 | 400 | 12 × 12 | |
| sink | sink | 6900 | 400 | 30 × 30 | |

dense3 的特殊之处：与 base3 同一堆叠和同一热场，轨距减半到 50 µm，轨数和段数各翻倍（每颗 162 条、每条 80 段）；线上的网格仍是每段 60 个单元。

## 文件

| 文件 | 作用 |
|---|---|
| `t3d_run.m` | 建模、求解、导出 `.mph`、`.csv`、`.mat` 和 COMSOL 渲染的图 |
| `t3d_figures.py`、`emfig.py` | 全部结果图，包括与论文堆叠图同画法的爆炸图 |
| `t3d_compare.py` | 温度场与 HotSpot 的对照图和统计，由 `t3d_figures.py` 自动调用 |
| `t3d_emulate.py` | 不用 COMSOL 的检查程序：用 HotSpot 温度在 Python 里解同一方程，按 `t3d_run.m` 的格式写到 `out_emul\`，用来事先测试出图链路（不是 COMSOL 结果） |
| `t3d_make_data.py` | 在服务器的 StackEM 环境里重新生成 `data\dense3\`（已生成，一般不用再跑） |
| `data\dense3\rails.csv`、`segments.csv`、`constants.json`、`rails.json` | 全部 486 条轨的几何、电流、焦耳温升、热残余应力和材料常数（`cf_export.py`） |
| `data\dense3\stack.json`、`power_blocks.csv` | 堆叠各层、插入块、封装参数、网格默认值；每颗芯片的功耗块（取自 HotSpot 实际运行的 .lcf/.flp/.ptrace/.config） |
| `data\dense3\hotspot_T.csv` | HotSpot 给出的每颗芯片 64 × 64 温度 |
| `data\dense3\stackem_result.csv`、`stackem_timing.json` | 参考求解器和 StackEM 在 HotSpot 温度下的成核时间（对照用；json 里的秒数是生成数据的机器上的，不是计时结果） |

## 模型

**体网格只用于出图。** 自下而上按 `stack.json` 的层列表建模（上表），均热板和散热器以芯片叠层为中心。默认面内尺寸：芯片叠层 200 µm、均热板 1000 µm、散热器 3000 µm；厚度方向 [芯片 键合层 TIM 均热板 散热器] = [2 1 1 2 2] 层（约 8.2e3 个棱柱）。

**应力，线上的瞬态。** 每条轨是一条折线，位置和长度取实际值，横线在所在芯片上表面上方 2 µm，竖线 4 µm（两层金属，交叉处不相连）。线上解 Korhonen 方程，两端零通量，每段 60 个 Chebyshev 加密的单元（共 2332800 个线单元），BDF 变步长，相对容差 1e-6，默认积分到 10 年，输出每 0.05 个数量级一个时刻（约 93 个）。线上的温度取 `segments.csv` 的 HotSpot 值加解析焦耳温升；电流密度用 StackEM 电源网的结果，初始应力是当地温度下的热残余应力。线是独立的几何对象，解完后有两项自检（初始应力、线上温度）确认线上用的温度正确。

## 运行

启动 COMSOL 服务器，打开 MATLAB：

```matlab
addpath('D:\Program Files\COMSOL\COMSOL62\Multiphysics\mli'); mphstart(2036);   % 路径和端口按自己电脑的安装改
cd <本文件夹>
t3d_run('max_rails', 2, 'tag', 'smoke')     % 冒烟测试：每颗芯片 2 条横线 + 2 条竖线
t3d_run                                     % 全部 486 条轨，到 10 年，结果在 out_run\
t3d_run('tend_years', 100, 'tag', 'run100') % 积分到 100 年
t3d_run('rail_plots', 0)                    % 不导出逐轨的 1-D 图
```

也可以直接用 “COMSOL Multiphysics with MATLAB” 启动，那样不需要前两条命令。

出图（只需要 numpy 和 matplotlib）：

```bash
python t3d_figures.py --out out_run
```

不开 COMSOL 先试出图链路：

```bash
python t3d_emulate.py            # 写 out_emul\，屏幕上给出与参考求解器的成核时间对比
python t3d_figures.py --out out_emul
```

## 输出（`out_<标签>\`）

| 文件 | 内容 |
|---|---|
| `dense3_3d.mph` | 模型、解和全部绘图组 |
| `t3d_die_temperature.csv` | 每颗芯片在 HotSpot 网格点上的 温度（两列都是 HotSpot） |
| `t3d_nodes.csv` | 每个结点：坐标、温度、各时间点的应力 |
| `t3d_nodes_history.csv` | 时刻 × 结点的应力（约 93 × 39366） |
| `t3d_history.mat` | 完整的时间 × 位置应力场：线网格每个顶点（2332800 个点）在每个时刻的应力（single，约 0.9 GB），`'history',0` 可不存 |
| `t3d_field_<H>yr.csv` | 各时间点每段 11 个点的沿线应力 |
| `t3d_rail_result.csv`、`t3d_rail_smax.csv` | 每条轨的成核时间和各时刻的最大结点应力 |
| `t3d_meta.json` | 设置、各步秒数、单元数、走过的退路 |
| `comsol_images\` | COMSOL 渲染的图：`pgT_all`（整个封装的温度）、`pgT_stack`（只有芯片叠层）、`pgS3d_<k>`（第 k 个时间点的三维轨应力）、`pgS2d_<芯片>_<k>`（每颗芯片每个时间点的俯视应力图，3 × 4 张）、`pgTn`（成核时间）、`rails\rail_*.png`（每条轨一张，486 张） |
| `figures\`、`fig_t3d_*.png`、`t3d_compare.json` | `t3d_figures.py` / `t3d_compare.py` 的图和统计（爆炸图、各芯片电源线应力图、成核时间、应力历史、CDF、直方图、沿线应力、对照散点图） |

## 读结果时要注意

- 内存和时间：2332800 个线单元 × 约 93 个输出时刻都存在 `.mph` 里；`t3d_history.mat` 约 0.9 GB。
- 冒烟测试通过的标准：两项自检通过（初始应力偏差 < 5 MPa、线上温度偏差 < 0.2 K），屏幕上最早成核时间与 `stackem_result.csv` 的参考值同量级（HotSpot 温度版应在 1 % 以内），没有 “API fall-backs taken” 或只有无害的那几条。
- 报 “found N rail edges, expected M”“layer ...: found ... domains”：选择框没选对，把屏幕输出发回来。结束时列出的退路如果不认识，也发回来确认。
