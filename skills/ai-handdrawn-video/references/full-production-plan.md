# 全流程计划与数据

## 示例约定

连贯示例：源片 25 fps，目标 1280×720、30 fps，总长 6 秒、180 帧，三个镜头各 2 秒。
数字仅作技术示意，不能代替新项目的真实探测或审核结果。
JSON 中 `null` 表示尚未计算，正式生产前必须补实；示例状态不是实际审批。
从零制作时用获准脚本/音轨建立时间结构，没有源视频就不伪造 source PTS。

## P0：素材、能力和技术测试

1. 指定本轮根目录与可写范围，检查现有文件，不覆盖原始参考。
2. 列出主时间参考、辅助画风参考、角色身份图、音频、文字和第三方资产，逐项记录用途。
3. 检查生成/编辑/识别工具是否真的可调用，报告可用能力而非假定模型版本。
4. 记录 Python、FFmpeg/ffprobe 和实际需要的库；先用已有工具，不为一次分析安装整套平台。
5. 检查磁盘与预计中间帧体积，包含非 ASCII/空格的路径要实际读写导出验证。
6. 输出一小段标注“技术测试”的视频，验证读取、alpha 合成、编码和全解码；不冒称动态美术样片。
7. 若生成图小于目标，优先生成紧凑透明角色层，检查主体像素，再缩小贴回背景；低清放大不算原生高清。

目录按需建立，输入保持已有名字，不强迫移动或重命名：

```text
inputs/                     原始视频、图、音频；只读保留
analysis/                   manifest、meta、shots、timeline、beats、text cues
analysis/reference-frames/  源帧、联系表与证据索引
assets/characters/          identity、母版、outfits、poses
assets/backgrounds/         背景与clean plates
assets/shots/               每镜头图层、姿态映射与生成记录
reviews/                   样片、镜头、评论与验收记录
output/previews/            技术测试、动态样片、全片预览
output/shot-renders/        冻结的局部输出
output/final.mp4            本次约定成片
```

素材清单 `analysis/input-manifest.json` 示例：

```json
{
  "schemaVersion": 1,
  "assets": [
    {"id": "source-main", "path": "inputs/reference.mp4", "role": "timing-layout", "version": "v1", "contentHash": null, "rights": "user-provided-scope-recorded"},
    {"id": "identity-A", "path": "inputs/character-A.png", "role": "character-identity", "version": "v1", "contentHash": null, "rights": "user-provided-scope-recorded"},
    {"id": "identity-B", "path": "inputs/character-B.png", "role": "character-identity", "version": "v1", "contentHash": null, "rights": "user-provided-scope-recorded"}
  ],
  "status": "example-not-production"
}
```

实际补全文件大小、真实哈希、可读性/完整性与授权范围。
“用户提供”不自动证明所有用途获准；区分制作、内部审阅与公开发布。
缺少必要输入时只阻塞依赖它的阶段，不拿辅助视频顶替主参考。

## P1：探测与共同原点

以下 JSON 重定向命令以 PowerShell 7 的 UTF-8 输出为前提；Windows PowerShell 5 使用 `| Set-Content -Encoding utf8` 替代 `>`，读取时允许 UTF-8 BOM。路径按实际输入修改。

```powershell
New-Item -ItemType Directory -Path "analysis/reference-frames" -Force | Out-Null
ffprobe -v error -show_format -show_streams -of json "inputs/reference.mp4" > "analysis/probe.json"
ffprobe -v error -select_streams v:0 -show_frames -show_entries frame=best_effort_timestamp,pts,pkt_duration -of json "inputs/reference.mp4" > "analysis/source-frames.json"
```

保存视频/音频各自的索引、起点、时长、time_base、帧率分数、尺寸、帧数、像素格式和色彩字段。
同时保存容器时长；`r_frame_rate` 与 `avg_frame_rate` 不能代替展示帧 PTS。
包 PTS 的毫秒量化抖动或帧率字段略有不同，不足以单独断言 VFR。
结合展示顺序 PTS、重复/缺失时间戳与间隔分布判断；解码顺序与展示顺序不应混用。
保留 PTS 整数与 time_base 分数，秒数只作为派生展示值。

```json
{
  "video": {"streamIndex": 0, "timeBase": {"num": 1, "den": 90000}, "startPts": 900000, "endPtsExclusive": 1440000, "rFrameRate": {"num": 25, "den": 1}, "avgFrameRate": {"num": 25, "den": 1}, "displayFrameCount": 150, "width": 1280, "height": 720},
  "projectOrigin": {"num": 10, "den": 1},
  "audio": {"streamIndex": 1, "sampleRate": 48000, "channels": 2, "projectOffsetSeconds": -0.04, "audibleEndProjectSeconds": 5.88},
  "effectiveVideoSeconds": 6,
  "containerDurationSeconds": 6.04,
  "status": "example-not-measured"
}
```

项目原点为视频 PTS 900000 对应的绝对 10 秒；音频可能更早开始。
根据流起点计算偏移，不把各流各自归零后错误对齐。
VFR 转 CFR 保留输出帧→源展示帧/PTS 映射；本指南默认取覆盖目标时刻的源显示区间 `[pts_i, pts_next)`。
如采用最近帧策略，明确距离和等距取前/后规则；不得无记录地混用策略。
末帧显示区间终点取已确认的有效视频终点。

## 候选切点与源证据

硬切检测仅产生候选；检查候选首中末和边界附近原帧。
分类为硬切、叠化、闪白、动作变化、文字变化或不确定；候选数量不等于确切镜头数。
长镜头的动作段可保留为姿态节点；缺失内部硬切按证据补入。
画面相似不能自动合并有节奏意义的切点。

`analysis/shots.json` 只保存源证据：

```json
{
  "schemaVersion": 1,
  "sourceAssetId": "source-main",
  "shots": [
    {"shotId": "s010", "sourceStartFrame": 0, "sourceEndFrameExclusive": 50, "sourceStartPts": 900000, "sourceEndPtsExclusive": 1080000, "classification": "confirmed-cut", "reviewStatus": "example-approved"},
    {"shotId": "s020", "sourceStartFrame": 50, "sourceEndFrameExclusive": 100, "sourceStartPts": 1080000, "sourceEndPtsExclusive": 1260000, "classification": "confirmed-cut", "reviewStatus": "example-approved"},
    {"shotId": "s030", "sourceStartFrame": 100, "sourceEndFrameExclusive": 150, "sourceStartPts": 1260000, "sourceEndPtsExclusive": 1440000, "classification": "confirmed-cut", "reviewStatus": "example-approved"}
  ]
}
```

实际补充检测来源/分数、分类依据、代表帧路径与未解决项。
末尾排他 PTS 来自有效内容终点，不要求存在下一张实际源帧。
抽指定源展示帧可从开头解码后 `select`，不依赖近似 seek：

```powershell
ffmpeg -v error -i "inputs/reference.mp4" -vf "select=eq(n\,50)" -frames:v 1 -fps_mode vfr "analysis/reference-frames/source-0050.png"
```

`n` 是解码输出的展示索引，从 0 开始；先确认所选视频流对应关系。
输入侧 `-ss` 适合快速预览，但可能关键帧近似且重置局部计数，不能不校验就当精确源帧。

## 绝对边界到输出时间轴

统一左闭右开 `[startFrame, endFrameExclusive)`，首帧编号 0。
用有理数计算 `t = sourcePts × timeBase − projectOrigin`，再按已记录策略取整 `t × outputFps`。
共享源边界只映射一次；每个绝对边界独立映射，再相减得镜头长度，避免累加逐段舍入时长。
记录边界误差；零帧镜头、缺口、越界和未声明重叠阻止锁定，不静默删镜头。
示例绝对边界 10、12、14、16 秒减 10 秒原点，输出为 0、60、120、180。

`analysis/timeline.json` 是唯一输出边界来源：

```json
{
  "schemaVersion": 1,
  "output": {"width": 1280, "height": 720, "fps": {"num": 30, "den": 1}, "durationFrames": 180},
  "rounding": "nearest-half-up",
  "shots": [
    {"shotId": "s010", "startFrame": 0, "endFrameExclusive": 60, "characterIds": ["A"], "reviewStatus": "example-approved"},
    {"shotId": "s020", "startFrame": 60, "endFrameExclusive": 120, "characterIds": ["A", "B"], "reviewStatus": "example-approved"},
    {"shotId": "s030", "startFrame": 120, "endFrameExclusive": 180, "characterIds": ["B"], "reviewStatus": "example-approved"}
  ],
  "transitions": [],
  "status": "example-not-production"
}
```

显示秒数、分镜表和渲染代码由它派生，不手工维护另一套输出边界。
非整数输出 fps 保留分数，如 `30000/1001`，不存近似小数再反算。

## 音频尾部与节拍

提取选定连续音轨供波形与试听，不按镜头拼接、不改 BPM 凑时长：

```powershell
ffmpeg -v error -i "inputs/reference.mp4" -map 0:a:0 -vn -c:a pcm_s16le "analysis/audio.wav"
```

WAV 不携带原项目偏移，需在元数据保留采样率、声道、流偏移和裁切区间。
波形峰/onset 是候选；拍点、重音、段落与动作触发仍需试听复核。
尾部结合样本峰值/RMS与试听，区分有效尾音、静音、编码填充和容器差异。
有效声音超出画面时提出保留尾音/补画面/淡出方案，按授权选择，不自动裁掉。
无声交付仍可保留节奏表；替换音乐时重新复核受影响卡点。

```json
{
  "beats": [
    {"id": "beat-release", "projectTime": {"num": 12, "den": 5}, "outputFrame": 72, "kind": "accent", "reviewStatus": "example-reviewed"},
    {"id": "beat-catch", "projectTime": {"num": 17, "den": 5}, "outputFrame": 102, "kind": "accent", "reviewStatus": "example-reviewed"}
  ]
}
```

## 文字 cue 与分镜

歌词、字幕、设计文字分别登记内容、语言、位置、字体/字号、方向、对齐、层级和显隐/移动动画。
OCR 只是候选，逐项核字词、顺序、精确出现/消失帧；未确定标 approximate/uncertain。
候选镜头范围不能冒充逐字字幕时码；文字变化也不自动构成切镜。
获准保留源文字时复用可靠源像素/图层；需重建时用校对 cue 独立合成。
不要把带有原人物的整张源图当文字层贴回。

```json
{
  "cues": [
    {"cueId": "t010", "kind": "subtitle", "text": "示例台词", "startFrame": 12, "endFrameExclusive": 54, "layout": {"x": 640, "y": 650, "fontSize": 32, "align": "center", "vertical": false}, "layer": "foreground-text", "animation": "hold", "transcriptionStatus": "example-reviewed", "timingStatus": "example-reviewed"},
    {"cueId": "t020", "kind": "design", "text": "传递", "startFrame": 72, "endFrameExclusive": 114, "layout": {"x": 100, "y": 120, "fontSize": 56, "align": "left", "vertical": false}, "layer": "background-text", "animation": "slide-in", "transcriptionStatus": "example-reviewed", "timingStatus": "example-reviewed"}
  ]
}
```

分镜表每镜头一行：区间、源证据、构图、相机、角色/服装、动作、道具、cue、节拍、资产缺口。
P1 通过条件：首尾/原点清楚、误差可解释、覆盖连续、文字/节拍可查，未确认内容没有标为批准。
