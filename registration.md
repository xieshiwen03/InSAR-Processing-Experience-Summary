# GAMMA

_先把文件分成升轨和降轨，方便后续处理_

### 1 `S1_import_SLC_from_zipfiles`/`read _S1_TOPS_SLC.py`主影像转gamma

    read_S1_TOPS_SLC.py S1C_IW_SLC_1SDV_20260722T091318_20260722T008654_01125E_027E0E_4884.zip --root_name 20260722 --pol VV

输入：1、input(原始zip文件)、--root_name name(用时间命名)、 --pol txt(选择极化方式)、其他的比如brust文件可以忽略，只整个处理也行

### 2 `ls`看一下生成的文件

- 20260722.vv.iw1.slc
- 20260722.vv.iw1.slc.par
- 20260722.vv.iw1.slc.tops_par
- 20260722.vv.iw2.1.slc
- 20260722.vv.iw2.1.slc.par
- 20260722.vv.iw2.1.slc.tops_par
- calibration-s1c-iw2-slc-vv-20260722t091318-20260722t091344-008654-01125e-005.xml
- noise-s1c-iw2-slc-vv-20260722t091318-20260722t091344-008654-01125e-005.xml
- s1c-iw2-slc-vv-20260722t091318-20260722t091344-008654-01125e-005.tiff
- s1c-iw2-slc-vv-20260722t091318-20260722t091344-008654-01125e-005.xml

### 3 `multi_look`

    multi_look 20260722.vv.iw1.slc 20260722.vv.iw1.slc.par 20260722.vv.iw1.mli 20260722.vv.iw1.mli.par 10 2

输入：1、原始文件 2、原始文件参数 3、输出文件 4、输出文件参数 5、多频 6、多行

> _注意：要记住生成的width和lines，后面处理时用到_

### 4 `dispwr`查看影像

    dispwr 20260722.vv.iw1.mli 2066

输入：dispwr 要看的影像名字 width

### 5 `SLC_mosaic_S1_TOPS`把三个条带拼在一起

    SLC_mosaic_S1_TOPS 20260722.VV.SLC_tab 20260722.slc 20260722.slc.par 10 2

### 6 `multi_look` 检查是否拼在了一起

    multi_look 20260722.slc 20260722.slc.par 20260722.mli 20260722.mli.par 10 2

> _记得看宽度width_

### 7 `dispwr`查看影像

    dispwr 20260722.mli 6763

### 8`raspwr`影像变成图片

    raspwr 20260722.mli 6763 1 0 1 1 1.0

输入：data width start nlines pixavx pixavy scale

### 9 `eog`看图片

    eog 20260722.mli.bmp

注意：bmp是`raspwr`生成的图片

### 10`SLC_corners`查看经纬度，目的

    SLC_corners 20260722.slc.par 1000

输入：SLC_corners 配置文件 terra_alt(1000随便选的)

### 11 在`FABDEM`中找到相应经纬度的DEM

> _查看经纬度，注意DEM的经纬度要比SAR影像的经纬度大_  
> 得到的结果：33.75 31.64 130.19 133.26，所以DEM选择31 34 130 134

### 12`DEM`

##### 在`FABDEM`运行脚本

    python fabdem2gamma.py
    python fabdem2gamma.py -h
    python fabdem2gamma.py -lon 130 134 -lat 31 34 -o .
    pwd（记住路径）

##### 回到最开始的终端

    mv /media/xieshiwen/Expansion/FABDEM/DEM_31_34_130_134.dem* .
    mkdir DEM
    mv DEM_31_34_130_134.dem* DEM/
    mv 20260722.mli* DEM/
    cd DEM/
    ls

`ls`结果：

- 20260722.mli
- 20260722.mli.bmp
- 20260722.mli.par
- DEM_31_34_130_134.dem
- DEM_31_34_130_134.dem.par

### 13`gc_map2` 查找表

    gc_maps 20260722.mli.par DEM_31_34_130_134.dem.par DEM_31_34_130_134.dem EQA.dem.par DEM.dem 20260722.lt 1 1 ls_map -  inc - -  20260722.sim_sar u v psi pix

> _看 DEM segment:width和 nlines_  
> _EQA.dem.par是DEM的配置文件_

### 14`pixel_area`检查SAR和DEM是否一样了

    pixel_area 20260722.mli.par EQA.dem.par DEM.dem 20260722.lt  ls_map  inc  pux_sigma0 pix_gamma0

### 15`dis2pwr`看对比

    dis2pwr 20260722.mli  pix_sigma0 6763 6763

> **这两个数据的宽度是从哪里看到的，为什么都是6763？**  
> **range pixels: 6763 azimuth lines: 6560**

### 16`create_diff_par`存偏移量

    create_diff_par 20260722.mli.par - 20260722.diff.par  1 0

### 17`cat 20260722.diff_par`查看偏移量

### 18`oddset_pwrm`

    offset_pwrm pix_sigma0 20260722.mli 20260722.diff_par  offd snr 256 256 offsets 2 32 32 0.2

> _`number of offsets above correlation threshold: 573 of 1024` 要超过10%_  
> _效果不好时：256 256 调大、32 32 调小_

### 19`oddset_fitm`

    offset_fitm offs snr  20260722.diff_par  - -  0.3 3

> _看结果：`final model fit std. dev. (samples) range: 0.0709 azimuth:0.0381`_  
> **range小于0.1、azimuth小于0.2**

### 20`gc_map_fine`修正查找表

    grep 'width' EQA.dem.par (获得宽度10872)
    gc_map_fine 20260722.lt 10872 20260722.diff_par 20260722.lt_fine 1

### 21 `pixel_area`再次对比一下

    pixel_area 20260722.mli.par EQA.dem.par DEM.dem 20260722.lt_fine  ls_map  inc  pux_sigma1 pix_gamma1

> _看range pixels:6763_

### 22 `dis2pwr 20260722.mli  pix_sigma1 6763 6763` 查找表OVER

### 23`geocode`地理编码

    grep 'width' EQA.dem.par --10872
    grep 'range' 20260722.mli.par --6763
    grep 'azimuth' 20260722.mli.par --6560
    geocode 20260722.lt_fine DEM.dem 10872 20260722.hgt 6763 6560 1 0

### 24`disdt_pwr`看浮点数

    dis2pwr 20260722.hgt 20260722.mli 6763 1 0 0 500 1

> _50/500都行_

### 25`ls`查看文件夹文件

- 20260722.diff_par
- 20260722.lt
- 20260722.mli
- 20260722.mli.par
- DEM_31_34_130_134.dem
- DEM.dem
- inc
- offs
- pix
- pix_gamma1
- pix_sigma1
- snr
- v
- 20260722.hgt
- 20260722.lt_fine
- 20260722.mli.bmp
- 20260722.sim_sar
- DEM_31_34_130_134.dem.par
- EQA.dem.par
- ls_map
- offsets
- pix_gamma0
- pix_sigma0
- psi
- u#

### 26 `geocode_back` 反投影

    geocode_back 20260722.mli 20260722.lt_fine geo_20260722.mli 10872

### 27`grep 'lines' EQA.dem.par` --7314

### 28`diispwr geo_20260722.mli `看结果

    dispwr geo_20260722.mli 10872

### 29`ScanSAR_`配准

### 30`S1_import_SLC_from_zipfiles`/`read _S1_TOPS_SLC.py`做辅影像

    read_S1_TOPS_SLC.py S1C_IW_1SDV_20260809T091319_20260809T091346_011833_BE60.zip --pol VV

### 31`ls`

- 20260722.burst_number_table
- 20260722.vv.SLC_tab
- BURST_tab
- 20260722.slc
- 20260803.burst_number_table
- DEM
- 20260722.slc.par
- 20260803.vv.iw1.slc
- read_S1_TOPS_SLC.20260811T192531.log
- 20260722.vv.iw1.slc
- 20260803.vv.iw1.slc.par
- read_S1_TOPS_SLC.20260811T192548.log
- 20260722.vv.iw1.slc.par
- 20260803.vv.iw1.slc.tops_par
- read_S1_TOPS_SLC.20260811T204701.log
- 20260722.vv.iw1.slc.tops_par
- 20260803.vv.iw2.slc
- S1C_IW_SLC\_\_1SDV_20260722T091318_20260722T091345_008654_01125E_4884.burst_number_table
- 20260722.vv.iw2.slc
- 20260803.vv.iw2.slc.par
- S1C_IW_SLC\_\_1SDV_20260722T091318_20260722T091345_008654_01125E_4884.SAFE
- 20260722.vv.iw2.slc.par
- 20260803.vv.iw2.slc.tops_par
- S1C_IW_SLC\_\_1SDV_20260722T091318_20260722T091345_008654_01125E_4884.zip
- 20260722.vv.iw2.slc.tops_par
- 20260803.vv.iw3.slc
- S1C_IW_SLC\_\_1SDV_20260803T091319_20260803T091346_008829_011833_BE60.burst_number_table
- 20260722.vv.iw3.slc
- 20260803.vv.iw3.slc.par
- S1C_IW_SLC\_\_1SDV_20260803T091319_20260803T091346_008829_011833_BE60.zip
- 20260722.vv.iw3.slc.par
- 20260803.vv.iw3.slc.tops_par
- 20260722.vv.iw3.slc.tops_par
- 20260803.vv.SLC_tab

### 32 `ScanSAR_coreg.py`

```bash
cat 20260803.vv.SLC_tab
```

- 20260803.vv.iw1.slc
- 20260803.vv.iw1.slc.par
- 20260803.vv.iw1.slc.tops_par
- 20260803.vv.iw2.slc
- 20260803.vv.iw2.slc.par
- 20260803.vv.iw2.slc.tops_par
- 20260803.vv.iw3.slc
- 20260803.vv.iw3.slc.par
- 20260803.vv.iw3.slc.tops_par

```bash
cp 20260803.vv.SLC_tab 20260803.vv.RSLC_tab
vim 20260803.vv.RSLC_tab --slc→rslc
ScanSAR_coreg.py 2026072.vv.SLC_tab 20260722 20260803.vv.SLC_tab    20260803 20260803.vv.RSLC_tab DEM/20260722.hgt  10 2
```
