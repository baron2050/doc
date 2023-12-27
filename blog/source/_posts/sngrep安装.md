---
title: centos安装sngrep
---

# centos安装sngrep

sngrep可以可视化显示呼叫过程中SIP流程。
sngrep开源软件的网址为：https://github.com/irontec/sngrep。


安装步骤：

1. 添加 /etc/yum.repos.d/irontec.repo文件，文件中输入以下内容：

```
[irontec]
name=Irontec RPMs repository
baseurl=http://packages.irontec.com/centos/$releasever/$basearch/123
```

1. 执行以下命令添加 Irontec repositories 的公钥

```
rpm --import http://packages.irontec.com/public.key1
```

1. 执行以下命令开始安装

```
yum install sngrep
```