# **GAMMA：2016 熊本地震 T163 降轨两段 SLC 拼接与 IW2 处理流程**

_数据：Sentinel-1A，下降轨 T163；主影像 2016-03-27，辅影像 2016-04-20。每期由沿轨相邻的 a、b 两个 slice 组成。_

## 处理思路

```text
四个 ZIP 分别导入
→ 同日期的 a、b slice 用 SLC_cat_ScanSAR 拼接
→ 得到每期完整 IW1 / IW2 / IW3
→ 仅保留 IW2（本次采用：保留完整 IW2 的 18 个 burst）
→ SLC_mosaic_ScanSAR 生成主影像 SLC、MLI
→ DEM 与主影像查找表
→ ScanSAR_coreg.py 配准辅影像 IW2
→ SLC_mosaic_ScanSAR 生成 RSLC、RMLI
→ 干涉、滤波、解缠、地理编码、LOS
```

> **为什么只用 IW2：** 熊本 AOI（约 `130.35–131.20°E`、`32.40–33.25°N`）被 IW2 完整覆盖；IW1 不覆盖目标区，IW3 仅覆盖最西端的窄条。只保留 IW2 后，后续数据量约为三条带共同处理的三分之一。

> **本次 burst 结论：** 每个原始 slice 的 IW2 都有 9 个 burst。拼接后 IW2 共 18 个 burst。当前选择保留完整 IW2，因此不运行 `SLC_copy_ScanSAR`；若内存仍不足，可改为仅保留拼接后 burst `6–13`，即原始 a 景的 `6–9` 与 b 景的 `1–4`。

> 新版 GAMMA 已将 `*_S1_TOPS` 改名为 `*_ScanSAR`；旧命令名可作为兼容别名，但本文优先使用新版名称。

## IW 条带与 burst 的选择依据

### 1 为什么不能直接猜测 IW 条带或 burst 编号

Sentinel-1 的每个 slice 都包含 IW1、IW2、IW3，且 burst 编号在相邻 a、b slice 中都会重新从 `1` 开始。仅凭文件名不能判断熊本 AOI 落在哪个条带、哪个 burst；必须读取每个原始 SLC 的 annotation XML 中的 burst 时间和地理定位格网。

`read_S1_TOPS_SLC.py` 在默认清理模式下可能不保留 annotation XML，因此直接从原始 ZIP 解出小型 XML 文件即可；无需重新导入 SLC，也不会解压大 TIFF。

### 2 从 ZIP 提取 annotation XML

先确认 ZIP 中有 annotation：

```bash
unzip -l S1A_IW_SLC__1SSV_20160327T211629_20160327T211657_010560_00FB2D_CC6B.zip \
  '*annotation*iw1*vv*.xml'
```

批量提取四景、三条带的 VV annotation XML：

```bash
for z in S1A_IW_SLC__1SSV_*.zip; do
  case "$z" in
    *20160327T211629*) name=20160327a ;;
    *20160327T211655*) name=20160327b ;;
    *20160420T211630*) name=20160420a ;;
    *20160420T211656*) name=20160420b ;;
  esac

  for iw in iw1 iw2 iw3; do
    xml=$(unzip -Z1 "$z" | grep -i "/annotation/s1a-${iw}-slc-vv-.*\\.xml$")
    [ -n "$xml" ] && unzip -p "$z" "$xml" > "${name}.${iw}.xml"
  done
done
```

### 3 `burst_extent.py`：计算每个 burst 的经纬度范围

建立脚本：

```bash
vim burst_extent.py
```

```python
import bisect
import csv
import sys
import xml.etree.ElementTree as ET
from datetime import datetime

def tag(node, name):
    return [x for x in node.iter() if x.tag.rsplit("}", 1)[-1] == name]

def text(node, name):
    x = next(iter(tag(node, name)), None)
    return None if x is None else x.text.strip()

def tsec(s):
    return datetime.fromisoformat(s.replace("Z", "+00:00")).timestamp()

xml_file = sys.argv[1]
root = ET.parse(xml_file).getroot()

bursts = tag(root, "burst")
burst_time = sorted(tsec(text(b, "azimuthTime")) for b in bursts)
if not burst_time:
    raise RuntimeError(f"{xml_file}: 未找到 <burst>；请确认输入的是 annotation XML")

group = [[] for _ in burst_time]
for p in tag(root, "geolocationGridPoint"):
    try:
        tt = tsec(text(p, "azimuthTime"))
        lat = float(text(p, "latitude"))
        lon = float(text(p, "longitude"))
    except (TypeError, ValueError):
        continue

    i = bisect.bisect_left(burst_time, tt)
    candidates = [j for j in (i - 1, i) if 0 <= j < len(burst_time)]
    j = min(candidates, key=lambda k: abs(burst_time[k] - tt))
    group[j].append((lat, lon))

out = xml_file.replace(".xml", ".burst_extent.csv")
with open(out, "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["burst", "lat_min", "lat_max", "lon_min", "lon_max"])
    for i, pts in enumerate(group, start=1):
        if pts:
            lat, lon = zip(*pts)
            w.writerow([i, min(lat), max(lat), min(lon), max(lon)])

print(out)
```

批量生成 12 个 CSV：

```bash
for x in *.iw1.xml *.iw2.xml *.iw3.xml; do
  python3 burst_extent.py "$x"
done
```

每个输出 CSV 的列依次为：`burst, lat_min, lat_max, lon_min, lon_max`。

### 4 以熊本 AOI 筛选 burst

本次筛选 AOI 为：

```text
经度：130.35–131.20°E
纬度：32.40–33.25°N
```

运行：

```bash
for f in *.burst_extent.csv; do
  echo "===== $f ====="
  awk -F, '
    NR > 1 &&
    $3 >= 32.40 && $2 <= 33.25 &&
    $5 >= 130.35 && $4 <= 131.20 {
      print "burst=" $1, "lat=" $2 "~" $3, "lon=" $4 "~" $5
    }
  ' "$f"
done
```

实际结果显示：

```text
IW1：没有 burst 与 AOI 相交

IW2：
  a slice：burst 7–9 与 AOI 相交
  b slice：burst 1–3 与 AOI 相交

IW3：只在 AOI 最西端（约 130.35–130.48°E）出现窄条交集，
     不能增加熊本主震区的有效覆盖。
```

因此选择 **IW2**。若需要进一步裁切，在相交 burst 两端各保留一个 burst：原始 a slice 取 `6–9`，原始 b slice 取 `1–4`；完成 `SLC_cat_ScanSAR` 后对应为拼接 IW2 的 `6–13`。

## 配准

### 1 `read_S1_TOPS_SLC.py`：分别导入四个原始 ZIP

```bash
read_S1_TOPS_SLC.py S1A_IW_SLC__1SSV_20160327T211629_20160327T211657_010560_00FB2D_CC6B.zip --root_name 20160327a --pol VV
read_S1_TOPS_SLC.py S1A_IW_SLC__1SSV_20160327T211655_20160327T211722_010560_00FB2D_9908.zip --root_name 20160327b --pol VV

read_S1_TOPS_SLC.py S1A_IW_SLC__1SSV_20160420T211630_20160420T211658_010910_01059E_8779.zip --root_name 20160420a --pol VV
read_S1_TOPS_SLC.py S1A_IW_SLC__1SSV_20160420T211656_20160420T211723_010910_01059E_F15F_2.zip --root_name 20160420b --pol VV
```

每景生成：

```text
YYYYMMDDa.vv.iw1.slc / .slc.par / .slc.tops_par
YYYYMMDDa.vv.iw2.slc / .slc.par / .slc.tops_par
YYYYMMDDa.vv.iw3.slc / .slc.par / .slc.tops_par
YYYYMMDDa.vv.SLC_tab
YYYYMMDDa.burst_number_table
```

### 2 `SLC_cat_ScanSAR`：将同日期 a、b slice 拼接为完整三条带

先建立两个拼接输出 tab。每行依次是 `SLC SLC_par TOPS_par`，顺序必须为 IW1、IW2、IW3。

```bash
for d in 20160327 20160420; do
  for iw in 1 2 3; do
    echo "${d}.cat.vv.iw${iw}.slc ${d}.cat.vv.iw${iw}.slc.par ${d}.cat.vv.iw${iw}.slc.tops_par"
  done > "${d}.cat.vv.SLC_tab"
done
```

拼接：

```bash
SLC_cat_ScanSAR 20160327a.vv.SLC_tab 20160327b.vv.SLC_tab 20160327.cat.vv.SLC_tab
SLC_cat_ScanSAR 20160420a.vv.SLC_tab 20160420b.vv.SLC_tab 20160420.cat.vv.SLC_tab
```

输出为每期连续的 IW1、IW2、IW3 burst SLC，例如：

```text
20160327.cat.vv.iw1.slc / .par / .tops_par
20160327.cat.vv.iw2.slc / .par / .tops_par
20160327.cat.vv.iw3.slc / .par / .tops_par
20160327.cat.vv.SLC_tab
```

> `SLC_cat_ScanSAR` 产生完整拼接文件，主要增加磁盘占用；它不是导致解缠内存过大的关键步骤。实际减小后续处理规模的是只保留 IW2，或进一步按 burst 裁切。

### 3 只建立 IW2 的 SLC tab

```bash
grep '\.iw2\.' 20160327.cat.vv.SLC_tab > 20160327.cat.iw2.SLC_tab
grep '\.iw2\.' 20160420.cat.vv.SLC_tab > 20160420.cat.iw2.SLC_tab

cat 20160327.cat.iw2.SLC_tab
cat 20160420.cat.iw2.SLC_tab
```

每个 tab 都必须只有一行，例如：

```text
20160327.cat.vv.iw2.slc 20160327.cat.vv.iw2.slc.par 20160327.cat.vv.iw2.slc.tops_par
```

### 4 `SLC_copy_ScanSAR`：可选 burst 裁切

本次决定保留完整 IW2，因此**跳过本步骤**，后面直接使用：

```text
20160327.cat.iw2.SLC_tab
20160420.cat.iw2.SLC_tab
```

如后续解缠内存仍不足，则仅保留熊本 AOI 的 8 个 burst。建立单行 `BURST_tab`：

```bash
printf '6 13\n' > 20160327.keep.BURST_tab
printf '6 13\n' > 20160420.keep.BURST_tab

printf '%s\n' '20160327.keep.iw2.slc 20160327.keep.iw2.slc.par 20160327.keep.iw2.slc.tops_par' > 20160327.keep.iw2.SLC_tab
printf '%s\n' '20160420.keep.iw2.slc 20160420.keep.iw2.slc.par 20160420.keep.iw2.slc.tops_par' > 20160420.keep.iw2.SLC_tab

SLC_copy_ScanSAR 20160327.cat.iw2.SLC_tab 20160327.keep.iw2.SLC_tab 20160327.keep.BURST_tab
SLC_copy_ScanSAR 20160420.cat.iw2.SLC_tab 20160420.keep.iw2.SLC_tab 20160420.keep.BURST_tab
```

> 新版 `SLC_copy_ScanSAR` 的第三个参数是 `BURST_tab` 文件，**不要**写旧语法 `1 6 1 13`。成功时应显示输出 8 个 burst。

若执行了裁切，本章下文所有 `*.cat.iw2.SLC_tab` 都替换成相应 `*.keep.iw2.SLC_tab`。

### 5 `SLC_mosaic_ScanSAR`：主影像 IW2 生成标准 SLC

```bash
SLC_mosaic_ScanSAR 20160327.cat.iw2.SLC_tab 20160327.slc 20160327.slc.par 10 2
multi_look 20160327.slc 20160327.slc.par 20160327.mli 20160327.mli.par 10 2
```

输出：

```text
20160327.slc
20160327.slc.par
20160327.mli
20160327.mli.par
```

检查尺寸：

```bash
grep -E 'range_samples|azimuth_lines' 20160327.mli.par
dispwr 20160327.mli "$(grep 'range_samples' 20160327.mli.par | awk '{print $2}')"
```

### 6 DEM、查找表与主影像 SAR 坐标 DEM

按 `los.md` 的 DEM、`gc_map2`、`pixel_area`、`gc_map_fine`、`geocode` 步骤执行；输入主影像改为：

```text
20160327.mli
20160327.mli.par
```

必须得到后续 ScanSAR 配准使用的主影像 SAR 坐标 DEM，例如：

```text
DEM/20160327.hgt
```

### 7 `ScanSAR_coreg.py`：仅配准辅影像 IW2

不要复制或使用旧的三行 `20160420.vv.RSLC_tab`。建立只有 IW2 的输出 RSLC tab：

```bash
printf '%s\n' '20160420.cat.vv.iw2.rslc 20160420.cat.vv.iw2.rslc.par 20160420.cat.vv.iw2.rslc.tops_par' > 20160420.cat.iw2.RSLC_tab
```

配准：

```bash
ScanSAR_coreg.py 20160327.cat.iw2.SLC_tab 20160327 \
                 20160420.cat.iw2.SLC_tab 20160420 \
                 20160420.cat.iw2.RSLC_tab \
                 DEM/20160327.hgt 10 2
```

输出：

```text
20160420.cat.vv.iw2.rslc
20160420.cat.vv.iw2.rslc.par
20160420.cat.vv.iw2.rslc.tops_par
20160420.cat.iw2.RSLC_tab
```

### 8 `SLC_mosaic_ScanSAR`：生成辅影像 RSLC 与 RMLI

```bash
SLC_mosaic_ScanSAR 20160420.cat.iw2.RSLC_tab 20160420.rslc 20160420.rslc.par 10 2
multi_look 20160420.rslc 20160420.rslc.par 20160420.rmli 20160420.rmli.par 10 2
```

> 若这里出现 `cannot open ...iw1.rslc`，说明错误地使用了三行的 `20160420.vv.RSLC_tab`；必须改回一行的 `20160420.cat.iw2.RSLC_tab`。

## 干涉

### 1 基线与地形相位

```bash
base_orbit 20160327.slc.par 20160420.rslc.par 20160327_20160420.base
phase_sim 20160327.slc.par 20160327_20160420.off 20160327_20160420.base \
          DEM/20160327.hgt 20160327_20160420.sim_unw
```

> `20160327_20160420.off` 使用 `ScanSAR_coreg.py` 实际生成的 offset 文件名；如名称不同，以实际文件名替换。

### 2 `SLC_diff_intf`：生成差分干涉图

```bash
SLC_diff_intf 20160327.slc 20160420.rslc \
              20160327.slc.par 20160420.rslc.par \
              20160327_20160420.off \
              20160327_20160420.sim_unw \
              20160327_20160420.diff_int 10 2
```

### 3 后续滤波、解缠与地理编码

从 `los.md` 的 `adf → cc_wave → rascc_mask → mcf → geocode_back → dispmap_LOS → data2geotiff` 顺序继续。

所有命令中的宽度必须从当前裁切后文件的 `.par` 读取，不能再沿用原三条带处理时的 `6763`：

```bash
grep -E 'range_samples|azimuth_lines' 20160327.mli.par
```

例如：

```bash
adf 20160327_20160420.diff_int \
    20160327_20160420.diff_int_filt \
    20160327_20160420.diff.cc \
    <当前MLI宽度> 0.8
```

## 关键检查

- `SLC_cat_ScanSAR` 后，IW2 `tops_par` 应显示 18 个 burst。
- 只保留完整 IW2 时，不需要 `SLC_copy_ScanSAR`。
- 如执行 burst 裁切，`BURST_tab` 为一行 `6 13`，输出应为 8 个 burst。
- `ScanSAR_coreg.py` 的主、辅、RSLC tab 都必须是**仅 IW2 的一行 tab**。
- `SLC_mosaic_ScanSAR` 生成 RSLC 时必须输入 `20160420.cat.iw2.RSLC_tab`，不能输入旧三行 RSLC tab。
- 所有后续影像显示、滤波、解缠命令均使用裁切后的实际宽度。
