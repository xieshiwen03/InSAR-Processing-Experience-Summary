# **GAMMA + GMT：InSAR LOS 手动 ROI 保留与剔除流程**

_适用数据：已完成解缠、地理编码和 LOS 换算后的 Sentinel-1 InSAR LOS GeoTIFF，例如 `geo_20260722_20260803.diff.los.tif`。_

## 处理思路

```text
GAMMA：adf → cc_wave → rascc_mask → mcf → geocode_back → dispmap_LOS → data2geotiff
→ 最终 LOS GeoTIFF
→ grd2xyz 输出有效像元
→ gmt select 按手动多边形筛选
→ xyz2grd 按原网格重建（未保留像元为 NaN）
→ grdconvert 导出中间 GeoTIFF
→ gdal_translate 写入 EPSG:4326 / NoData 元数据
```

本流程有两种相反目的：

| 目的           | `gmt select` 的写法          | 结果                             |
| -------------- | ---------------------------- | -------------------------------- |
| **仅保留 ROI** | `-Fkeep_roi_asc.xy -fg`      | 仅多边形内部保留 LOS，外部为 NaN |
| **剔除 ROI**   | `-Fremove_hinagu.xy -If -fg` | 多边形内部为 NaN，外部保留 LOS   |

> `-If` 是“反选多边形”的核心开关：保留 ROI 时**不能**写它；剔除 ROI 时必须写它。

> 本流程只处理最终 LOS 产品，不回写 `*.diff.unw`、`*.diff_int_filt` 或 `*.diff.cc`，也不替代前面基于相干性的掩膜和解缠质检。

## 1. 运行前检查

### 1.1 输入文件

工作目录中至少需要：

```text
geo_YYYYMMDD_YYYYMMDD.diff.los.tif    # GAMMA 的最终 LOS GeoTIFF
keep_roi_asc.xy                       # 仅保留区域时使用
remove_hinagu.xy                      # 剔除区域时使用
```

先确认 LOS 网格可被 GMT 和 GDAL 正确读取：

```bash
src=geo_20260722_20260803.diff.los.tif

gmt grdinfo "$src"
gdalinfo "$src"
```

重点检查：

- 坐标系为 `EPSG:4326` / WGS84；
- 经度、纬度范围正确；
- 网格间隔、行列数合理；
- 原始数据的 NoData 为 `nan`（若已写入）。

> 若出现 Conda/PROJ 的 `proj.db` 版本警告，但 `gmt grdinfo` 仍能正确列出范围、间隔和数据最小/最大值，说明该警告不是本流程的掩膜错误。若读不到网格或坐标系异常，再单独修复 GMT 与 Conda 的 PROJ 环境。

### 1.2 多边形文件格式

GMT 使用 WGS84 十进制度，列顺序必须为：

```text
经度 纬度
```

一个闭合多边形的最简格式如下。最后一行必须与第一行相同：

```text
> roi_1
130.5000000000 32.6000000000
130.6000000000 32.6000000000
130.6000000000 32.7000000000
130.5000000000 32.7000000000
130.5000000000 32.6000000000
```

多个 ROI 写入同一个文件时，以 `>` 开始下一段。例如：

```text
> roi_1
lon_1 lat_1
...
lon_1 lat_1
> roi_2
lon_2 lat_2
...
lon_2 lat_2
```

`gmt select -F多边形文件` 会把所有闭合多边形视为并集：落在任意一个多边形内部的像元均可通过。手动绘制后应至少检查：坐标单位、经纬度顺序、首尾闭合、重复多边形和自交。

## 2. 仅保留手动 ROI

以下示例以升轨 `20260722_20260803` 为例。先将 `keep_roi_asc.xy` 复制到该升轨数据目录。

### 2.1 由 LOS GeoTIFF 重建 NaN 掩膜版网格

```bash
src=geo_20260722_20260803.diff.los.tif
roi=keep_roi_asc.xy
out=geo_20260722_20260803.diff.los_keep_roi
```
```bash
gmt grd2xyz "geo_20260722_20260803.diff.los.tif" -s --FORMAT_FLOAT_OUT=%.12g | gmt select -F"keep_roi_asc.xy" -fg |  gmt xyz2grd -R"geo_20260722_20260803.diff.los.tif" -fg -G"geo_20260722_20260803.diff.los_keep_roi.nc"
```

各参数含义：

- `grd2xyz -s`：只输出原 LOS 中非 NaN 的 `longitude latitude LOS` 像元；
- `--FORMAT_FLOAT_OUT=%.12g`：避免经纬度与 LOS 值写成 ASCII 时精度不足；
- `select -F"$roi" -fg`：仅让 ROI 内的地理坐标点通过；
- `xyz2grd -R"$src" -fg`：继承原始 GeoTIFF 的范围、间隔、行列数和注册方式；
- 没有通过筛选的节点在 `${out}.nc` 中自动为 **NaN**。

> 不要先运行 `gmt grd2xyz ... > los.xyz`。完整 LOS 网格可能有数千万节点，直接流式管道处理能避免生成巨大的中间文本文件。

### 2.2 导出带 NoData 的 GeoTIFF

若当前 GDAL 不能直接读取 GMT 写出的 `.nc`（例如提示缺少 `gdal_HDF5.so`），不要直接对 `.nc` 执行 `gdal_translate`。先由 GMT 输出 GDAL GeoTIFF，再由 GDAL 写入坐标系与 NoData 元数据：

```bash
gmt grdconvert "geo_20260722_20260803.diff.los_keep_roi.nc"  -G"geo_20260722_20260803.diff.los_keep_roi_gmt.tif=gd:GTiff"
```
```bash
gdal_translate -of GTiff -a_srs EPSG:4326 -a_nodata nan "geo_20260722_20260803.diff.los_keep_roi_gmt.tif" "geo_20260722_20260803.diff.los_keep_roi.tif"
```

最终分析与制图文件为：

```text
geo_20260722_20260803.diff.los_keep_roi.tif
```

## 3. 手动剔除指定区域

以下示例以降轨 `20260716_20260728` 为例。`remove_hinagu.xy` 是需要删除的闭合多边形；它可以有一段或多段。

### 3.1 由 LOS GeoTIFF 重建 NaN 掩膜版网格

```bash
src=geo_20260716_20260728.diff.los.tif
remove=remove_hinagu.xy
out=geo_20260716_20260728.diff.los_manualmask
```
```bash
gmt grd2xyz "geo_20260716_20260728.diff.los.tif" -s --FORMAT_FLOAT_OUT=%.12g |  gmt select -F"$remove" -If -fg | gmt xyz2grd -R"geo_20260716_20260728.diff.los.tif" -fg  -G"geo_20260716_20260728.diff.los_manualmask.nc"
```

与“仅保留 ROI”相比，唯一的逻辑差异是 `-If`：

```text
保留 ROI：select -Fkeep_roi_asc.xy -fg
剔除 ROI：select -Fremove_hinagu.xy -If -fg
```

`-If` 使 `gmt select` 只输出所有删除多边形外部的点。因此，删除多边形内部没有 XYZ 点写入新网格，结果为 NaN。

### 3.2 导出 GeoTIFF

```bash
gmt grdconvert "geo_20260716_20260728.diff.los_manualmask.nc" -G"geo_20260716_20260728.diff.los_manualmask_gmt.tif=gd:GTiff"
```
```bash
gdal_translate -of GTiff -a_srs EPSG:4326 -a_nodata nan "geo_20260716_20260728.diff.los_manualmask_gmt.tif" "geo_20260716_20260728.diff.los_manualmask.tif"
```

最终输出为：

```text
geo_20260716_20260728.diff.los_manualmask.tif
```

## 4. NaN、0 与绘图显示

### 4.1 推荐：NaN 作为 ROI 外部/剔除区

NaN 表示“该像元没有有效 LOS 值”，适合后续反演、统计、计算范围和科学绘图。它不会被当作真实的 `0 m` 位移。

绘制 NaN 掩膜版时，在 LOS `grdimage` 后加入 `-Q`：

```bash
gmt grdimage "geo_20260716_20260728.diff.los_manualmask.tif" -R... -J... -C你的LOS色标.cpt -Q
```

`-Q` 会让网格 NaN 节点透明；这样底图可见，ROI 外部不会显示为 CPT 的黑色、灰色或背景色。

### 4.2 可选：仅为展示建立 0 值版

若作图软件无法处理透明 NaN，或确实需要让所有掩膜位置显示为色标中的 `0`，可另建一个 0 值版：

```bash
gmt grdmath "geo_20260716_20260728.diff.los_manualmask.nc" 0 AND = "geo_20260716_20260728.diff.los_manualmask_zero.nc"
```
```bash
gmt grdconvert "geo_20260716_20260728.diff.los_manualmask_zero.nc" -G"geo_20260716_20260728.diff.los_manualmask_zero_gmt.tif=gd:GTiff"
```
```bash
gdal_translate -of GTiff -a_srs EPSG:4326 "geo_20260716_20260728.diff.los_manualmask_zero_gmt.tif" "geo_20260716_20260728.diff.los_manualmask_zero.tif"
```

`AND` 的逻辑是：原网格值为 NaN 时取第二个输入 `0`；原网格有值时保留原值。因此该操作会把**所有** NaN（包括原始无覆盖区）改成 `0`。

> `geo_20260716_20260728.diff.los_manualmask_zero.tif` 只能作为展示辅助文件。不要用于 LOS 统计、断层反演、误差分析或任何将 `0` 解释为真实无位移的处理。

## 5. 结果检查

每次处理后都检查源网格和新网格：

```bash
gmt grdinfo "geo_20260716_20260728.diff.los.tif" "geo_20260716_20260728.diff.los_manualmask.nc"
gdalinfo "geo_20260716_20260728.diff.los_manualmask.tif"
```

正确结果应满足：

- 新、旧网格的 `x_min/x_max`、`y_min/y_max`、`x_inc/y_inc`、行列数和注册方式一致；
- 新网格的 `v_min/v_max` 可以变化，这是 ROI 筛选后的正常现象；
- NaN 掩膜版 GeoTIFF 的 `NoData Value` 应为 `nan`；
- 制图时 ROI 外部不应产生黑/灰色的伪值；应使用 `-Q` 透明显示；
- 使用 `keep_roi_asc.xy` 时只有 ROI 内显示 LOS；使用 `remove_hinagu.xy` 时只有删除区内部不显示 LOS。

## 6. 常见问题

### 1 `gdal_translate` 提示 `.nc` 不支持或缺少 `gdal_HDF5.so`

原因：当前 GDAL 环境没有 NetCDF/HDF5 驱动，无法读取 GMT 的 NetCDF 输出。

处理：始终使用本流程中的两步导出：

```text
GMT grdconvert：.nc → *_gmt.tif
GDAL gdal_translate：*_gmt.tif → 最终 .tif
```

不要把 `.nc` 直接交给当前 GDAL。

### 2 剔除区或 ROI 外部在图上显示黑色/灰色

原因：NaN 已正确生成，但绘图命令没有处理 NaN 透明度，或 CPT 为 NaN 指定了颜色。

处理：使用 NaN 版 GeoTIFF，并在 `gmt grdimage` 中加入 `-Q`。

### 3 保留区与删除区的结果完全相反

原因：`-If` 使用位置错误。

处理：

```text
仅保留 ROI：不写 -If
手动剔除 ROI：写 -If
```

### 4 输出几乎为空或完全为空

依次检查：

1. 多边形文件坐标是否为 `经度 纬度`，而非 `纬度 经度`；
2. 坐标是否仍为度分秒，尚未转成十进制度；
3. 多边形是否首尾闭合；
4. 多边形范围是否与 `gmt grdinfo "$src"` 的范围相交；
5. 命令中是否遗漏地理坐标标识 `-fg`。

### 5 生成 `los.xyz` 后磁盘很快被占满

原因：全分辨率网格导出为文本的体积很大。

处理：使用本流程的管道命令，不建立完整 `los.xyz` 中间文件。

## 7. 处理记录与数据保存

每一次手动筛选都应保留以下内容，以保证图件和数据可复现：

```text
原始 geo_*.diff.los.tif
所用多边形文件（keep_roi_*.xy 或 remove_*.xy）
本流程命令或 shell 脚本
输出的 NaN 掩膜版 .nc / .tif
若生成，仅展示用 *_zero.tif
使用的相干性阈值、处理日期及 ROI 的用途说明
```

手动 ROI 用于限定研究范围或移除已确认无效区域时，应在图注、方法或处理记录中说明其用途。若要让数据有效区自然反映质量，应优先结合相干性、解缠质量和几何阴影/叠掩掩膜，而非用 0 值替代无数据。

## 参考文档

- [GMT `select`：按多边形选择/反选表格点](https://docs.generic-mapping-tools.org/latest/gmtselect.html)
- [GMT `xyz2grd`：由 XYZ 重建网格](https://docs.generic-mapping-tools.org/latest/xyz2grd.html)
- [GMT `grdconvert`：网格格式转换与 GDAL 输出](https://docs.generic-mapping-tools.org/latest/grdconvert.html)
- [GMT `grdimage -Q`：NaN 节点透明](https://docs.generic-mapping-tools.org/latest/grdimage.html)
- [GMT `grdmath AND`：以 0 替代 NaN](https://docs.generic-mapping-tools.org/latest/grdmath.html)
