# 时间轴与分层合成

承接 [全流程数据](full-production-plan.md)；所有参数是示例，按实际源与简报修改。

## P4：统一帧函数

用一个输出时间轴提供 fps 分数、总帧数和镜头区间。
全局帧 `g` 定位镜头；局部帧 `l = g − shot.startFrame` 只决定镜头内部动作。
全局时间 `g × fps.den / fps.num` 与源 PTS 映射分开，不能把 local 直接当源帧。
渲染值仅依赖显式帧、稳定资产版本、镜头 ID 和固定 seed，不依赖上次渲染的状态。
固定 seed 只能保证已实现的确定性逻辑，不承诺远程生图服务逐次输出相同像素。

接口示意，需接已有工程的读取、合成与转场实现，不是独立可运行脚本：

```python
def frame_at(global_frame, timeline, assets):
    shot = lookup_shot(global_frame, timeline)
    local_frame = global_frame - shot["startFrame"]
    base = render_shot(shot, local_frame, global_frame, assets)
    return apply_declared_transition(base, global_frame, timeline, assets)
```

单镜头预览也传入原全局起点、相邻镜头与转场上下文；相同位置应与全片帧一致。
留帧用明确局部区间选择 pose，使用左闭右开边界；姿态变化和相机/文字运动可用不同频率。
只为当前镜头需要的变换写逻辑，已有帧选择器/合成器优先复用。

自包含的有理数边界自检，保存为临时 Python 文件即可运行：

```python
from fractions import Fraction

def half_up_nonnegative(value):
    assert value >= 0
    return (2 * value.numerator + value.denominator) // (2 * value.denominator)

def map_boundary(pts, time_base, origin, fps):
    return half_up_nonnegative((pts * time_base - origin) * fps)

bounds = [900000, 1080000, 1260000, 1440000]
mapped = [map_boundary(p, Fraction(1, 90000), Fraction(10), Fraction(30)) for p in bounds]
assert mapped == [0, 60, 120, 180]
lengths = [b - a for a, b in zip(mapped, mapped[1:])]
assert all(n > 0 for n in lengths) and sum(lengths) == 180
assert half_up_nonnegative(Fraction(1, 2)) == 1
```

实际实现还检查无缺口、越界、异常重叠和映射误差；负时间点由原点/裁切方案先处理，不默默截断为 0。

## 显式转场窗口

基础镜头仍覆盖 `[0,180)`，转场另记全局窗口、参与镜头与类型。
下面是可选替代方案，不是全流程示例默认硬切时间轴的一部分：

```json
{
  "transitionId": "tr010",
  "startFrame": 56,
  "endFrameExclusive": 64,
  "fromShot": "s010",
  "toShot": "s020",
  "type": "crossfade",
  "requiredHandles": {"s010PostFrames": 4, "s020PreFrames": 4}
}
```

窗口内渲染参与双方的延长帧，确认它们可用；窗口外无隐式重叠。
不要通过把下一镜头起点前移来“吃掉”转场时长，后续 60/120/180 边界保持不变。
转场前后、首末窗口和单镜头导出都验证一致；没有 handles 时选择已授权的替代转场或补素材。

## 图层与原布局保护

原片换人先保留完整源画面，再在替换区域生成 clean plate，清掉原角色可见部分。
背景/动物/道具/字幕不必跟人物一起重生；授权范围内可用原像素保留原运动与文字动画。
全新制作则直接构建背景与必要分层，仍保持相同图层关系。

```json
{
  "shotId": "s020",
  "layers": [
    {"id": "plate", "role": "clean-background", "path": "assets/shots/s020/plate.png", "z": 0},
    {"id": "rear-text", "role": "text", "cueId": "t020", "z": 10},
    {"id": "characters", "role": "character-poses", "z": 20},
    {"id": "ball", "role": "prop", "z": 30},
    {"id": "front-hand", "role": "occlusion", "z": 40}
  ],
  "occlusionNote": "按接触状态调整手与球的局部前后关系"
}
```

z 值不是全片固定真理：球可被手指局部挡住，手球又可从头前经过，必要时拆前后片。
文字颜色可能与衣服、发色、背景接近；仅按色阈值会误删人物或把背景带入前景。
按可核实的文字层/局部区域/边缘遮罩提取，查看其与人物交叉处。
沙发、双人、动物与枕头等共同构图要一起检查 clean plate，不用不透明矩形遮住旧角色。
兔耳、枕头、衣袖或听筒若有独立遮挡/运动，按需要拆层，不把每个物体过度拆解。
修补后全画面查旧手/衣残留、断线、道具缺片、矩形边、颈部接缝和窗口裁切。

## 音轨连续编排

音轨只放一次；每镜头导出不重复从音乐开头播放。
记录源音频→项目原点的偏移；例中音频早 0.04 秒，抽出的 WAV 要先裁去 0.04 秒再项目归零。
以下仅用于已经确认尾音都在 6 秒内的示例，不能套用于有超尾有效声音的素材：

```powershell
ffmpeg -v error -i "output/visual.mp4" -i "analysis/audio.wav" -filter_complex "[1:a]atrim=start=0.04,asetpts=PTS-STARTPTS,apad,atrim=end=6[a]" -map 0:v:0 -map "[a]" -c:v copy -c:a aac -movflags +faststart "output/final-with-audio.mp4"
```

音频晚开始时应显式延迟或补静音，不用改变音乐速度补偿。
无声交付编码时不映射音轨；是否静音由简报决定。

## 色彩：先知道像素是什么

分别记录原色、传递特性、YUV 矩阵、full/limited 范围及像素格式。
raw 字节无元数据：RGB 和 YUV 都需发送端与接收端约定，不能只看输出文件标签。
标签声明不是转换；把错误 RGB↔YUV 运算后的视频标 BT.709 不会还原颜色。
未知输入先核来源/对照，不把某个项目的 BT.709 假设推广到全部素材。

### 已确认 BT.709 limited YUV → RGB → YUV

前提：真实源为 BT.709 limited、SDR；示例处理链输出 full-range rgb24。
下例输出 raw RGB 供 Python 处理，尺寸必须用 probe 确认；长片应改为管道流式读取：

```powershell
ffmpeg -v error -i "inputs/reference.mp4" -map 0:v:0 -vf "scale=in_color_matrix=bt709:in_range=tv:out_range=pc,format=rgb24" -an -f rawvideo -pix_fmt rgb24 "output/source.rgb"
```

Python 生成 `output/composite.rgb` 后编码；已知输入为 full RGB，执行实际矩阵/范围转换再写匹配标签：

```powershell
ffmpeg -v error -f rawvideo -pixel_format rgb24 -video_size 1280x720 -framerate 30 -i "output/composite.rgb" -vf "scale=in_range=pc:out_range=tv:out_color_matrix=bt709,format=yuv420p" -frames:v 180 -c:v libx264 -crf 18 -preset medium -threads 2 -color_range tv -colorspace bt709 -color_primaries bt709 -color_trc bt709 -an "output/visual.mp4"
```

输入 RGB 若不是上述 SDR 色域/传递特性，需要相应色彩管理，而不是继续贴这些标签。

### 已确认的 raw YUV 保持同一矩阵/范围

前提：`output/composite.yuv` 已是 BT.709 limited 的 yuv420p；只编码，不执行 RGB 往返转换：

```powershell
ffmpeg -v error -f rawvideo -pixel_format yuv420p -video_size 1280x720 -framerate 30 -color_range tv -colorspace bt709 -color_primaries bt709 -color_trc bt709 -i "output/composite.yuv" -frames:v 180 -c:v libx264 -crf 18 -preset medium -threads 2 -pix_fmt yuv420p -color_range tv -colorspace bt709 -color_primaries bt709 -color_trc bt709 -an "output/visual-yuv.mp4"
```

raw YUV 输入侧也明确范围与矩阵，避免后续滤镜或自动转换采用默认值。
如果实际 YUV 为 full range 或另一矩阵，应先按真实输入执行转换，再声明真实输出。
对比未改背景、中性灰、肤色及高饱和区域的源/最终帧，检查整体轻微漂色，不能只查标签。

## 资源与缺失输入

逐帧流式、有限缓存、有限编码线程，先局部验证后整片编码；不要一次载入所有高清帧。
资源不足先减自己的并发/批次，不结束用户应用；取消时停止自己启动的进程并标记不完整输出。
缺图/缺姿态/版本不符应阻止对应正式导出，明确报告镜头与缺口。
预览可显式显示“缺失素材”，正式成片不能悄悄回退原角色、旧缓存或几何占位。
时间映射、留帧选择、缺图拒绝和单镜头/全片一致性属于应有小型自检的非平凡逻辑。
