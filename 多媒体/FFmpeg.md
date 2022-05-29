# FFmpeg

* [官方文档](https://ffmpeg.org/ffmpeg.html)
* [中文参考](https://longqi.cf/tools/2015/02/13/ffmpegcn/)

## 有用的脚本

* 处理metadata信息

```sh
# 增加标题信息
ffmpeg -i a.mp4 -meta_data title="my title" b.mp4

# 去除文件里的标题信息（例如一些广告信息）这种情况只去除global信息就可以
# -meta_data:g 指global -1表示不要了
ffmpeg -i a.mp4 -meta_data:g -1 -c copy b.mp4
# -metadata title= 表示删除title
ffmpeg -i a.mp4 -metadata title= -c copy b.mp4

# 复制metadata信息
# -meta_data 0:s:0 0:g 表示输入第1个文件的第1个流信息输出到输出的第一个文件的全局信息
# 把音频信息复制到全局信息
ffmpeg -i a.mp4 -meta_data:s:a 0:g b.mp4
```

* 转码保持多路音频

```sh
# 0:a 第一个输入文件的全部音频
# 1:a:1 第二个输入文件的第2路音频
ffmpeg -i a.mkv -i b.rmvb -map 0:v -map 0:a -map 1:a -vcodec copy -acodec aac b.mkv
```

* 音画同步

```sh
# itsoffset 视频推迟的时间，如-5是视频加快5秒
# itsoffset 和 ss 区别是，例如60秒视频，ss 5 后截断为55秒了，而itsoffset是静止5秒后播放，总时长不变
ffmpeg -itsoffset 00:00:00.900 -i src.mp4 -i src.mp4 -map 0:v -map 1:a -vcodec copy -acodec copy out.mp4

#如果是播放速度不一致，播放一段之后变慢了，可以把音频进行重采样来调整
ffmpeg -i src.mp4 -vn -filter:a "[0:a]atempo=1.0004[a]" out.mp4
```

* 快速合并多个视频

```sh
ffmpeg -f concat -safe 0 -i <(for f in ./*.mp4; do echo "file '$PWD/$f'"; done) -c copy output.mp4
```

* 音频文件添加黑白视频（方便显示字幕）

```sh
ffmpeg -f lavfi -i color=Black:640x480 -i audio.m4a -b:v 60k -r 1 -shortest -map 0:v -map 1:a out.mp4
```

* 视频对比

```sh
ffmpeg -i left.mp4 -i right.mp4 -filter_complex "[1:v][0:v]scale2ref=w=iw:h=ih[rv][lv];[lv]pad='2*iw:ih'[lv2];[lv2][rv]overlay=x=w:y=0" -codec:v libx264 -b:v 10m -f flv - | ffplay -
```

* Mac 屏幕录制

```sh
#需要解决屏幕视频帧数过大问题
#性能差的机器编码延迟问题
#声音播放速度异常。声音还是有一点爆音
ffmpeg -f avfoundation -r 15 -i 1:0 -vsync cfr -filter_complex "scale=-2:720;aresameple" -preset ultrafast -crf 28 out.mp4 
```
