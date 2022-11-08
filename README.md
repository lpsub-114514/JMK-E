# JMK-E
Jimaku Encoder based on `VapourSynth`, which suits for Subtitle Groups encoding.<br>
基于`VapourSynth`的压制工具（适合字幕组压制用）**（即将开坑）**

## 目录
1. [前言](#前言)
2. [简介](#简介)
3. [解决思路](#解决思路)
   1. [方案1](#方案1)
   2. [方案2](#方案2)
4. [测试数据](#测试数据)
5. etc.


## 前言
* 此文所述的方案由x酱所提出，由 Lambholl 所整理<br>
* 此文偏向于科普向，因为有很多字幕组压制并不知道vs的原理（其中有很多还在用avs）<br>
* 此文假设各字幕组和读者使用`VapourSynth`而非`AviSynth`进行压制<br>
* **在阅读此文前，请先阅读 VCB-Studio 的[科普教程3](https://vcb-s.com/archives/2726)和[科普教程6](https://vcb-s.com/archives/4738)**

## 简介
&nbsp;&nbsp;字幕组，特别是中文字幕组的压制，经常需要同时进行简繁的压制；除此之外，有部分字幕组需要同时压制`1080p`和`720p`的版本<br>
&nbsp;&nbsp;但是我们发现，这么做浪费了部分压制的算力。为什么这么说呢？我们先来看一个示例的压制脚本代码：
```python
import vapoursynth as vs
import mvsfunc as mvf
import adptvgrnMod as adp

# 字幕组新番 web源 x264 通用脚本
# 基于 lolihouse 一周年礼包修改

core = vs.core
#OKE:MEMORY
core.max_cache_size = 8000
#OKE:INPUTFILE
source = r"E:\Animations\4个人各自有着自己的秘密\[SubsPlease] 4-nin wa Sorezore Uso wo Tsuku - 02 (1080p) [A5D310BC].mkv" # 片源
ass = source[:-4]+'.sc.ass'
src8 = core.lsmas.LWLibavSource(source)
src16 = mvf.Depth(src8, depth=16)
#Denoise
down444 = core.fmtc.resample(src16,960,540, sx=[-0.5,0,0], css="444", planes=[3,2,2], cplace="MPEG2")
nr16y = core.knlm.KNLMeansCL(src16, d=2, a=2, s=3,  h=0.8, wmode=2, device_type="GPU")
nr16uv = core.knlm.KNLMeansCL(down444, d=2, a=1, s=3,  h=0.4, wmode=2, device_type="GPU")
nr16 = core.std.ShufflePlanes([nr16y,nr16uv], [0,1,2], vs.YUV)
#Deband
nr8    = mvf.Depth(nr16, depth=8)
luma   = core.std.ShufflePlanes(nr8, 0, vs.YUV).resize.Bilinear(format=vs.YUV420P8)
nrmasks = core.tcanny.TCanny(nr8,sigma=0.8,op=2,gmmax=255,mode=1,planes=[0,1,2]).std.Expr(["x 7 < 0 65535 ?",""],vs.YUV420P16)
nrmaskb = core.tcanny.TCanny(nr8,sigma=1.3,t_h=6.5,op=2,planes=0)
nrmaskg = core.tcanny.TCanny(nr8,sigma=1.1,t_h=5.0,op=2,planes=0)
nrmask  = core.std.Expr([nrmaskg,nrmaskb,nrmasks, nr8],["a 20 < 65535 a 48 < x 256 * a 96 < y 256 * z ? ? ?",""],vs.YUV420P16)
nrmask  = core.std.Maximum(nrmask,0).std.Maximum(0).std.Minimum(0)
nrmask  = core.rgvs.RemoveGrain(nrmask,[20,0])
debd  = core.f3kdb.Deband(nr16,12,24,16,16,0,0,output_depth=16)
debd  = core.f3kdb.Deband(debd,20,56,32,32,0,0,output_depth=16)
debd  = mvf.LimitFilter(debd,nr16,thr=0.6,thrc=0.5,elast=2.0)
debd  = core.std.MaskedMerge(debd,nr16,nrmask,first_plane=True)
#Addnoise
adn = adp.adptvgrnMod(debd, strength=0.50, size=1.5, sharp=30, static=False, luma_scaling=12, grain_chroma=False)
out16 = core.std.MaskedMerge(adn,debd,nrmask,first_plane=True)
#OKE:DEBUG
debug = 0
if debug:
    src1 = mvf.ToRGB(src16,depth=8)
    src2 = mvf.ToRGB(out16,depth=8)
    compare = core.butteraugli.butteraugli(src1, src2)
    res = core.std.Interleave([src1,src2,compare])
else: 
    res = core.sub.TextFile(out16,ass)
    res = mvf.Depth(res,8)
res.set_output()
src8.set_output(1)
```
&nbsp;&nbsp;`VapourSynth`基于`Python`，我们可以看到，在这个脚本中，首先是输入片源，，然后将其转换为`16bit`，并使用各种导入的第三方库（称之为`滤镜`）对画面进行处理（如降噪、去色带、抗锯齿、加噪等，称之为`画面预处理`），其中处理的结果通过`Python`的变量向下传参，最后转换回`8bit`，输出画面；<br>
&nbsp;&nbsp;输出的画面的格式为`Y4M`，`VsPipe`可以执行此脚本，并将执行所得画面输出给`x264`, `x265`, `ffmpeg`等编码器，编码成`H264`, `H265`等格式的视频，最终连同音轨封装进`MP4`, `MKV`等格式的文件中，就得到我们平时播放的视频了。

&nbsp;&nbsp;**例如：**<br>
`vspipe -c y4m 114514.vpy - | ffmpeg -i - -c:v libx264 -preset veryslow -profile:v high -crf 19 -x264opts "deblock=1,-1:keyint=480:min-keyint=1:ref=9:vbv-bufsize=35000:vbv-maxrate=34000:chroma-qp-offset=1:qcomp=0.65:rc-lookahead=80:aq-mode=3:aq-strength=0.90:merange=24:psy-rd=0.60,0.20:no-fast-pskip:subme=10" 114514.mkv`

`vspipe -c y4m 114514.vpy - | x264 --demuxer y4m --level 5.0 --preset veryslow --profile high --crf 19.0 --deblock -1:-1 --keyint 480 --min-keyint 1 --ref 9 --vbv-bufsize 35000 --vbv-maxrate 34000 --chroma-qp-offset 1 --qcomp 0.65 --rc-lookahead 80 --aq-mode 3 --aq-strength 0.90 --merange 24 --fgo 1 --psy-rd 0.60:0.20 --no-fast-pskip --colorprim bt709 --transfer bt709 --colormatrix bt709 --subme 10 -o "114514.h264" -`

&nbsp;&nbsp;即可得到**不包含音轨**的视频文件
  
&nbsp;&nbsp;同时，显而易见的一件事情是，将字幕内嵌进视频这一步是写在压制脚本中的：<br>
&nbsp;&nbsp;&nbsp;&nbsp;`res = core.sub.TextFile(out16,ass)` (使用`libass`的情况)；<br>
&nbsp;&nbsp;&nbsp;&nbsp;`res = core.vsfm.TextSubMod(out16,ass)` (使用`VsFilterMod`的情况)


&nbsp;&nbsp;这时候，要压制简繁两个版本，字幕组通常会选择跑两遍这个脚本；<br>
&nbsp;&nbsp;但是显而易见的事情是，这么做会把相同的画面预处理执行两遍，而画面预处理这一步又是**十分耗费算力**的（特别是我们平时使用的压制脚本中包含了`rescale`和`bm3d`等）<br>
&nbsp;&nbsp;那么假设能把简繁两个版本使用同一个脚本输出，那么就只需要做一遍相同的画面预处理，将此结果分别内嵌字幕输出，就可以省去不少算力。<br>
> 假设压制简繁两个版本，就可以省去一次画面预处理；<br>
> 假设同时再压制一个 720p 的版本，就可以省去三次画面预处理；<br>
> 假设同时再压制一个 HEVC 内封版本……<br><br>
> 同一个片源，压制的版本越多，就越能省去算力，因此适合字幕组的压制

*那么，要怎么做到这一点呢？*

### 解决思路

#### 方案1
&nbsp;&nbsp;经过x酱的思考，发现确实有办法：

&nbsp;&nbsp;&nbsp;&nbsp;可以使用`core.std.StackVertical`将多个画面拼接在一个画面中输出视频，然后使用`ffmepg`将此视频裁剪开来，分别编码成不同的视频；<br>
&nbsp;&nbsp;&nbsp;&nbsp;按照此思路，也可以将`x265 10bit`的内封版本同时压制

&nbsp;&nbsp;**示例代码如下：**
```python
res=out16.fmtc.bitdepth(bits=10,dmode=8)
out16_sc=core.sub.TextFile(out16, source[:-4]+'.sc.ass')
amp1_sc=core.fmtc.bitdepth(out16_sc,bits=8,dmode=8,ampo=1.5)
amp2_sc=core.fmtc.bitdepth(out16_sc,bits=8,dmode=8,ampo=2)
dmask_sc=core.std.Expr(out16_sc.std.ShufflePlanes(0,vs.GRAY).fmtc.bitdepth(bits=8),'x 100 > 0 255 ?')
res_sc=core.std.MaskedMerge(amp1_sc,amp2_sc,dmask_sc)
out16_tc=core.sub.TextFile(out16, source[:-4]+'.tc.ass')
amp1_tc=core.fmtc.bitdepth(out16_tc,bits=8,dmode=8,ampo=1.5)
amp2_tc=core.fmtc.bitdepth(out16_tc,bits=8,dmode=8,ampo=2)
dmask_tc=core.std.Expr(out16_tc.std.ShufflePlanes(0,vs.GRAY).fmtc.bitdepth(bits=8),'x 100 > 0 255 ?')
res_tc=core.std.MaskedMerge(amp1_tc,amp2_tc,dmask_tc)
res_sc,res_tc=[i.fmtc.bitdepth(bits=10) for i in (res_sc,res_tc)]
res=core.std.StackVertical([res,res_sc,res_tc])
res.set_output()
src8.set_output(1)
```
> 为什么这么做呢？因为压制 x265 通常输出 10bit，因此选择输出 10bit 的画面，在压制时再输出 x264 8bit；<br>
> 同时，压制 x264 时选择 8bit，因此比起 x265 来，通常会使用噪点和 dither 来防止色带的产生。

&nbsp;&nbsp;**示例编码参数如下：**<br>
&nbsp;&nbsp;`vspipe -c y4m t.vpy - | ffmpeg -i - -filter_complex [0:v]crop=1920:1080:0:0[v1];[0:v]crop=1920:1080:0:1080,zscale,format=yuv420p[v2];[0:v]crop=1920:1080:0:2160,zscale,format=yuv420p[v3] -map [v1] -c:v libx265 a.mkv -map [v2] -c:v libx264 b.mkv -map [v3] -c:v libx264 c.mkv` (为了尽量简洁，省去了高级编码设置)

#### 方案2
&nbsp;&nbsp;在此方案被提出之后，Jan 大佬又给出了一种[新的解决思路](https://www.skyey2.com/forum.php?mod=viewthread&tid=38690)，其好处是可以调用`x264`和`x265`进行压制，因此可以用上各个自定义版本的编码器（比如说 AmusementClub 编译的 [x265](https://github.com/AmusementClub/x265/releases)）

&nbsp;&nbsp;此思路直接在`vpy`脚本内调用编码器进行编码，目前尚未被我们测试，但是我们相信此方案一定可行；

### 测试数据
&nbsp;&nbsp;那么，这么做真的能省时间吗？能省多少时间？<br>
&nbsp;&nbsp;首先，从理论上来说，压制脚本中的画面预处理越多，这么做能节省的时间就越多。例如，使用了`rescale`和`bm3d`的脚本，使用此方案能节省的时间一定比上面给出的示例脚本能节省的更多；<br>
（当然大概也就我们组会在新番上搞`rescale`和`bm3d`了吧）

**因此，我们使用了较为复杂的脚本进行测试：**<br>
&nbsp;&nbsp;在使用了`rescale`, `SMDegrain`, `TAAmbk`等处理的`BDRip`脚本上：<br>
<br>输出1440帧

| 方案               | 压制速率 | 耗时 |
| ----------------- | ------ | ---- | 
| 上文所述方案1      | 4.8fps | 300s |
| 分开输出 (同时运行) | 1.9fps | 757s |
| 分开输出 (依次运行) | 5.7fps | 757s |

&nbsp;&nbsp;可见省去了一大半的时间（如果是上面的示例脚本，能省去的时间一定比这个少了）<br>

---

考虑到新兴的压制工具`VapourSynth`到现在为止较为成熟的`GUI`工具只有 [OKEGui](https://github.com/vcb-s/OKEGui) 和 [staxrip](https://github.com/staxrip/staxrip)，而其使用门槛对一般人来说也有点高，因此我们开始了这款压制工具的制作；也藉此希望，因为需要压制多个版本而无奈选择裸压的一些字幕组后期，能因为这个压制方案的出现，而选择使用非裸压脚本进行压制。
