---
title: 解决freeSwitch播放多个视频文件，切换时首帧黑屏的问题
---

我们在做视频客服时，需要连接播放多个mp4文件，但在调用playback进行播放时，在两个mp4文件播放切换时，中间会有一帧的黑屏，造成播放效果非常不理想；经过多方尝试及咨询各种专家，终于有了一个完美的解决方案：

​    （1）第一步需要修改FreeSwitch代码，FreeSwitch在一个文件播放前及播放后会插入一帧的黑色背景，所以造成切换时有一个黑屏的现象；我们的做法是暴力将该段代码注释掉即可；代码在switch_core_media.c的video_write_thread函数内：
![image-20210713142132402](/images/解决freeSwitch播放多个视频文件，切换时首帧黑屏的问题/image-20210713142132402.png)

​    (2)防黑屏设置：Ivr hisancc_ctidialplan.xml

注意配置一定放在“<action application="pre_answer"/>”之前，设置core_video_blank_image=false

![img](/images/解决freeSwitch播放多个视频文件，切换时首帧黑屏的问题/GetImage.ashx)

 

:[/home/ucp/ipcc/media/conf/scc/dialplan]cat hisancc_ctidialplan.xml

 

![img](/images/解决freeSwitch播放多个视频文件，切换时首帧黑屏的问题/GetImage.ashx)