# GNSS 震后形变统一处理流程

本流程适用于同一地震事件影响下的任意 `.tenv3` GNSS 测站。它实现以下目标：固定同震时刻、避免使用混合日解污染同震阶跃、优先拟合震后早期快速变化、输出 ENU 图件与五年累计形变。

当前实现脚本为 `j714_decomposition.py`；脚本名称来自最初的 J714 站点，但可以读取任意站点的 `.tenv3` 文件。

## 1. 输入与固定设置

- 输入：一个测站的 `.EU.tenv3` 日解时间序列。
- 分析窗口：`2013.0 <= t < 2024.085`，可按研究需要改为其他窗口。
- 同震时刻：`t0 = 2016.2875`。
- 分量：East、North、Up 分别独立拟合，单位统一为 mm。
- 每个站必须至少有一条包含 `t0` 的同震日解，以及一条 `t > t0` 的完整震后日解；否则不能使用本流程的首日锚定约束。

## 2. 模型

对每个分量拟合：

\[
y(t)=c_0+v(t-t_{ref})+S_1\sin(2\pi t)+C_1\cos(2\pi t)
+S_2\sin(4\pi t)+C_2\cos(4\pi t)
+\Delta_cH(t-t_0)+A[1-\exp(-(t-t_0)/\tau)]H(t-t_0).
\]

其中：

- 前六项是构造趋势及年/半年周期项；
- `Δc` 是同震阶跃；
- `A[1-exp(-dt/tau)]` 是指数震后项；`A` 为渐近震后位移，`tau` 为弛豫时间（年）；
- `H` 为 Heaviside 阶跃函数；指数项从 `t0` 开始。

## 3. 同震日与首个完整震后日：必须这样处理

1. 固定同震时刻为 `t0`，不要让其在拟合中漂移。
2. **排除 `t0` 所在日解**。该日常为同震与首日震后过程的混合坐标，不能直接代表静态同震位移。
3. 找到首个完整震后日：

   ```text
   t1 = min(t | t > t0)
   ```

4. 使用 `t1` 锚定阶跃，使模型严格通过该坐标：

   \[
   \Delta_c=y(t_1)-[c_0+v(t_1-t_{ref})+periodic(t_1)]
   -A[1-\exp(-(t_1-t_0)/\tau)].
   \]

因此，`t0` 决定阶跃发生的时刻，`t1` 决定阶跃的幅度；两者不能混淆。

## 4. North 的早期震后约束

East 和 Up 默认自由估计 `tau`。North 常有明显的早期快速变化，必须对每个站单独扫描固定 `tau`，不能将某一站的值直接用于全部站点。

### 4.1 选择规则

对候选 `tau` 逐一重拟合，并计算从 `t0` 起前 50 天的 RMS：

\[
RMS_{1-50}=\sqrt{\frac{1}{N}\sum_{1\leq dt\leq50d}(y-\hat y)^2}.
\]

选择前 50 天 RMS 最小、且全时段 RMS 没有明显恶化的候选值。建议先扫描：

```text
0.03, 0.05, 0.07, 0.08, 0.10, 0.11, 0.12, 0.125,
0.13, 0.135, 0.14, 0.15  年
```

示例：J714 的扫描选择 `tau_N=0.15` 年；J701 的扫描选择 `tau_N=0.13` 年。这些仅是站点实例，不是通用常数。

### 4.2 扫描示例

在工作目录中运行以下 Python 代码，将 `source` 换成目标测站文件：

```python
from pathlib import Path
import numpy as np
from j714_decomposition import read_tenv3, event_day_mask, fit_component_fixed_first_post_step

source = Path(r"C:\Users\xsw20\Desktop\J701.EU.tenv3")
t0 = 2016.2875
data = read_tenv3(source, 2013.0, 2024.085)
time = data["time"].to_numpy()
y = data["north_m"].to_numpy() * 1000.0
sigma = np.hypot(data["sigma_n_m"].to_numpy() * 1000.0, 2.0)
fit_mask = ~event_day_mask(time, t0)
early = fit_mask & (time > t0) & (time <= t0 + 50.0 / 365.25)

for tau in (0.03, 0.05, 0.07, 0.08, 0.10, 0.11, 0.12, 0.125, 0.13, 0.135, 0.14, 0.15):
    fit = fit_component_fixed_first_post_step(time, y, sigma, t0, fixed_tau_years=tau)
    residual = y - fit["terms"]["total"]
    early_rms = np.sqrt(np.mean(residual[early] ** 2))
    print(f"tau={tau:.3f} years, RMS_1_50={early_rms:.3f} mm, RMS_all={fit['rms_mm']:.3f} mm")
```

## 5. 运行全部 ENU 拟合和出图

PowerShell 示例。每次仅修改 `source`、`station`、`north_tau` 和输出目录：

```powershell
$source = 'C:\Users\xsw20\Desktop\J701.EU.tenv3'
$station = 'J701'
$north_tau = 0.13
$output = "analysis-output-$station-north-tau-$north_tau"

& 'D:\anaconda\python.exe' 'j714_decomposition.py' $source `
  --model north-tau-0p15-single-exp `
  --north-tau $north_tau `
  --output $output
```

输出目录包含：

- `north_fixed_tau_parameter_estimates.csv`：ENU 阶跃、`A`、`tau`、早期/全时段 RMS；
- `north_fixed_tau_parameter_summary.json`：参数摘要；
- `analysis-report.md`：模型定义及参数表；
- `figures/j714_raw_and_fitted_model.png`：ENU 总拟合；
- `figures/j714_structural_deformation.png`：构造趋势；
- `figures/j714_periodic_deformation.png`：周期项；
- `figures/j714_postseismic_deformation.png`：阶跃与震后指数项。

图件文件名中的 `j714` 是历史命名；图中实际数据和输出目录由本次传入的测站文件决定。判别站点时应以输出目录、参数表中的数值及原始文件为准。

## 6. 五年累计形变

对于第 `T=5` 年，纯震后指数累计形变为：

\[
P_5=A[1-\exp(-5/\tau)].
\]

若需要“震前至震后五年”的总地震相关位移，则加入同震阶跃：

\[
D_5=\Delta_c+P_5.
\]

这两者都**不包含**长期构造趋势与年/半年周期项。对 CSV 直接计算：

```python
import math
import pandas as pd

table = pd.read_csv("analysis-output-J701-north-tau-0.13/north_fixed_tau_parameter_estimates.csv")
post5, total5 = {}, {}
for _, row in table.iterrows():
    p5 = row.exp_A_mm * (1.0 - math.exp(-5.0 / row.tau_years))
    post5[row.component] = p5
    total5[row.component] = row.step_anchored_to_first_post_mm + p5

horizontal_post5 = math.hypot(post5["East"], post5["North"])
three_d_post5 = math.sqrt(horizontal_post5**2 + post5["Up"]**2)
horizontal_total5 = math.hypot(total5["East"], total5["North"])
three_d_total5 = math.sqrt(horizontal_total5**2 + total5["Up"]**2)

print("pure postseismic ENU (mm):", post5)
print("step + 5-year postseismic ENU (mm):", total5)
print("pure postseismic horizontal / 3D (mm):", horizontal_post5, three_d_post5)
print("step + postseismic horizontal / 3D (mm):", horizontal_total5, three_d_total5)
```

## 7. 必做质量检查

每个测站完成后必须检查：

1. `t0` 当日是否被排除：`fit_mask[event_day]` 必须为 `False`；
2. 首个完整震后日 `t1` 的模型残差是否为 0（数值精度内）；
3. North 所选 `tau` 是否是前 50 天 RMS 扫描的最优或近最优值；
4. 全时段 RMS 是否未因过小 `tau` 明显变坏；
5. East、North、Up 的总拟合、构造、周期和震后四类图是否均生成；
6. 参数表内 North 的 `tau_constraint` 是否为 `fixed`，East/Up 是否为 `free`。

## 8. 常见错误

- 把同震日坐标直接作为阶跃锚点：会使快速震后变化与同震阶跃混合，尤其会使 North 早期失配。
- 对所有站使用同一个 North `tau`：不同站的早期形变和噪声不同，应逐站扫描。
- 将 `A` 误认为五年累计量：`A` 是无限时间的渐近值；五年量应使用 `A[1-exp(-5/tau)]`。
- 把 `step + postseismic` 称为纯震后形变：两者应在报告中分别给出。
- 仅看全时段 RMS：若研究目标是早期震后过程，应优先比较 `RMS_1_50`。
