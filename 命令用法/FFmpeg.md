# FFmpeg

* [官方文档](https://ffmpeg.org/ffmpeg.html)
* [中文参考](https://longqi.cf/tools/2015/02/13/ffmpegcn/)

## 有用的脚本

* 处理metadata信息

```sh
# 增加标题信息
ffmpeg -i a.mp4 -metadata title="my title" b.mp4

# 去除文件里的标题信息（例如一些广告信息）这种情况只去除global信息就可以
# -meta_data:g 指global -1表示不要了
ffmpeg -i a.mp4 -metadata:g -1 -c copy b.mp4
# -metadata title= 表示删除title
ffmpeg -i a.mp4 -metadata title= -c copy b.mp4

# 复制metadata信息
# -meta_data 0:s:0 0:g 表示输入第1个文件的第1个流信息输出到输出的第一个文件的全局信息
# 把音频信息复制到全局信息
ffmpeg -i a.mp4 -metadata:s:a 0:g b.mp4

# 把第一个字幕和第二个字幕轨道交换，并把第二个字幕设置为默认
# -disposition:s:0 default 设置s:0为默认
# -disposition:s:1 0 删除s:1的默认属性
ffmpeg -i a.mkv -map 0:s:1 -map 0:s:0 -disposition:s:0 default -disposition:s:1 0 -c:s copy b.mkv
```

* 转码保持多路音频

```sh
# 0:a 第一个输入文件的全部音频
# 1:a:1 第二个输入文件的第2路音频
ffmpeg -i a.mkv -i b.rmvb -map 0:v -map 0:a -map 1:a -vcodec copy -acodec aac b.mkv

# 去除mkv中多余信息，重新组织音视频，如把粤语音轨+中文字幕设置成默认
ffmpeg -i in.mkv -metadata:g "title=" \
 -metadata:s:a:1 language=chi -metadata:s:a:1 title="粤语" -disposition:a:1 default -map 0:a:1 \
 -metadata:s:a:0 language=chi -metadata:s:a:0 title="国语" -disposition:a:0 0 -map 0:a:0 \
 -metadata:s:s:1 language=chi -metadata:s:s:1 title="中文" -disposition:s:1 default -map 0:s:1 \
 -metadata:s:s:0 language=eng -metadata:s:s:0 title="English" -disposition:s:0 0 -map 0:s:0 \
 -metadata:s:v:0 title= -map 0:v \
-c copy out.mkv

# 默认音轨对调
ffmpeg -i in.mkv -map 0:v -map 0:a:1 -map 0:a:0 -metadata:s:a:0 language=chi -metadata:s:a:0 title="粤语" -disposition:a:0 default -metadata:s:a:1 language=chi -metadata:s:a:1 title="国语" -disposition:a:1 0 -c copy out.mkv

```

* 音画同步

```sh
# itsoffset 视频推迟的时间，如-5是视频加快5秒
# itsoffset 和 ss 区别是，例如60秒视频，ss 5 后截断为55秒了，而itsoffset是静止5秒后播放，总时长不变
# -avoid_negative_ts 1 可以避免变成负数的帧
ffmpeg -itsoffset 00:00:00.900 -i src.mp4 -i src.mp4 -map 0:v -map 1:a -vcodec copy -acodec copy out.mp4

#如果是播放速度不一致，播放一段之后变慢了，可以把音频进行重采样来调整
ffmpeg -i src.mp4 -vn -filter:a "[0:a]atempo=1.0004[a]" out.mp4

#视音频一起变慢0.8，其中PTS和atempo需要的值刚好去反
ffmpeg -i in.mp4 -filter_complex "[0:v]setpts=1.25*PTS[v];[0:a]atempo=0.8[a]" -map "[v]" -map "[a]" out.mp4
```

* 音量调整

```sh
ffmpeg -i in.mp4 -vcodec copy -af "volume=25dB" -f flv - | ffplay -
ffmpeg -i in.mp4 -af "volumedetect" -f null /dev/null
```

* 快速合并多个视频

```sh
ffmpeg -f concat -safe 0 -i <(for f in ./*.mp4; do echo "file '$PWD/$f'"; done) -c copy output.mp4
```

* 强大视频字幕编辑

https://github.com/mli/autocut

* 音频文件添加黑白视频（方便显示字幕）

```sh
ffmpeg -f lavfi -i color=Black:640x480 -i audio.m4a -b:v 60k -r 1 -shortest -map 0:v -map 1:a out.mp4
```

* 视频对比

```sh
ffmpeg -i left.mp4 -i right.mp4 -filter_complex "[1:v][0:v]scale2ref=w=iw:h=ih[rv][lv];[lv]pad='2*iw:ih'[lv2];[lv2][rv]overlay=x=w:y=0" -codec:v rawvideo -f avi - | ffplay -f avi -

#vmaf评分
ffmpeg -i $2 -i $1 -filter_complex "[0:v]scale=$3:flags=bicubic[main];[main][1:v]libvmaf" -f null - 2>&1 | grep VMAF
```

* Mac 屏幕录制

```sh
#Mac下录制屏幕
#列出资源列表，video:audio, e.g. -i 0:1
ffmpeg -f avfoundation -list_devices true -i "" 2>&1 | grep AVFoundation \
    | sed -e 's/\[[^]]\{10,\}\] //g' \
    | sed -e 's/\b[a-z]/\u&/g' \
    | sed -e 's/AVFoundation //g' | sed 's/^\[/ &/' \
    | grep --color=auto " \|\[[0-9]\]\|Capture\|Aggregate"

#源参数，在-i前指定：
# 源大小 -video_size 1080x720
# 录制鼠标位置 -capture_cursor 1
# 录制鼠标点击 -capture_mouse_clicks 1
#注意帧数会比较大限制-r输出，另外机器性能差可能会音画不同步，注意ffmpeg编码速度得保持1x
#录制屏幕某个区域 -filter_complex "crop=1080:450:0:26" -> w:h:x:y
#屏幕缩放 -filter_complex "scale=-2:720"
#建议加 -t 2:00:00 限制最长时长
#录制输出声音建议使用soundflower(2ch)
#声音捕捉容易有爆音，有几个参数有时会有用，包括-vsync cfr，-vsync 2，-ac 1，-filter_complex="xxx;aresameple"等
ffmpeg -f avfoundation -framerate 30 -pix_fmt nv12 -i 1:0 \
  -r 25 -filter_complex "crop=1080:480:0:162" -c:v libx264 -preset slow -crf 22 \
  -c:a aac -b:a 64k -ac 2 -t 1:38:20 out.mp4
```

* 保留透明层

```sh
ffmpeg -i src.gif -filter_complex "[0:v]scale=320:-2,split[a][b];[a]palettegen=reserve_transparent=on:transparency_color=ffffff[p];[b][p]paletteuse" out.gif
```

* 处理表面是png/jpg，实际是ts文件

```sh
#hexdump -v -s 2 -x -n 100 a.png
ls *.png | xargs -I {} dd if={} of={}.ts bs=4 skip=53
```

* 截图

```sh

# 关键帧截图
ffmpeg -i src.mp4 -vf "select=eq(pict_type\,I)" -frames:v 1 -pix_fmt yuvj422p  -vsync vfr -qscale:v 2 -f image2 /tmp/mp4.jpg -y

# 变化大于50%帧截图
ffmpeg -i src.mp4 -vf "select=gt(scene\,0.5)" -vsync vfr photo%03d.png

```
