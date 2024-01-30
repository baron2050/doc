---
title: 基于DINet+Openface实现自训练数字人
date: 2024-01-30
---

### 前言

本文档记载基于DINet+openface的数字人模型训练和推理流程。 先给大家展示一下我们自己训练出来的效果吧： [www.bilibili.com/video/BV1zG…](https://link.juejin.cn?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1zG411d7rh%2F)

### 参考文档和源码地址

> \1. DINet代码地址：[github.com/MRzzm/DINet](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FMRzzm%2FDINet)
>
> \2. DINet论文地址：[arxiv.org/pdf/2303.03…](https://link.juejin.cn?target=https%3A%2F%2Farxiv.org%2Fpdf%2F2303.03988.pdf)

### 训练硬件环境说明

> \1. 腾讯云GPU服务器，1*NVIDIA A100 40G
>
> \2. 操作系统镜像信息：Windows Server 2016 64bit 内存96GB 16核 CPU 

### 训练软件环境说明

> 1. 安装Anaconda （一款开源的包、环境管理器，可自动管理同一设备不同版本环境，这里使用的版本是23.5.2）
> 2. 使用conada创建python虚拟环境并激活（DINet的核心语言使用Python，我这里使用的python版本为3.6）
> 3. 安装CUDA（NVIDIA推出的利用GPU运算平台，该训练过程是依赖GPU算力，该软件很重要，注意：CUDA的版本需和显卡型号对应上，这边使用的版本是11.1，下载地址：[developer.nvidia.com/cuda-toolki…](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.nvidia.com%2Fcuda-toolkit-archive)），安装完成后，系统环境变量需要进行配置
> 4. 安装Pytorch并配置，地址：[pytorch.org/](https://link.juejin.cn?target=https%3A%2F%2Fpytorch.org%2F)，注意：根据自己的CUDA版本选择，首页推荐都是最新的，点击previous version选择之前版本，安装后在command可出现安装命令，启动命令行工具使用conda命令进行安装即可，注意：选择和CUDA相匹配的版本安装
> 5. 也可通过whl的形式安装Pytorch，地址：[download.pytorch.org/whl/torch_s…](https://link.juejin.cn?target=https%3A%2F%2Fdownload.pytorch.org%2Fwhl%2Ftorch_stable.html)，还是要通过CUDA版本选择whl版本，torch、torchvision、python版本不匹配程序是跑不起来的。下载完成后可以使用pip install的形式安装本地whl。（也可以在线通过pip install安装）
> 6. 安装完Pytorch后进行验证，看是否可以通过GPU使用Pytorch，执行一下一下python代码，打印True表示配置完成：
>
> ```python
> python复制代码import torch
> print(torch.cuda.is_available()) #返回True表示可用GPU算力
> ```

### 三方依赖库安装说明

```ini
ini复制代码pip install opencv_python == 4.6.0.66
pip install numpy == 1.16.6
pip install python_speech_features == 0.6
pip install resampy == 0.2.2
pip install scipy == 1.5.4
pip install tensorflow == 1.15.2
pip install torch==1.7.1+cu110 torchvision==0.8.1+cu110 torchaudio==0.10.0+cu111 -f https://download.pytorch.org/whl/torch_stable.html
```
也可以直接使用三方docker
```ini
docker run -itd mcxai/dinet:v1.0
```
### 下载并配置openface，这里给大家提供好下载地址

> \1. 项目链接: [github.com/TadasBaltru…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FTadasBaltrusaitis%2FOpenFace) 
>
> 下载链接：[github.com/TadasBaltru…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FTadasBaltrusaitis%2FOpenFace%2Freleases%2Ftag%2FOpenFace_2.2.0)
>
> 我的系统是win10 所以选择下载这个包，直接解压就可以用：[OpenFace_2.2.0_win_x64.zip](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FTadasBaltrusaitis%2FOpenFace%2Freleases%2Fdownload%2FOpenFace_2.2.0%2FOpenFace_2.2.0_win_x64.zip)
>
> \2. 在openface的文件夹路径下下载模型，打开openface安装目录，找到download_model.ps1文件，选中右键选择“使用PowerShell运行”，等待运行完毕，查看同级目录中的model是否有内容（亲测并非非得某视频中443MB可以正常执行，该过程可能国内网络较慢，可以尝试科学上网）

### 训练大致流程和原理说明

训练的原理大致是将嘴型的范围进行了不同大小的切割，分四阶段训练：

> \1. 第一阶段64x64范围的训练；
>
> \2. 第二阶段128x128范围的训练，该阶段需要加载第一阶段预训练好的64x64的收敛模型；
>
> \3. 第三阶段256x256范围的训练，该阶段需要加载第二阶段预训练好的128x128的收敛模型；
>
> \4. 第四阶段是剪辑训练阶段，使用感知损失、帧/剪辑GAN损失和同步损失。需使用个人预训练的面部和口部框架模型外（即第三阶段的256x256模型），还需要预训练的同步网络模型syncnet。

syncnet模型的下载地址：[pan.baidu.com/s/1Roqdts5i…](https://link.juejin.cn?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F1Roqdts5iPDrjThAbm2NiBQ%3Fpwd%3Dfnc8) 提取码: fnc8 ,在DINNet项目根目录下创建assets，将模型放入即可

### 训练数据集和文件夹路径准备

> 我这里使用iPhone11ProMax录制的4k视频，视频帧率25fps
>
> 录制视频时长15min，使用剪映剪切成大概20s每段视频并导出，不用特别精准非得20s
>
> 注意视频录制过程人物口型清晰，不要有太大的头部动作，保持面部始终在镜头中
>
> 录制环境安静，我们使用了绿幕和打光
>
> 注意视频片段命名不能带特殊字符和中文，我这里使用01、02、03.....这种方式

**在assets文件夹下创建如下3个文件夹：**

![img](/images/基于DINet实现自训练数字人/1.png)

- inference_result：存放最后生成的推理视频（自动存放）
- traininng_model_weight：存放预训练的模型权重
- traininng_data：存放要进行训练时使用到的处理数据

**在traininng_data下创建如下6个文件夹：** ![img](/images/基于DINet实现自训练数字人/2.png)

- split_video_25fps：存放处理好的25fps的视频片段
- split_video_25fps_audio：存放提取到的视频片段中的所有音频片段
- split_video_25fps_crop_face：存放所有视频中裁剪面部的图像文件
- split_video_25fps_deepspeech：存放所有音频中的深度语音特征文件txt格式
- split_video_25fps_frame：存放所有视频片段中提取到的帧画面
- split_video_25fps_lanndmark_openface：存放视频帧中提取到的面部特征点数据csv文件

### openface处理视频数据集

打开OpenFaceOfflinne.exe工具，如下图在工具栏的Record选项下取消Record pose和Record gaze选项

![img](/images/基于DINet实现自训练数字人/3.png)

然后点击file，选择你要处理的视频（可多选），注意如果视频处理过程中出现卡死或者OpenFace软件闪退，那该视频需删除，不再使用，同时删除对应的处理数据（下面会说处理数据的位置）

![img](/images/基于DINet实现自训练数字人/4.png)

正常处理视频片段的效果如下，(这个是在Mac平台安装Openface后的处理效果图，不建议在Windows进行，mac下安装Openface地址：[github.com/TadasBaltru…](https://github.com/TadasBaltrusaitis/OpenFace) ，后面有时间了我出一期Openface在mac下的安装教程，还有些坑，装完后的使用也摸索了一下，总之真香~ ![img](/images/基于DINet实现自训练数字人/4.png)

openface在处理某个视频片段过程会生成三个文件，路径如下，在openface安装路径下的processed文件中，这三个文件中txt格式文件是视频解析过程中详细数据文件，没什么用，可以删掉，avi文件打开后是脸部打点的视频转换文件，没有声音，后面也不会用到；最重要的是csv文件，里面存放着脸部的数据解析文件，打开后可以看到有一些坐标信息，后面要用到。注意：如果某个视频处理中断了，或者openface闪退了，这里需将对应的三个文件也删除掉：

![img](/images/基于DINet实现自训练数字人/6.png)

视频片段全部处理完毕，需将processed文件夹中所有csv文件拷贝到上述说到的assets文件夹下的traininng_data文件夹下的split_video_25fps_lanndmark_openface文件夹中：

![img](/images/基于DINet实现自训练数字人/7.png)

注意需要将开始用来训练的mp4片段放到assets文件夹下的traininng_data文件夹下的split_video_25fps文件夹中，不是openface处理后的avi文件，是原始mp4文件：

![img](/images/基于DINet实现自训练数字人/8.png)

视频处理时长和视频的清晰度以及视频片段的长短都有关系，我这里15min的4k视频大概处理了1h左右。

### 提取视频片段中的图片帧

注意需要先切换到conda创建的python虚拟环境中（非conda部署可以不切），因为执行脚本在DINet路径下，所以执行命令时在该路径下执行，data_processing.py文件是核心的代码文件，可使用python的IDE打开，仔细查看对应代码

```css
css
复制代码python .\data_processing.py --extract_video_frame
```

执行完成后，打开./asserts/training_data/split_video_25fps_frame文件夹，可以看到处理的图片帧数据

![img](/images/基于DINet实现自训练数字人/9.png)

注意：这一步在执行过程中可能会报错，因为在生成帧过程中，会产生空帧（概率性），所以代码中需要做下过滤，要不就会卡住，打开data_processing.py文件，我这里用的是pycharm，找到extract_video_frame函数，在函数最后找到写入图片帧的部分，添加判断：如果当前帧为空帧，不再写入，具体代码如下：

![img](/images/基于DINet实现自训练数字人/10.png)

```css
css复制代码if frame is not None:
    cv2.imwrite(result_path, frame)
```

### 提取视频片段中的音频文件

同样实在DINet目录下执行该命令

```css
css
复制代码python .\data_processing.py --extract_audio
```

执行完成后，打开./asserts/training_data/split_video_25fps_audio文件夹，可以看到提取后的音频文件

![img](/images/基于DINet实现自训练数字人/11.png)

### 提取音频文件中的深度语音特征并保存

```css
css
复制代码python .\data_processing.py --extract_deep_speech
```

执行完成后，打开./asserts/training_data/split_video_25fps_deepspeech文件夹，可以看到提取的深度语音特征文件

![img](/images/基于DINet实现自训练数字人/12.png)

### 提取视频中面部特征图像

```css
css
复制代码 python .\data_processing.py --crop_face
```

执行完成后，打开./asserts/training_data/split_video_25fps_crop_face文件夹，可以看到提取的面部特征图片

![img](/images/基于DINet实现自训练数字人/13.png)

### 生成预训练json文件

```css
css
复制代码python .\data_processing.py --generate_training_json
```

执行完成后，打开./asserts/training_data文件即可看到生成的预训练json文件

![img](/images/基于DINet实现自训练数字人/14.png)

### 开始模型训练（Frame training）

模型训练上面说分4个阶段，如果按类型划分，可分为两类，一类是模型训练（Frame training），一类是剪辑训练（Clip training）.

接下来进行104x80的嘴部区域为64x64分辨率的训练，具体执行命令如下：

```css
css
复制代码 python train_DINet_frame.py --augment_num=32 --mouth_region_size=64 --batch_size=40 --result_path=./asserts/training_model_weight/frame_training_64
```

注意，这里我设置的batch_size=40，github上为24，这个表示训练过程批处理的大小，值越大对显卡的要求越高，我的是A100，40G显存，大家可结合自己的显存情况自行测试设置。（注意：可以先设置大点，如果报错，逐步调小，充分利用硬件算力）

该步骤我这边执行了大概8h左右，因为我是直接执行到了400pth自动结束的，网上有一些资料显示，训练过程lr_g学习率参数逐步衰减为原来的50%就可以停下来了，因为前期学习率大，会加速学习，使得模型更容易接近局部或全局最优解。但是在后期会有较大波动，甚至出现损失函数的值围绕最小值徘徊，波动很大，始终难以达到最优。所以这里代码中使用了学习率衰减的概念，直白点说，就是在模型训练初期，会使用较大的学习率进行模型优化，随着迭代次数增加，学习率会逐渐进行减小，保证模型在训练后期不会有太大的波动，从而更加接近最优解，当lr_g衰减到原来的50%，模型基本可以使用了（大家可自行测试）。训练过程中会显示如下画面，剩下就是漫长的等待了.......

![img](/images/基于DINet实现自训练数字人/15.png)

这一步训练如果等他自动结束，可以打开./asserts/training_model_weight/frame_training_64文件夹，查看训练的模型，保留最后一个netG_model_epoch_400.pth就可以了，其他可以删掉了

![img](/images/基于DINet实现自训练数字人/16.png)

第一步训练结束，接下来进行128x128的模型训练，这一步需要加载第一步中预训练的（面部：104x80,嘴巴64x64）模型，并开始（面部208x160,嘴部：128x128）的训练，具体命令如下：

```css
css
复制代码 python train_DINet_frame.py --augment_num=100 --mouth_region_size=128 --batch_size=55 --coarse2fine --coarse_model_path=./asserts/training_model_weight/frame_training_64/netG_model_epoch_400.pth --result_path=./asserts/training_model_weight/frame_training_128
```

状态同第一步，我这里也是等待他自动停之后再进行下一步，当然也可以根据收敛判断，lr_g降低为原来的50%即可，这一步训练如果等他自动结束，可以打开./asserts/training_model_weight/frame_training_128文件夹，查看训练的模型，保留最后一个netG_model_epoch_400.pth就可以了，其他可以删掉。

这个过程时间会更长，我这一步等待他自动结束大概走了33h，看日志，lr_g降低为原来50%时，大概25h，（这次4k视频，时长为15min）

第二步训练结束，接下来进行256x256的模型训练，这一步需要加载第二步中预训练的（面部：208x160,嘴巴128x128）模型，并开始（面部416x320,嘴部：256x256）的训练，具体命令如下：

```css
css
复制代码 python train_DINet_frame.py --augment_num=20 --mouth_region_size=128 --batch_size=12 --coarse2fine --coarse_model_path=./asserts/training_model_weight/frame_training_128/netG_model_epoch_400.pth --result_path=./asserts/training_model_weight/frame_training_256
```

状态同上，接下来依旧是漫长的等待，训练结束后，可以打开./asserts/training_model_weight/frame_training_256文件夹，查看训练的模型，同样是保留最后1个，其他的可以删掉

### 开始剪辑训练（Clip training）

该训练步骤使用感知损失、帧/剪辑GAN损失和同步损失。需要加载预训练框架模型(面部：416x320,嘴部：256x256)，以及预训练同步网络模型(嘴部：256x256，该模型为已训练好的模型，暂无该模型训练教程)，具体剪辑训练的指令如下：

将模型文件下载好，放入指定目录： ./asserts/syncnet_256mouth.pth

```css
css
复制代码 python train_DINet_clip.py --augment_num=3 --mouth_region_size=256 --batch_size=2 --pretrained_syncnet_path=./asserts/syncnet_256mouth.pth --pretrained_frame_DINet_path=./asserts/training_model_weight/frame_training_256/netG_model_epoch_400.pth --result_path=./asserts/training_model_weight/clip_training_256
```

开始执行后，依旧是漫长的等待过程，训练结束后，./asserts/training_model_weight/clip_training_256文件夹，查看训练的模型。

### 合成数字人视频

至此，训练阶段告一段落，开始检验训练的效果了，这一步也叫做推理过程，需要准备以下文件：

- 找一段最初始录制的25fps的mp4文件
- 使用openface处理一下，获取该段视频的csv文件
- 任意录制一段wav的音频

在DINet下创建inputs文件夹（其实路径无所谓），在inputs下分别创建video、csv、audio三个文件夹，分别将上面的mp4文件、csv文件、wav文件放入对应的文件夹中，开始执行以下指令，将mp4、csv、wav换成你自己的名字

```css
css
复制代码 python inference.py --mouth_region_size=256 --source_video_path=./DINet/inputs/video/001.mp4 --source_openface_landmark_path=./DINet/inputs/csv/001.csv --driving_audio_path=./DINet/inputs/audio/my.WAV --pretrained_clip_DINet_path=./asserts/training_model_weight/clip_training_256/netG_model_epoch_400.pth
```

这个步骤不会执行太久，根据你提供的wav音频的长短决定，一般十几秒的几分钟左右，执行完成后，会生成3个文件

注意：我这里推理出来的面部带关键点，是因为开始的训练视频使用avi导致，应该使用最初的mp4视频片段

![img](/images/基于DINet实现自训练数字人/17.png)

-001_synthetic_face.mp4：推理出来脸部合成视频

-001_facial_dubbing.mp4：合成的脸部视频和推理视频的融合视频

-001_facial_dubbing_add_audio.mp4：带声音的推理视频

### 踩坑点：

1. openface处理视频时，如果出现卡死或者闪退，证明该视频资源不可用，需移除已生成的avi和csv文件
2. 放到split_video_25fps文件夹中的视频一定是mp4格式的视频片段，不要放openface处理后的avi视频，否则最后推理的视频面部都带标记的关键点
3. 提取视频关键帧时，会出现空帧，写入报错，需修改写入模块代码，判断frame非空时再写入
4. 调整训练的batch_size后，训练过程中有可能会中断，需要关注一下训练状态，否则要等很久
