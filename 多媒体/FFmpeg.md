# FFmpeg

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