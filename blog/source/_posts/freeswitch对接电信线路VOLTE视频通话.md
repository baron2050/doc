---
title: freeswitch对接电信线路VOLTE视频通话
---

对接VOLTE视频通话需在profile设置上视频编码。或在public.xml上设置出局视频编码。

<action application="export" data="nolocal:absolute_codec_string=PCMA,H264"/>

此时发出的invite消息里带有的video

![img](/images/freeswitch对接电信线路VOLTE视频通话/GetImage.ashx)

此处尽量使用H264，其他的编码运营商可能不支持。再着freeswitch对视频编码只能透传不能转码。所以尽可能的使用H264编码

除了以上的操作外，还需携带LEVELID参数，若不携带这个参数会导致运营商按照最低等级处理H264编码。目前测试发现不带这个的话，主叫可以看到被叫，但是被叫看不到主叫，所以特此需要带上此类参数。

我查了一圈没发现freeswitch怎么添加这个参数，只能以dialplan的方式修改SDP消息，如下：

```xml
<action application="set"><![CDATA[switch_r_sdp=v=0
o=- 123456 123 IN IP4 192.168.3.176
s=etmedia
c=IN IP4 192.168.3.176
t=0 0
a=X-nat:4002 Unknown
m=audio 4002 RTP/AVP 8 101
a=rtpmap:8 PCMA/8000
m=video 6666 RTP/AVP 99
b=AS:1024
a=rtpmap:99 H264/90000
a=fmtp:99 profile-level-id=42e00a; max-bar=1024
a=rtcp-fb:99 ccm fir
a=rtcp-fb:99 ccm tmmbr
a=rtcp-fb:99 nack
a=rtcp-fb:99 nack pli
]]>
 </action>
```

至此测试就正常了。