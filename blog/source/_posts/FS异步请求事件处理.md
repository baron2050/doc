---
title: FS异步请求事件处理
---

ExecuteApplication：sendmsg。收到EventName.ChannelExecuteComplete则执行成功返回ok

正常情况下执行sendmsg只要收到ChannelExecuteComplete就ok，如果是通道操作相关需要等待相关事件。

Bridge：ExecuteApplication：sendmsg。收到EventName.ChannelExecuteComplete且EventName.ChannelBridge或者EventName.ChannelHangup则执行成功或者结束。

BackgroundJob： bgapi。收到BackgroundJob则执行成功返回ok。

正常情况下执行bgapi只要收到BackgroundJob就ok，如果是通道操作相关需要等待相关事件。

InternalOriginate：收到BackgroundJob("originate", originateString)且EventName.ChannelAnswer, EventName.ChannelHangup, EventName.ChannelProgress任意一个事件就返回ok。



