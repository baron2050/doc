---
title: 余洪涌的建议 Freeswitch群呼
---

# 余洪涌的建议 Freeswitch群呼

/Sofia_Profiles/internal.xml

设置参数： 
```
<param name="manage-presence" value="**false**"/>
```
这个设置为 false
以提高性能。

此外，禁用数据库模式： FreeswitchConsole.exe -nonat -**nosql**

这个是比较省资源的。

FS运行时，cpu 最好不要超过40% ，不能使用cpu太透支。

现在买一个主流的E5 26XX-V3的跑1000线没有问题，主流的服务器估计也就2万，2个 E5 CPU , 跑1000线，平均每线10元

厦门-余洪涌 2014-9-16 14:29:08

CPU E5-2640 V2 * 2

内存16G 就够