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

# 复制metadata信息
# -meta_data 0:s:0 0:g 表示输入第1个文件的第1个流信息输出到输出的第一个文件的全局信息
# 把音频信息复制到全局信息
ffmpeg -i a.mp4 -meta_data:s:a 0:g b.mp4
```

* 转码保持多路音频

```sh
# 0:a 全部音频
# 0:a:1 保留第2路音频
ffmpeg -i a.mkv -map 0:v -map 0:a b.mkv
```

* 音画同步

```sh
# itsoffset 视频推迟的时间，如-5是视频加快5秒
# itsoffset 和 ss 区别是，例如60秒视频，ss 5 后截断为55秒了，而itsoffset是静止5秒后播放，总时长不变
ffmpeg -itsoffset 00:00:00.900 -i src.mp4 -i src.mp4 -map 0:v -map 1:a -vcodec copy -acodec copy out.mp4
```

* 快速合并多个视频

```sh
ffmpeg -f concat -safe 0 -i <(for f in ./*.mp4; do echo "file '$PWD/$f'"; done) -c copy output.mp4
```

* 音频文件添加黑白视频（方便显示字幕）

```sh
ffmpeg -f lavfi -i color=Black:640x480:d=3 -i audio.m4a -map 0:v -map 1:a out.mp4
```
