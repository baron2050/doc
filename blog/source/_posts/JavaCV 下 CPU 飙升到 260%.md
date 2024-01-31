---
title: JavaCV 下 CPU 飙升到 260%
tags: 
    - java
    - 音视频处理
---
### 描述

`Spring Boot` 引入 `JavaCV` 做视频图像处理，当然首选`FFmpeg`了，心里碎碎念，开源的东西就是好用。于是啪啦啪啦编码一整天，实现了`RTMP流媒体推送`、`每间隔时间截取帧图片`、`流媒体转MP4保存后用于回放`三个主要功能。

打完Jar包后一看，560M大小，天呢！！！ 一会 `exclusions`一下。

碎碎念！！！

运行了一下，两路RTSP流，很轻松！CPU占用不到10%，内存1.1G，可以接受。

于是，果断发往服务器，然后启动……两路RTSP流，CPU果断飘升到260%……完蛋！！！

于是开始调优，视频解码真心消耗资源，无奈中……

### JVM寻找问题&调优

网上各类这种文章，这里不多做介绍，一般就是两步，首先寻找CPU高的线程；然后找到线程的记录，看看有啥问题。

### 发现了问题

`JavaCV` 会对流媒体进行解析，这就很耗资源，无解的难题。

于是，最好的解决方案就是，不解析视频。  `FFmpeg`支持 `codec copy`,而 JavaCV也可以通过 `AVPacket`实现。

```java
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;
import org.bytedeco.javacv.*;

import org.bytedeco.ffmpeg.avcodec.AVPacket;

public class PacketRecorderTest {

    private static final DateFormat DATE_FORMAT = new SimpleDateFormat("yyyyMMdd__hhmmSSS");

    private static final int RECORD_LENGTH = 5000;

    private static final boolean AUDIO_ENABLED = false;

    public static void main(String[] args) throws FrameRecorder.Exception, FrameGrabber.Exception {

        String inputFile = "/home/usr/videos/VIDEO_FILE_NAME.mp4";

        // Decodes-encodes
        String outputFile = "/tmp/" + DATE_FORMAT.format(new Date()) + "_frameRecord.mp4";
        PacketRecorderTest.frameRecord(inputFile, outputFile);

        // copies codec (no need to re-encode)
        outputFile = "/tmp/" + DATE_FORMAT.format(new Date()) + "_packetRecord.mp4";
        PacketRecorderTest.packetRecord(inputFile, outputFile);

    }

    public static void frameRecord(String inputFile, String outputFile) throws FrameGrabber.Exception, FrameRecorder.Exception {

        int audioChannel = AUDIO_ENABLED ? 1 : 0;

        FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(inputFile);
        FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(outputFile, 1280, 720, audioChannel);

        grabber.start();
        recorder.start();

        Frame frame;
        long t1 = System.currentTimeMillis();
        while ((frame = grabber.grabFrame(AUDIO_ENABLED, true, true, false)) != null) {
            recorder.record(frame);
            if ((System.currentTimeMillis() - t1) > RECORD_LENGTH) {
                break;
            }
        }
        recorder.stop();
        grabber.stop();
    }

    public static void packetRecord(String inputFile, String outputFile) throws FrameGrabber.Exception, FrameRecorder.Exception {

        int audioChannel = AUDIO_ENABLED ? 1 : 0;

        FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(inputFile);
        FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(outputFile, 1280, 720, audioChannel);

        grabber.start();
        recorder.start(grabber.getFormatContext());

        AVPacket packet;
        long t1 = System.currentTimeMillis();
        while ((packet = grabber.grabPacket()) != null) {
            recorder.recordPacket(packet);
            if ((System.currentTimeMillis() - t1) > RECORD_LENGTH) {
                break;
            }
        }

        recorder.stop();
        grabber.stop();

    }
}
```

感受一下，还是好用的。但是在进行流媒体的转换时，总会出现各种问题。

### 最后的解决方案

碎碎念！！！

还是直接使用 `Process` 调用 `FFmpeg`吧！

```java
import org.bytedeco.javacpp.Loader;

import java.io.IOException;

public class CommondTest {

    public static void main(String[] args) throws IOException, InterruptedException {
        String ffmpeg = Loader.load(org.bytedeco.ffmpeg.ffmpeg.class);
        String from = "rtsp://*************";
        String to = "rtmp://********************";
        ProcessBuilder pb = new ProcessBuilder(ffmpeg,
            "-i", from, "-codec", "copy", "-f", "flv", "-y", to);
        Process process = pb.inheritIO().start();
        process.waitFor();
    }
}
```

测试了一下，还是好用的，那么可以开始改造了！

### 学会 FFmpeg 命令

学习知识的最好途径永远是官网，因为`权威` [ffmpeg Documentation](https://link.juejin.cn?target=https%3A%2F%2Fffmpeg.org%2Fffmpeg-all.html%23Main-options)

在用java调用的时候，要设置好输出，防止因为命令行的输出流没有被读取而造成堵塞！

所以，还是把ffmpeg的日志级别设置一下吧，`-loglevel quiet`

### CPU下来了

终于下来了！

技术问题解决后，剩下的截图，调用某度云的故障识别接口并实时报警……

碎碎念！！！某度云提供了5个图片故障识别的接口，每张截图得分别调用5个接口……，看来必然要使用异步了……

### 服务器信息 （4核8G/2.4GHz）

#### 服务器CPU基本信息: cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c

```scss
Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz
```

#### 服务器内存基本信息: cat /proc/meminfo

```makefile
MemTotal:        8008500 kB
```

#### 物理CPU个数: cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

#### 核数: cat /proc/cpuinfo| grep "cpu cores"| uniq

```
cpu cores	: 2
```

#### 逻辑CPU的个数： cat /proc/cpuinfo| grep "processor"| wc -l

#### 六路摄像头下消耗资源: CPU消耗 20.3%，内存占用1.3G

```sql
top - 18:14:33 up 17 days, 7 min,  1 user,  load average: 1.03, 0.92, 0.80
Tasks: 105 total,   1 running, 104 sleeping,   0 stopped,   0 zombie
%Cpu(s): 20.3 us,  0.6 sy,  0.0 ni, 78.7 id,  0.0 wa,  0.0 hi,  0.3 si,  0.1 st
KiB Mem :  8008500 total,   120588 free,  1940580 used,  5947332 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  5368844 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
28360 root      20   0 6307744   1.3g  24644 S  83.4 16.4  42:04.91 java
```

#### 查看网卡流量： watch -n 1 "/sbin/ifconfig eth0 | grep bytes"

```python
RX packets 234768965  bytes 315037366631 (293.4 GiB)
TX packets 224227493  bytes 113722298707 (105.9 GiB)
```



