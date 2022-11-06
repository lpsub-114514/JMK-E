# JMK-E
Jimaku Encoder based on `VapourSynth`, which suits for Subtitle Groups encoding.<br>
基于·VapourSynth`的压制工具（适合字幕组压制用）

## 简介
&nbsp;&nbsp;字幕组，特别是中文字幕组的压制，经常需要同时进行简繁的压制；除此之外，有部分字幕组需要同时压制`1080p`和`720p`的版本<br>
&nbsp;&nbsp;但是我们发现，这么做浪费了部分压制的算力。为什么这么说呢？我们先来看一个示例的压制脚本代码：
```python
import vapoursynth as vs
import mvsfunc as mvf
import adptvgrnMod as adp

# 字幕组新番 web源 x264 通用脚本

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
&nbsp;&nbsp;`VapourSynth`基于`Python`，我们可以看到，在这个脚本中，首先是输入片源，然后使用各种导入的第三方库（称之为`滤镜`）对画面进行处理（如降噪、去色带、抗锯齿、加噪等，称之为`画面预处理`），其中处理的结果通过`Python`的变量向下传参，最后输出画面；<br>
&nbsp;&nbsp;输出的画面的格式为`Y4M`，`VsPipe`可以执行此脚本，并将执行所得画面输出给`x264`, `x265`, `ffmpeg`等编码器，编码成`H264`, `H265`等格式的视频，最终连同音轨封装进`MP4`, `MKV`等格式的文件中，就得到我们平时播放的视频了。
  
&nbsp;&nbsp;同时，显而易见的一件事情是，将字幕内嵌进视频这一步是写在压制脚本中的：<br>
&nbsp;&nbsp;&nbsp;&nbsp;`res = core.sub.TextFile(out16,ass)` (使用`libass`的情况)；<br>
&nbsp;&nbsp;&nbsp;&nbsp;`res = core.vsfm.TextSubMod(src8,ass)` (使用`VsFilterMod`的情况)<br><br>
&nbsp;&nbsp;这时候，要压制简繁两个版本，字幕组通常会选择跑两遍这个脚本；但是显而易见的事情是，这么做会把相同的画面预处理执行两遍，而画面预处理这一步又是十分耗费算力的（特别是我们平时使用的压制脚本中包含了`rescale`和`bm3d`等），假设能把简繁两个版本使用同一个脚本输出，那么就只需要做一遍相同的画面预处理，分别内嵌字幕输出就行了。
