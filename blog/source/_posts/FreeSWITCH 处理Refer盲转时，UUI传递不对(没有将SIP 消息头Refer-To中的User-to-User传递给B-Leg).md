---
title: FreeSWITCH 处理Refer盲转时，UUI传递不对(没有将SIP 消息头Refer-To中的User-to-User传递给B-Leg)
---

**运行环境：**

  CentOS 7.6

  FreeSWICH 1.6.18

 

**一、问题场景：**

  FreeSWITCH收到REFER命令后，重新发起的INVITE消息中的 "User-to-User" 消息头信息不对，跟REFER命令的 "Refer-To" 消息头中的User-to-User参数值不同。

  具体报文情况如下(省略了部分SIP信息)：

```
REFER sip:mod_sofia@10.2.32.90:5080 SIP/2.0
Via: SIP/2.0/UDP 10.2.32.116:5080;rport;branch=z9hG4bKryvUZZerH16DN
From: <sip:449998@10.2.32.116:5080>;tag=XH27mSc4ZyjaF
To: "Extension 296898" <sip:296898@10.2.32.90>;tag=6g430SNK5Be3m
Call-ID: 46eda73c-76b2-1239-e1b5-487b6b8ad630
Contact: <sip:449998@10.2.32.116:5080;transport=udp>
Refer-To: <sip:296896@test.refer.com?User-to-User=00C82b264F7267416E693D3131313337303035303030303139393433333634264F7267446E69733D333030393637%3Bencoding%3Dhex>
Referred-By: <sip:10.2.32.116:5080>
User-to-User: 00C8426613D31%3Bencoding%3Dhex


INVITE sip:296896@10.32.26.19:50078;rinstance=b4e528536c8c5a3d SIP/2.0
Via: SIP/2.0/UDP 10.2.32.90;rport;branch=z9hG4bKtK4yS4SH849vD
Route: <sip:296896@10.32.26.19:50078>;rinstance=b4e528536c8c5a3d
From: "Extension 296898" <sip:296898@10.2.32.90>;tag=ymDeQZt2ZcK1B
To: <sip:296896@10.32.26.19:50078;rinstance=b4e528536c8c5a3d>
Call-ID: 4992bbb6-76b2-1239-e1b5-487b6b8ad630
Contact: <sip:mod_sofia@10.2.32.90:5060>
Referred-By: <sip:10.2.32.116:5080>
User-to-User: 00C8426613D31%3Bencoding%3Dhex
Remote-Party-ID: "Extension 296898" <sip:296898@10.2.32.90>;party=calling;screen=yes;privacy=off
```

**二、问题原因：**

  FreeSWICH默认情况下，处理REFER命令进行盲转时，新发起的INVITE消息中的 "User-to-User" 是从REFER的 "User-to-User" 头中获取的，而非"Refer-to"里的参数

 

  **FS处理Refer命令的逻辑：**

- 信令交互逻辑

​      ![]('/images/FreeSWITCH 处理Refer盲转时，UUI传递不对(没有将SIP 消息头Refer-To中的User-to-User传递给B-Leg)/773260-20200921221041354-1111469874.png') 

- 信令处理逻辑

- - REFER的SIP消息头"Refer-To"中的信息，只提取盲转的被叫号码，忽略其他参数(如User-to-User)，然后发起新的INVITE
  - 新的INVITE消息中的“User-to-User”头，是根据REFER命令中的“User-to-User”头获取，非“Refer-To”里的参数值

**三、解决方案**

  FreeSWITCH 收到 REFER 后，取出sip_refer_to头中的值，执行export nolocal 命令，将值设置到未来bridge的B-leg通道上即可。

  增加下面拨号方案即可：)

```
  <extension name="refer request" continue="true">
    <condition field="${sip_refer_to}" expression=".+User-to-User=(.+)%3B">
       <action application="log" data="INFO yuxiu:change User-to-User by sip_refer_to : sip_refer-to [$1]"/>
       <action application="export" data="nolocal:sip_h_User-to-User=$1;encoding=hex"/>
    </condition>
  </extension>
```

  通过上面方式，如果FS收到的REFER命令的“Refer-To”头中有User-to-User参数，就取该参数作为后续INVITE的“User-to-User”头，否则依旧取REFER命令中的“User-to-User”头

  修改后，盲转报文将变成下面形式：

```
REFER sip:mod_sofia@10.2.32.90:5080 SIP/2.0
From: <sip:449998@10.2.32.116:5080>;tag=XH27mSc4ZyjaF
To: "Extension 296898" <sip:296898@10.2.32.90>;tag=6g430SNK5Be3m
Refer-To: <sip:296896@test.refer.com?User-to-User=00C82b264F7267416E693D3131313337303035303030303139393433333634264F7267446E69733D333030393637%3Bencoding%3Dhex>
User-to-User: 00C8426613D31%3Bencoding%3Dhex

INVITE sip:296896@10.32.26.19:50078;rinstance=b4e528536c8c5a3d SIP/2.0
From: "Extension 296898" <sip:296898@10.2.32.90>;tag=ymDeQZt2ZcK1B
To: <sip:296896@10.32.26.19:50078;rinstance=b4e528536c8c5a3d>
User-to-User:  00C82b264F7267416E693D3131313337303035303030303139393433333634264F7267446E69733D333030393637%3Bencoding%3Dhex
Remote-Party-ID: "Extension 296898" <sip:296898@10.2.32.90>;party=calling;screen=yes;privacy=off
```

四、模拟的呼叫场景：

​    ![](/images/FreeSWITCH 处理Refer盲转时，UUI传递不对(没有将SIP 消息头Refer-To中的User-to-User传递给B-Leg)/773260-20200921221250616-1813717684.png)

 

**4.1、PBX-FS机器：充当分机注册服务器**

   在PBX-FS (10.2.32.90:5060)上注册两个分机：296896 和 296898

   然后让296989 呼叫 449998 进行测试

- 拨号方案详情：default.xml 

```
  <!-- 处理refer消息头 -->
  <extension name="refer request" continue="true">
    <condition field="${sip_refer_to}" expression=".+User-to-User=(.+)%3B">
       <action application="log" data="WARNING yuxiu:change User-to-User by sip_refer_to : sip_refer-to [$1]"/>
       <action application="export" data="nolocal:sip_h_User-to-User=$1;encoding=hex"/>
    </condition>
  </extension>

  <!-- 模拟呼叫IVR -->
  <extension name="outbound-ivr">
    <condition field="destination_number" expression="^(449998)$">
      <action application="bridge" data="sofia/external/$1@10.2.32.116:5080"/>
      <action application="hangup" data="Esl Server ERROR"/>
    </condition>
  </extension>

  <!-- 模拟呼叫PBX分机 -->
  <extension name="outbound-pbx">
    <condition field="destination_number" expression="^(296\d{3})$">
      <action application="bridge" data="user/${1}"/>
      <action application="hangup" data="Esl Server ERROR"/>
    </condition>
  </extension>
```

**4.2、IVR-FS机器：充当IVR导航服务器**

  收到来电后，自动应答，然后播放一段提示音后，refer 盲转到296896

[![复制代码](img/FreeSWITCH 处理Refer盲转时，UUI传递不对(没有将SIP 消息头Refer-To中的User-to-User传递给B-Leg)/copycode.gif)](javascript:void(0);)

```
  <extension name="outbound-test2">
    <condition field="destination_number" expression="^449998$">
      <action application="log" data="INFO  uui=${sip_h_User-to-User}"/>
      <action application="answer"/>
      <action application="playback" data="/usr/local/freeswitch/sounds/welcom_to_call_yuxiu_ivr.wav"/>
      <action application="deflect" data="sip:296896@test.refer.com?User-to-User=00C82b264F7267416E693D3131313337303035303030303139393433333634264F7267446E69733D333030393637%3Bencoding%3Dhex"/>
      <action application="log" data="WARNING refer finished"/>
      <action application="hangup" data="refer finished"/>
    </condition>
  </extension>
```

**4.3、FS测试日志**

```
EXECUTE sofia/internal/296898@10.2.32.90 bridge(sofia/external_90/449998@10.2.32.116:5080)

[DEBUG]  sofia.c:8544 Process REFER to [296896@test.refer.com]
[INFO] mod_dialplan_xml.c:637 Processing 296898 <296898>->296896 in context default
EXECUTE sofia/internal/296898@10.2.32.90 bridge(user/296896)
```