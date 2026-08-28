# **GAMMA**

_此部分内容是ALOS-2的GAMMA部分内容_

## 用`par_EORC_PALSAR` 转成gamma可以识别的格式

### 1 `ls` 看一下path23文件夹下面的文件

- 0000527273
- 0000527274

### 2 `mkdir caL` 建立处理文件夹

### 3 `cd caL/` 进入处理文件夹

### 4 `mkdir DEM` 建立对应DEM文件夹(可以手动)

### 5 `par_EORC_PALSAR`

```bash
par_EORC_PALSAR ../0000527273/0000018834_0000527273_ALOS2034752920-150114_150114_UBSL1.1__D/0000018834_0000527273_ALOS2034752920-150114_UBSL1.1__D/0000527273/LED-ALOS2034752920-150114-UBSL1.1__D 20150114.slc.par ../0000527273/0000018834_0000527273_alos2034752920-150114_ubsl1.1__D/0000018834_0000527273_ALOS2034752920-150114_UBSL1.1__D/0000527273/IMG-HH-ALOS2034752920-150114-UBSL1.1__D 20150114.slc
```

### 6 `ls`

- 20150114.slc
- 20150114.slc.par
- 20160420.slc
- 20160420.slc.par
- DEM

### 7 做DEM

```bash
cd DEM/
```

### 8 `ls` -空

### 9 `multi_look`

```bash
multi_look ../20150114.slc ../20150114.slc.par 20150114.mli 20150114.mli.par 7 6
```

### 10 `dispwr`

```bash
dispwr 20150114.mli 3846
```

### 11 `SLC_corners` 看经纬度

```bash
SLC_corners 20150114.slc.par 1000
```

- longitude: 32.25 - 33.07
- latitude: 130.22 - 131.08

### 12 在`FABDEM`操作

```bash
cd ~/FABDEM/
```

```bash
python fabdem2gamma.py -lon 130 132 -lat 32 34 -o srtm
```

```bash
mv srtm/DEM_32_34_130_132.dem* ~/insar_kumamoto2016/path23/caL/DEM/
```

```bash
cd ~/insar_kumamoto2016/path23/caL/DEM/
```

```bash
ls
```

- 20150114.mli
- 20150114.slc.par
- DEM_32_34_130_132.dem
- DEM_32_34_130_132.dem.par

### 13 制作查找表

```bash
gc_map2 20150114.mli.par DEM_32_34_130_132.dem.par DEM_32_34_130_132.dem EQA.dem.par EQA.dem 20150114.lt 1 1 ls_map - inc - -  u v psi pix
```

```bash
pixel_area 20150114.mli.par EQA.dem.par EQA.dem 20150110.lt ls_map inc pix_sigma0  pix_gamma0
```

```bash
create_diff_par 20150114.mli.par - 20150114.diff_par 1 0
```

```bash
offset_pwrm pix_sigma0 20150114.mli 20150114.diff_par offs snr 256 256 offsets 2 32 32 0.2
```

```bash
offset_fitm offs snr 20150114.diff_par - -  0.6 6
```

```bash
gc_map_fine 20150114.lt 2889 20150114.diff_par 20150114.lt_fine 1
```

```bash
geocode_back 20150114.mli 3846 20150114.lt_fine geo_20150114.mli 2889 2703 1 0
```

```bash
dispwr geo_20150114.mli 2889
```

```bash
geocode 20150114.lt_fine EQA.dem 2889 20150114.hgt 3846 6218 1 0
```

### 14 配准

```bash
cd ../
```

_cd这一步还是在DEM文件下进行_

```bash
SLC_coreg 20160420.slc 20160420.slc.par 20160420.rslc 20160420.rslc.par 20160420.rmli 20160420.rmli.par 20150114.slc 20150114.slc.par DEM/20150114.hgt 7 6
```

```bash
ls
```

- 20160420.rmli
- 20160420.rmli.bmp
- 20150114.slc
- 20150114.slc.par
- 20160420.rslc
- 20410220.rmli.par
- 20160420.rslc.par
- 20160420.rslc.coreg_quality
- 20160420.slc
- 20160420.slc.lt_fine
- 20160420.slc.off
- 20160420.slc.par
- DEM

### 当配准有问题的时候

1、`create_offset`
2、`init_offset_orbit`
3、`init_offset` 精化后两位
4、`offset_pwr`
5、`offset_fit` < 0.1 is OK
6、`SLC_int/SLC_interp_lt`

### 15 差分干涉\_接在配准后

```bash
create_offset 20150114.slc.par  20160420.rslc.par 20150114_20160420.off 1 7 6 0
```

```bash
base_orbit 20150114.slc.par 20160420.slc.par 20150114_20160420.base
```

```bash
phase_sim 20150114.slc.par 20150114_20160420.off 20150114_20160420.base DEM/20150114.hgt 20150114_20160420.phase_sim.unw
```

```bash
SLC_diff_intf  20150114.slc 20160420.rslc 20150114.slc.par  20160420.rslc.par  20150114_20160420.off 20150114_20160420.phase_sim.unw  20150114_20160420.diff_int 7 6
```

**基线精化：**

```bash
base_init 20150114.slc.par 20160420.slc.par 20160420.slc.off 20150114_20160420.diff_int 20150114_20160420.base_new 4
```

**老基线+新基线：**

```bash
 base_add 20150114_20160420.base 20150114_20160420.base_new 20150114_20160420.base_o 1
```

**更新地形相位(输入的数据用base_add的输出):**

```bash
phase_sim 20150114.slc.par 20150114_20160420.off 20150114_20160420.base_o DEM/20150114.hgt 20150114_20160420.phase_sim.unw
```

**更新差分干涉结果：**

```bash
SLC_diff_intf 20150114.slc 20160420.rslc 20150114.slc.par 20160420.rslc.par 20150114_20160420.off 20150114_20160420.phase_sim.unw 20150114_20160420.diff_int 7 6
```
``` bash
dismph_pwr 20150114_20160420.diff_int 20160420.rmli 3846
```
**减少蓝色**
``` bash
create_diff_par  20160420.rmli.par  - 20150114.diff_par 1 0
```
``` bash
quad_fit  20150114_20160420.diff.unw 20150114.diff_par  4 4 - plot_data 0 pmodel
```
``` bash
dis2dt_pwr  20150114_20160420.diff.unw  pmodel  3846 3846
```
``` bash
dis2dt_pwr  20150114_20160420.diff.unw  pmodel  20160420.rmli 3846 3846 1 0 0 0  -3.14  3.14 1
```
``` bash
sub_phase   20150114_20160420.diff.unw  pmodel  20150114.diff_par  20150114_20160420.diff.unw1 0 0 0 
```
``` bash
dis2dt_pwr  20150114_20160420.diff.unw  20150114_20160420.diff.unw1  20160420.rmli 3846 3846 1 0 0 0  -3.14  3.14 1
```

### 16 `mcf`滤波
