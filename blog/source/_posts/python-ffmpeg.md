---
title: ffmpeg-python的基本用法
---
# ffmpeg-python的基本用法

## ffmpeg-python 的背景资料

在文章 [ffmpeg操作集锦](https://chengzhaoxi.xyz/3237d8db.html) 中，我们有总结一些 ffmpeg 常用操作的命令行写法。

FFmpeg 的功能非常强大，但是它的命令行界面还是很复杂的，尤其涉及到处理信号图的时候。而如果使用 ffmpeg-python，代码就会变得简明易懂。该项目可以直接 pip 安装，相关资源如下：

- [API 参考](https://kkroening.github.io/ffmpeg-python/)
- [github 仓库](https://github.com/kkroening/ffmpeg-python)

更多例子可以看 github 仓库中的 [example 文件夹](https://github.com/kkroening/ffmpeg-python/tree/master/examples)。

## 安装

执行一下两条命令。

```
pip install ffmpeg-python
conda install -c conda-forge ffmpeg
```

------

# ffmpeg-python 常用 API

## (1) Stream 类

```
class Stream(upstream_node, upstream_label, node_types, upstream_selector=None)
```

Stream 对象有 audio 和 video 两部分。例如下面的代码中，out 的类型为：

```
class 'ffmpeg.nodes.OutputStream'>
import ffmpeg

stream = ffmpeg.input("video.mov")
audio = stream.audio.filter("aecho", 0.8, 0.9, 1000, 0.3)
video = stream.video.hflip()
out = ffmpeg.output(audio, video, "out.mp4")

ffmpeg.run(out)
```

## (2) ffmpeg.input

```
ffmpeg.input(filename, **kwargs)
```

对应 ffmpeg 的 `-i` 选项。

`pipe:` 作为 filename，则 ffmpeg 从 stdin 读取。

`**kwargs` 的例子：`t=20, f='mp4', acodec='pcm'`。

## (3) ffmpeg.merge_outputs

在一个 ffmpeg 命令中包含所有给定的 outputs。

```
ffmpeg.merge_outputs(*streams)
```

## (4) ffmpeg.output

```
ffmpeg.output(stream1[, stream2, stream3…], filename, **ffmpeg_args)
```

`**ffmpeg_args` 的例子：`t=20, f='mp4', acodec='pcm', vcodec='rawvideo'`。

有一些特殊处理的参数，如下：

- video_bitrate: 对应 `-b:v`, 例如 `video_bitrate=1000`。
- audio_bitrate: 对应 `-b:a`, 例如 `audio_bitrate=200`。
- format: 对应 `-f`, 例如 `format='mp4'`。

以 `pipe:` 为 filename，则写入 stdout。

## (5) ffmpeg.overwrite_output

```
ffmpeg.overwrite_output(stream)
```

对应 ffmpeg 的 `-y` 选项。输出覆盖已有文件时不询问。

## (6) ffmpeg.probe

对给定文件执行 ffprobe，并以 Json 输出。

```
ffmpeg.probe(filename, cmd='ffprobe', **kwargs)
```

## (7) ffmpeg.compile

构建命令行，以便执行 ffmpeg。

```
ffmpeg.compile(stream_spec, cmd='ffmpeg', overwrite_output=False)
```

run 函数用这个 compile 函数构建命令行参数，此外当 debug 时，直接运行 compile 也是有用的一个方法。

## (8) ffmpeg.get_args

构建命令行中传给 ffmpeg 的参数。

## (9) ffmpeg.run

对提供的 **node graph** 调用 ffmpeg。

```
ffmpeg.run(stream_spec
          ,cmd='ffmpeg'
          ,capture_stdout=False
          ,capture_stderr=False
          ,input=None
          ,quiet=False
          ,overwrite_output=False
          )
```

一些参数如下：

- `input`: 传给 stdin 的文本(与 `pipe:` 的 ffmpeg 输入一起使用)。
- `capture_stdout`: True 则抓取 stdout (与 `pipe:` 的 ffmpeg 输出一起使用).
- `capture_stderr`: True 则抓取 stderr.
- `quiet`: 设置 `capture_stdout` 和 `capture_stderr` 的简便方法。
- `**kwargs`: 传给 get_args() 的**keyword-arguments** (例如 `overwrite_output=True`)。

返回 (out, err) 元组，包括抓取的 stdout 和 stderr 数据。

## (10) ffmpeg.run_async

对提供的 **node graph** 异步调用 ffmpeg。

```
ffmpeg.run_async(stream_spec
                ,cmd='ffmpeg'
                ,pipe_stdin=False
                ,pipe_stdout=False
                ,pipe_stderr=False
                ,quiet=False
                ,overwrite_output=False
                )
```

一些参数如下：

- `pipe_in`: True 则将 pipe 连到 subprocess stdin (与 `pipe:` 的 ffmpeg 输入一起使用)。
- `pipe_stdout`: True 则将 pipe 连到 subprocess stdout (与 `pipe:` 的 ffmpeg 输出一起使用)。
- `pipe_stderr`: True 则将 pipe 连到 subprocess stderr。
- `quiet`: 设置 `capture_stdout` 和 `capture_stderr` 的简便方法。
- `**kwargs`: 传给 get_args() 的 keyword-arguments (例如 `overwrite_output=True`)。

run_async 的返回类型为 `<class 'subprocess.Popen'>`。

### run_async 的例子: 用 numpy 一帧一帧处理

下面的代码中，process1 读取 in_filename，然后传给 subprocess stdout；process2 从 stdin 读取，传给 out_filename。

numpy 处理的部分先从 process1.stdout 读取，然后用 np.frombuffer 将数据解码为 np 数据，处理之后，用 tobytes 方法变回 bytes 数据传给 process2.stdin。

```
in_filename = "video.mov"
out_filename = "outtt.mp4"
prop = ffmpeg.probe(in_filename)
width = int(prop["streams"][0]["coded_width"])
height = int(prop["streams"][0]["coded_height"])

process1 = (
    ffmpeg
    .input(in_filename)
    .output('pipe:', format='rawvideo', pix_fmt='rgb24')
    .run_async(pipe_stdout=True)
)

process2 = (
    ffmpeg
    .input('pipe:', format='rawvideo', pix_fmt='rgb24', s='{}x{}'.format(width, height))
    .output(out_filename, pix_fmt='yuv420p')
    .overwrite_output()
    .run_async(pipe_stdin=True)
)

while True:
    in_bytes = process1.stdout.read(width * height * 3)
    if not in_bytes:
        break
    in_frame = (
        np
        .frombuffer(in_bytes, np.uint8)
        .reshape([height, width, 3])
    )
    out_frame = in_frame * 0.3
    process2.stdin.write(
        out_frame
        .astype(np.uint8)
        .tobytes()
    )

process2.stdin.close()
process1.wait()
process2.wait()
```

## (11) ffmpeg.view

```
ffmpeg.view(stream_spec, detail=False, filename=None, pipe=False, **kwargs)
```

## (12) ffmpeg.colorchannelmixer

调整输入帧的颜色通道。

```
ffmpeg.colorchannelmixer(stream, *args, **kwargs)
```

## (13) ffmpeg.concat

拼接 audio 和 video Stream。

```
ffmpeg.concat(*streams, **kwargs)
```

例子：将文件夹下的若干短视频拼接成长视频

```
import ffmpeg
import glob

streams = []

for in_filename in glob.glob("*.mov"):
    stream = ffmpeg.input(in_filename)
    streams.append(stream)

out_filename = "output.mov"
out = ffmpeg.concat(*streams)
out = ffmpeg.output(out, out_filename)
ffmpeg.run(out)
```

## (14) ffmpeg.crop

裁剪输入 video。

```
ffmpeg.crop(stream, x, y, width, height, **kwargs)
```

例子：

```
in_filename = "his.mp4"
out_filename = "output.mp4"

prop = ffmpeg.probe(in_filename)
video_stream = prop["streams"][0]
width, height = video_stream["coded_width"], video_stream["coded_height"]

x = 0
y = height * 0.13

stream = ffmpeg.input(in_filename)
v = ffmpeg.filter(stream.video, "crop"
                 ,x=x
                 ,y=y
                 ,w=width
                 ,h=height*0.55
                 )
a = stream.audio
out = ffmpeg.output(v, a, out_filename)
ffmpeg.run(out)
```

## (15) ffmpeg.drawbox

在输入 video 上画框。

```
ffmpeg.drawbox(stream, x, y, width, height, color, thickness=None, **kwargs)
```

## (16) ffmpeg.drawtext

在 video 顶部写文字。

```
ffmpeg.drawtext(stream, text=None, x=0, y=0, escape_text=True, **kwargs)
```

参数很多，参考 [ffmpeg-filters 文档: drawtext](https://ffmpeg.org/ffmpeg-filters.html#drawtext)。

## (17) ffmpeg.filter

```
ffmpeg.filter(stream_spec, filter_name, *args, **kwargs)¶
```

例子：

```
ffmpeg.input('in.mp4').filter('hflip').output('out.mp4').run()
```

有一个替代的 `ffmpeg.filter_`，在需要的时候可以用于避免与 Python 内置的 filter 冲突。

## (18) ffmpeg.filter_multi_output

在多输出上应用 ffmpeg。可以用 stream 或 [] 来访问输出的 stream。

例子：

```
split = ffmpeg.input('in.mp4').filter_multi_output('split') 
split0 = split.stream(0) 
split1 = split[1] 
ffmpeg.concat(split0, split1).output('out.mp4').run()
```

## (19) ffmpeg.hflip

水平翻转。

```
ffmpeg.hflip(stream)
```

## (20) ffmpeg.hue

调整 hue，saturation。细节参考 [ffmpeg-filters 文档: hue](https://ffmpeg.org/ffmpeg-filters.html#hue)。

```
ffmpeg.hue(stream, **kwargs)
```

## (21) ffmpeg.overlay

将一个视频覆盖到另一个视频上面。细节参考 [ffmpeg-filters 文档: overlay](https://ffmpeg.org/ffmpeg-filters.html#overlay-1)。

```
ffmpeg.overlay(main_parent_node, overlay_parent_node, eof_action='repeat', **kwargs)
```

## (22) ffmpeg.setpts

改变 input frames 的 PTS(presentation timestamp)。

```
ffmpeg.setpts(stream, expr)
```

## (23) ffmpeg.trim

从输入中截取连续的一段。细节参考 [ffmpeg-filters 文档: trim](https://ffmpeg.org/ffmpeg-filters.html#trim)。

```
ffmpeg.trim(stream, **kwargs)
```

例子：去掉视频末尾 3 秒钟的平台签名

```
in_filename = ...
out_filename = "vertical_{}".format(in_filename)
prop = ffmpeg.probe(in_filename)
duration = float(prop["format"]["duration"])

stream = ffmpeg.input(in_filename)
v = stream.video.filter("trim", duration=duration-3.0)
a = stream.audio.filter("atrim", duration=duration-3.0)
out = ffmpeg.output(v, a, out_filename)
ffmpeg.run(out)
```

## (24) ffmpeg.vflip

垂直翻转。

```
ffmpeg.vflip(stream)
```

## (25) ffmpeg.zoompan

zoom 是缩放，pan 是取景。细节参考 [ffmpeg-filters 文档: zoompan](https://ffmpeg.org/ffmpeg-filters.html#zoompan)。

------

# 例子

## (1) 基本操作

### 水平翻转视频

这是 ffmpeg-python 项目的 github 仓库中给出的一个例子。实现水平翻转视频，命令行的写法如下：

```
ffmpeg -i video.mov -vf hflip output.mp4
```

如果用 Python 写，代码如下：

```
import ffmpeg

stream = ffmpeg.input("video.mov")
stream = ffmpeg.hflip(stream)
stream = ffmpeg.output(stream, "output2.mp4")
ffmpeg.run(stream)
```

或者：

```
import ffmpeg

(
    ffmpeg
    .input('video.mov')
    .hflip()
    .output('output3.mp4')
    .run()
)
```

### 将一个图片水平翻转后嵌入到视频里

这也是本项目的 github 仓库中的例子，比较复杂，其信号图如下：

命令行写法非常复杂，如下：

```
ffmpeg -i video.mov -i image.png -filter_complex "[0]trim=start_frame=10:end_frame=20[v0];\
    [0]trim=start_frame=30:end_frame=40[v1];[v0][v1]concat=n=2[v2];[1]hflip[v3];\
    [v2][v3]overlay=eof_action=repeat[v4];[v4]drawbox=50:50:120:120:red:t=5[v5]"\
    -map [v5] output.mp4
```

看起来非常眼晕，此时如果用 ffmpeg-python，则可读性大为改善：

```
import ffmpeg

in_file = ffmpeg.input("video.mov")
overlay_file = ffmpeg.input("image.png")
(
    ffmpeg
    .concat(
        in_file.trim(start_frame=10, end_frame=20),
        in_file.trim(start_frame=30, end_frame=40),
    )
    .overlay(overlay_file.hflip())
    .drawbox(50, 50, 120, 120, color="red", thickness=5)
    .output("out.mp4")
    .run()
)
```

### 将视频转换为 numpy 数组

下面的代码中，返回的 video 是 RGB 图像的数组。

```
out, _ = (
    ffmpeg.input(video_path)
    .output("pipe:", format="rawvideo", pix_fmt="rgb24")
    .run(capture_stdout=True)
)
video = (
    np
    .frombuffer(out, np.uint8)
    .reshape([-1, height, width, 3])
)
```

### 从帧序列组装视频(无声音)

```
(
    ffmpeg
    .input('/path/to/jpegs/*.jpg', pattern_type='glob', framerate=25)
    .output('movie.mp4')
    .run()
)
```

------

## (2) probe

使用 probe 可以读取视频信息。我们可以根据需要选取感兴趣的信息进行输出。

- 总帧数

```
in_filename = "video.mov"
prop = ffmpeg.probe(in_filename)
N = int(prop["streams"][0]["nb_frames"])
```

- 视频长度

```
def get_info_duration(_prop):
    # 长度
    duration = float(_prop["format"]["duration"])
    m, s = divmod(duration, 60)
    h, m = divmod(m, 60)
    return ("{}:{}:{}".format(h, m, s))
```

- 空间大小

```
def get_info_size(_prop):
    # 大小
    size = float(_prop["format"]["size"]) / 1024
    return ("{} KB".format(size))
```

- 宽和高

```
def get_code_size(_prop):
    # 宽 * 高
    for video_stream in _prop["streams"]:
        if video_stream["codec_name"] == "h264":
            return ("{} * {}".format(str(video_stream["coded_width"]), str(video_stream["coded_height"])))
    return ""
```

调用方法：

```
prop = ffmpeg.probe(video_path)
print("{}\t{}\t{}\t{}\n".format(video_path
                               ,get_info_duration(prop)
                               ,get_info_size(prop)
                               ,get_code_size(prop)
                               ))
```

## (3) filter

### 截取一张图，并进行缩放

```
(
    ffmpeg
    .input(in_filename, ss=ss)
    .filter('scale', width, -1)
    .output(out_filename, vframes=1)
    .run()
)
```

### 将 logo 图覆盖到视频画面上

```
main = ffmpeg.input("video.mov")
logo = ffmpeg.input("image.png")
(
    ffmpeg
    .filter([main, logo], 'overlay', 10, 10)
    .output('out.mp4')
    .run()
)
```

### 音频和视频管道

```
in1 = ffmpeg.input('video.mov')
in2 = ffmpeg.input('video.mov')
v1 = in1.video.hflip()
a1 = in1.audio
v2 = in2.video.filter('reverse').filter('hue', s=0)
a2 = in2.audio.filter('areverse').filter('aphaser')
joined = ffmpeg.concat(v1, a1, v2, a2, v=1, a=1).node
v3 = joined[0]
a3 = joined[1].filter('volume', 0.8)
out = ffmpeg.output(v3, a3, 'out.mp4')
out.run()
```

## (4) Pipe

### 通过管道将视频帧读取为 jpeg

取出某一帧

```
out, _ = (
    ffmpeg
    .input(in_filename)
    .filter('select', 'gte(n,{})'.format(frame_num))
    .output('pipe:', vframes=1, format='image2', vcodec='mjpeg')
    .run(capture_stdout=True)
)

img_buffer_np = np.frombuffer(out, dtype=np.uint8)
img_np = cv2.imdecode(img_buffer_np, 1)
```

取出所有帧

```
in_filename = "video.mov"
prop = ffmpeg.probe(in_filename)
N = int(prop["streams"][0]["nb_frames"])
for frame_num in range(1, N + 1):
    out, _ = (
        ffmpeg
        .input(in_filename)
        .filter('select', 'gte(n,{})'.format(frame_num))
        .output('pipe:', vframes=1, format='image2', vcodec='mjpeg')
        .run(capture_stdout=True)
    )

    img_buffer_np = np.frombuffer(out, dtype=np.uint8)
    img_np = cv2.imdecode(img_buffer_np, 1)
    cv2.imwrite("img/{}.png".format(frame_num), img_np)
```

## (5) 声音

### 将声音转换为原始 PCM 音频

```
out, _ = (ffmpeg
    .input(in_filename, **input_kwargs)
    .output('-', format='s16le', acodec='pcm_s16le', ac=1, ar='16k')
    .overwrite_output()
    .run(capture_stdout=True)
)
```

### 带偏移的单声道到立体声

```
audio_left = (
    ffmpeg
    .input('audio-left.wav')
    .filter('atrim', start=5)
    .filter('asetpts', 'PTS-STARTPTS')
)

audio_right = (
    ffmpeg
    .input('audio-right.wav')
    .filter('atrim', start=10)
    .filter('asetpts', 'PTS-STARTPTS')
)

input_video = ffmpeg.input('input-video.mp4')

(
    ffmpeg
    .filter((audio_left, audio_right), 'join', inputs=2, channel_layout='stereo')
    .output(input_video.video, 'output-video.mp4', shortest=None, vcodec='copy')
    .overwrite_output()
    .run()
)
```