---
title: Java使用JavaCV中的FFmpeg合成视频
tags: 
    - freeswitch
    - 5g
---
# 一、FFmpeg简介

FFmpeg是领先的多媒体框架，能够**解码，编码， 转码，mux，demux，流，过滤和播放**人类和机器创建的几乎所有内容。

> 如果你安装了FFmpeg，可以使用命令行来实现推流或视频处理，此方面内容可以搜索【FFmpeg使用】。总而言之，FFmpeg是一个开源的多媒体编解码工具。

环境搭建 注：因此处只需使用openCV中的API，故无需安装FFmpeg。

maven引入：

```xml
xml复制代码<dependency>
    <groupId>org.bytedeco</groupId>
    <artifactId>javacv-platform</artifactId>
    <version>1.5.3</version>
</dependency>
```

# 二、具体使用

代码下载链接：[CSDN下载](https://link.juejin.cn?target=https%3A%2F%2Fdownload.csdn.net%2Fdownload%2Fweixin_38500202%2F13188001)

- ### 根据本地图片生成视（实现及调用）

> 参数解析：
>  outPath：输出路径
>  width：视频长像素个数
>  height：视频宽像素个数
>  imgNum：imgNum图片个数（此参数决定视频时间）
>  file：图片文件

```java
java复制代码    public static void generateJpeg2Mp4(String outPath, int width, int height, int frameRate, int imgNum, File file) throws FrameRecorder.Exception {
        FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(outPath, width, height);
        // 设置编码
        recorder.setVideoCodec(avcodec.AV_CODEC_ID_H264);
        recorder.setFrameRate(frameRate);
        // 设置视频数据格式
        recorder.setPixelFormat(avutil.AV_PIX_FMT_YUV420P);
        recorder.setFormat("mp4");
        try {
            recorder.start();
            Java2DFrameConverter converter = new Java2DFrameConverter();
            // 录制视频
            int duration = imgNum / frameRate;
            for (int i=0; i<duration; i++) {
                BufferedImage bufferedImage = ImageIO.read(file);
                for (int j=0; j<frameRate; j++) {
                    recorder.record(converter.getFrame(bufferedImage));
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            // 释放资源
            recorder.stop();
            recorder.release();
        }
    }

    public static void main(String[] args) throws FrameRecorder.Exception {
        File img = new File("test.jpeg");
        VideoUtil.generateJpeg2Mp4("test.mp4", 2448, 2048, 19, 5*19, img);
    }
```

- ### 根据图片的byte数组列表生成视频（实现）

> 参数解析：
>  duration：时长（实际取流中，因任务结束不及时，导致多取了一张照片的流，所以多余的需要丢弃）
>  frameRate：帧（1s中多少张照片）
>  images：images是jepg图片的列表。因为从摄像头取流一般都是byte数组，可以取流结束后，调用此接口（防止调帧）。

```java
java复制代码    public static void generateJpegStream2Mp4(String outPath, int width, int height, int duration, int frameRate, List<byte[]> images) throws FrameRecorder.Exception {
        FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(outPath, width, height);
        // 设置编码
        recorder.setVideoCodec(avcodec.AV_CODEC_ID_H264);
        recorder.setFrameRate(frameRate);
        // 设置视频数据格式
        recorder.setPixelFormat(avutil.AV_PIX_FMT_YUV422P);
        recorder.setFormat("mp4");
        try {
            recorder.start();
            Java2DFrameConverter converter = new Java2DFrameConverter();
            // 录制视频
            for (int i=0; i<duration; i++) {
                for (int j=0; j<frameRate; j++) {
                    BufferedImage bufferedImage = ImageIO.read(new ByteArrayInputStream(images.get(i*frameRate + j)));
                    recorder.record(converter.getFrame(bufferedImage));
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            // 释放资源
            recorder.stop();
            recorder.release();
        }
    }
```

- ### 结果

经测试，可以正常生成视频。（晚上拍的，所以内容是黑乎乎的，但是播放无问题）

# 三、总结

经过两天的Code，学习到了视频的处理。我发现绝大部的多媒体处理都用的FFmpeg，🐮

