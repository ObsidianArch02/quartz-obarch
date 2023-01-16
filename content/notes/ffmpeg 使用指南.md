---
title: ffmpeg 使用指南
---

# 小 tips

- [将字幕压制到视频中 | Verne in GitHub](https://einverne.github.io/post/2022/10/embedded-subtitle-into-video.html)
- [Golang 调用 FFMpeg – 小金鱼儿](https://haoyu.love/blog1394.html)

# 进行屏幕录制

## 配置视频/音频设备

FFmpeg 的录制命令 gdigrab 不支持音频录制，也不支持直接调用摄像头，此时需使用开源的  [screen-capture-recorder-to-video-windows-free](https://github.com/rdp/screen-capture-recorder-to-video-windows-free/releases)  增强 FFmpeg 的录制功能，其最新版本为 0.12.12。

通过命令  `ffmpeg -list_devices true -f dshow -i dummy`  查看支持的 Windows DirectShow 输入设备，采集视频和音频设备，包含设备名称，设备类型等信息 1。比如我这里得到了视频设备「V380 FHD Camera」和音频设备「Analogue 1/2 (Audient iD4)」，之后会用到。

## 录制屏幕

从坐标 0:0 开始圈定出一个 2560x1440 的屏幕范围，然后以每 50 秒截图 1 帧，输出为 mp4 格式的视频，录制命令为： `ffmpeg -f gdigrab -r 20/1001 -draw_mouse 1 -offset_x 0 -offset_y 0 -video_size 2560x1440 -i desktop -s 1280x720 output.mp4`

以下是录制命令的说明：

- `-f gdigrab`  使用 FFmpeg 内置屏幕录制命令  [gdigrab](https://ffmpeg.org/ffmpeg-all.html#gdigrab)，录制对象可为全屏、指定范围和指定程序。
- `-r 20/1001`  帧率为 0.02，每 50 秒录制 1 帧。主流大家喜欢用  `-r 30`  录制，我这因为是每日监测视频，用了超低帧率。
- `-c:v libx264`  是用于设置视频编解码器，一般可不填使用默认配置，`-c:a`  为音频编码。2
- `-draw_mouse 1`  在 gdigrab 录制的视频中显示鼠标。
- `-offset_x 0 -offset_y 0 -video_size 2560x1440`  为起始坐标和选定录制范围。坐标可使用截图软件获取，比如我用 Snipaste，点击 F1 后进入截图界面，鼠标经过当前区域就会显示坐标。
- `-s 1280x720`  用 scale 方法，设置视频分辨率为 720p。
- `-i desktop`  为输入设备，指代显示屏。
- `out.mp4`  为输出视频的名字与格式。默认保存在命令运行文件夹，可以在此处设置输出位置，如  `D:\Backup\Libraries\Desktop\out.mp4`。

除上方命令外，FFmpeg 还有许多参数可以设置，比如  `-pix_fmt yuv420p -preset ultrafast`  提升编码速度，`-filter:v "setpts=0.1*PTS"`  减少视频抽样，但 setpts 不是视频加速，对于低帧率的视频影响很小。3

## 录制摄像头

然后，我们使用上方获取的视频设备，即可用摄像头进行录制，如： `ffmpeg -f dshow -i video="V380 FHD Camera" output.mp4`

如果录屏的同时需要录制音频，则在命令中加入之前获取的音频设备，命令变为： `ffmpeg -f dshow -i audio="Analogue 1/2 (Audient iD4)" -f dshow -i video="V380 FHD Camera" output.mp4`

## 输出视频：画中画

清楚如何用 FFmpeg 录制屏幕、摄像头和音频后，我需要将他们放置于同一画面中，将摄像头画面放在录制画面的右下侧，并用 overlay 方法将其置于屏幕画面的上方，遮挡对应区域。4

综合了以上三步，最终的录制命令为： `ffmpeg -f gdigrab -r 1 -draw_mouse 1 -offset_x 0 -offset_y 0 -video_size 2560x1440 -i desktop -s 1280x720 -b:v 0 -crf 32 output.mp4 -f dshow -i audio="Analogue 1/2 (Audient iD4)" -f dshow -s 640x360 -i video="V380 FHD Camera" -filter_complex "overlay=W-w-1:H-h-1" -y`

- `-b:v 0 -crf 32`  是将视频比特率设置为最小，同时使用恒定质量，CRF5  的范围可以从 0（最佳质量）到 63（最小文件大小）。
- `overlay=W-w-1:H-h-1`  这是一个坐标，指浮层放在右下角，距离边缘 1px。
- `-y`  遇到选项时，默认执行 yes 命令，比如覆盖同名的视频文件。

命令中的录制帧率较低，但不会影响同时录制的音频。之后的录屏只需在终端中运行这段命令，就会自动录制屏幕，按  `q`  即可停止录制。使用 FFmpeg 后，我的录屏再也没有莫名其妙的崩溃了。

## 常见问题

### Could not set video options

报错  `Could not set video options`，多是由于录制设置的帧率、分辨率超出设备范围造成的。使用命令  `ffmpeg -f dshow -list_options true -i video="V380 FHD Camera" -loglevel debug`  检查设备的输出属性，调整录制属性。

### real-time buffer

报错  `real-time buffer [xxxxxx] [video input] too full or near too full (181% of size: 3041280 [rtbufsize parameter])! frame dropped!`，解决方案参考  [issue 136](https://github.com/rdp/screen-capture-recorder-to-video-windows-free/issues/136)。我未修复成功，不过也未影响视频录制。

### 摄像头分辨率错误

如果摄像头画面出现裁切，分辨率与预想不同，则检查摄像头录制属性和摄像头应用输出分辨率。例如部分版本的 SplitCam Video Driver 对外场景尺寸被固定为 4:3，导致输出画面被裁剪，只能更换其他视频输入源。

### 录制画面偏移

录制画面比例异常或画幅偏移，这是 Windows 的屏幕缩放造成的，勾选 ffmpeg.exe 的属性「高 DPI 缩放替代」即可解决。

# 压制、转码视频

- [ffmpeg 转码视频真的好用！（ffmpeg 的简单使用方法） - 简书](https://www.jianshu.com/p/4f399b9dfb43)
- [使用 FFMPEG 进行视频转码 - 知乎](https://zhuanlan.zhihu.com/p/162352065)
- [H.265 编码码率参考 - crocuta - 博客园](https://www.cnblogs.com/crocuta/p/13199341.html)
- [视频码率设定参考值 - 简书](https://www.jianshu.com/p/be38f54dafcb)
- [[FFmpeg] ffmpeg 常用命令 - 晏过留痕 - 博客园](https://www.cnblogs.com/frost-yen/p/5848781.html)

# 解决国标 AVS2 及 AVS3 编码问题

众所周知，CCTV 采用了一种特殊的编解码标准以提供 8K 的媒体转播服务，即国标 AVS2 及 AVS3，电视转播的北京冬奥会的开幕式就有用到。

编解码这种类型的视频，需要用到：

> [Releases · xatabhk/FFmpeg-avs2-avs3](https://github.com/xatabhk/FFmpeg-avs2-avs3/releases)
>
> 或者 [FFmpeg-avs2-avs3 发行版 - Gitee.com](https://gitee.com/zhengtianbo/FFmpeg-avs2-avs3/releases)

当然也可以采用整合包的方案，比较方便：

> LAVFilters-X64-0.76.X-642bf121d.zip（改版解码器）
>
> gitee.com/zhengtianbo/LAVFilters-GB-CAVS-AVS2-AVS3-decoder/releases/0.76
>
> MPC-HC.1.9.19.x64-avs2-avs3.zip（改版播放器）
>
> gitee.com/zhengtianbo/cavs-avs2-avs3_decoder_added_to_mpc_hc/releases

然后就可以参照常规 ffmpeg 使用方法进行操作。

# 命令生成与参考文档

[FFmpeg.guide - One stop solution to all things FFmpeg](https://ffmpeg.guide/)

[ffmpeg Documentation](https://ffmpeg.org/ffmpeg.html)

# 参考文档

[如何满足小众的录屏需求？自己配置 FFmpeg 解决问题 - 少数派](https://sspai.com/post/76637)
